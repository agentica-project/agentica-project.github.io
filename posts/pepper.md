---
title: "Pepper: A Real‑Time, Event‑Driven Architecture for Proactive Agentic Systems"
author: "Tianhao Wu, Sijun Tan, and the Agentica Team"
author_line: "Tianhao Wu, Sijun Tan, and the Agentica Team"
date: "2025-10-02"
citation_key: "agentica2025pepper"
---

<aside>

## TL;DR

We introduce [**Pepper**](https://github.com/agentica-org/pepper), a real-time, event-driven architecture enabling the next generation of proactive agentic systems. With Pepper, we built a personal assistant that proactively fetches and summarizes emails, provides context before you even start a conversation, and continues to follow up while working on tasks. We open-source [Pepper](https://github.com/agentica-org/pepper) to advance the creation of proactive agentic systems.

</aside>

![Traditional chatbots (right) wait for user prompts, while Pepper (left) runs in a continuous, event-driven loop—ingesting inputs, acting autonomously, and proactively engaging with the user.](posts/pepper/pepper-arch.png)

## From Reactive Chatbots to Proactive Assistants

Today’s chatbots are largely static and reactive. You ask a question, they generate an answer. If you don’t follow up, the interaction ends.

The next generation of chatbots, as exemplified by ChatGPT Pulse, is dynamic and proactive. These systems interact with you in real time, work continuously in the background, and keep the conversation flowing—even as tasks branch off and run in parallel.

Unlocking this paradigm requires a fundamental shift in infrastructure: *from static, request-driven pipelines to real-time, event-driven architectures.* In this world, LLM calls happen asynchronously in the background, triggering one another in a constantly evolving loop.

Pepper is a architecture built for exactly this, it adopts the design principles behind [Event-Driven Architecture (EDA)](https://miladezzat.medium.com/event-driven-architecture-how-to-implement-in-distributed-systems-29bd82b02ace)—enabling the creation of real-time, event-driven agentic systems. With Pepper, we create agents that proactively fetches and summarizes your emails, provides context before you even start a conversation, and continues to follow up while working on the tasks you assign in the background. We open-source Pepper, as well as the personal assistant we built on top of it, to advance the creation of proactive agentic systems.

## Overview of our Architecture

To power a real-time, proactive assistant, Pepper is built on a modular, event-driven architecture. At its heart is a simple yet powerful pattern: a continuous **Sense-Think-Act** loop, facilitated by three core components working in concert, which make it extremely general.

- **Feeds (Sense):** The system’s sensory input. These are intelligent pipelines that monitor external sources like emails, messages, and calendars. They filter out noise and transform raw data into high-quality, actionable textual signals.
- **Scheduler (Think):** The central brain and orchestrator. It consumes a prioritized queue of signals from the Feeds, maintains the overall state of the interaction, and decides which actions to take. Its non-blocking design ensures the system remains responsive at all times.
- **Workers (Act):** The execution layer of the system. These are specialized agents that receive commands from the *Scheduler* and execute tasks using tools, such as sending an email or searching for information.

These components are connected together via **Context Store**, a real-time data serving layer that allows these decoupled components to communicate seamlessly with each other. 

### System Flow

Here’s how the loop operates in practice:

1. **Feeds** continuously monitor external sources. When a new event (e.g., an incoming email, calendar update, or user message) arrives, the corresponding Feed processes it into a clean signal and writes it to the **Context Store**, ensuring state is always up to date.
2. The **Scheduler** maintains an FIFO event queue that subscribes to these signals. The event queue operates in the producer-consumer pattern. When an event is triggered, it is added to the queue waiting for consumption.
3. The **Scheduler** continuously monitors this queue. At each step, it consumes a batch of events, enriches them with relevant information from the **Context Store** (e.g., conversation history, user memory, current tasks), and decides what actions to take—such as notifying the user, invoking tools, or spawning *Workers*. Each scheduler step here is a single LLM call.
4. While the Scheduler is working, new events continue to arrive. Once it completes a cycle, it immediately fetches the next batch from the queue—creating a **continuous, non-blocking event loop**.

## Major components

### 1. Scheduler: Turning Stateless LLM calls into Real-time Event Processors

Scheduler is the central component of Pepper. It listens to different sources — your messages, important emails, reminders—and reacts as soon as something happens. And it owns a queue to collect and prioritize events.

At its core, the Scheduler operates in a while loop where each step is a single LLM call. In each cycle, it pulls a small batch of events from the FIFO event queue, fetches relevant real-time information from the context store (e.g. conversation history, user profile), and then dispatches any necessary actions, such as calling a tool or handing off a task to a worker agent. 

Since multiple events may have dependencies that needs to be processed together, we design the Scheduler to be synchronous so that there is always only one Scheduler step (LLM call) taking place. And since it is synchronous, each scheduler step has to be lightweight so that it can yield control immediately without blocking the main event processing loop.

**The Bottleneck: Synchronous Tool Calls, and How We Address**

A major bottleneck we encountered is the **synchronous tool-calling model** enforced by APIs such as OpenAI and Anthropic. When the assistant invokes a tool, the conversation must halt until the tool returns a result—no new input can be processed in the meantime.

For example, the following sequence is invalid in the standard paradigm because a user message arrives before the tool result is returned:

```json
{
  "role": "assistant",
  "content": "I'm starting the analysis now.",
  "tool_calls": [{"id": "tool_call_1", "function": {"name": "run_analysis"}}]
},
/*
{
	"role": "tool",
	"tool_call_id": "tool_call_1",
	"content": {"result": "here"}
},
*/
{
  "role": "user",
  "content": "Actually, can you add a filter for last quarter?"
}
// ILLEGAL: A new user message cannot be added. 
// Unless uncomment above tool result.
```

This blocking behavior is especially problematic for long-running tools—such as complex data analysis functions or external agentic systems—which can stall the event loop and introduce significant latency.

![Sync tool call requires the tool's result to be fully returned and processed by the LLM before it can accept a new user message, which causes a large, perceptible delay for the user. In contrast, our async tool call replies to the user immediately, resulting in no delay.](posts/pepper/tool-call.png)

**Our Solution: Asynchronous Tool Call**

To overcome this, Pepper decouples tool execution from the LLM step. Instead of relying on API-enforced synchronous calls, the Scheduler simply **appends the tool invocation to the conversation history** and continues processing subsequent events. The conversation history acts as a context block, packaged into the next prompt alongside other relevant context.

This enables the following message structure where tools are processed asynchronously:

```xml
<tool_call>
  id: tool_call_1
  function: run_analysis
</tool_call>

<user_msg> Actually, can you add a filter for last quarter. </user_msg>

<assistant_msg>Sure. I'll add that filter to the analysis.</assistant_msg>

<tool_result>
  id: tool_call_1
  result: { "initial_analysis_complete": true, ... }
</tool_result>
```

More concretely, the scheduler core logic can be implemented by the following code snippet: 

```python
async def scheduler_step(self):
    """A condensed version of the scheduler's core, non-blocking loop."""
    
    events = await self.get_batch_events()
    self.state_tracker.add_events(events)

    messages = [
        {"role": "system", "content": SCHEDULER_SYSTEM_PROMPT},
        {"role": "user", "content": self.state_tracker.get_user_prompt()},
    ]
    
    response = await create_completion(messages, self.tools)

    # Record the full LLM response (thoughts and tool calls) to the state log.
    self.state_tracker.add_event(response)

    # Schedule tool calls for the engine to execute in the background.
    if response.tool_calls:
        for tool_call in response.tool_calls:
            self.tool_call_engine.schedule(tool_call)
```

### 2. Feeds: Curating High Quality Real-Time Input for Agent

If the Scheduler is the brain of our system, Feeds are its sensors. A real-time system is only as good as the information it receives, and the Scheduler relies on Feeds to provide a constant stream of relevant, high-quality signals to act upon.

The primary role of a Feed is to serve as an intelligent filter. It’s a dedicated pipeline that subscribes to raw, noisy event streams and processes them into concise, informative textual signals. Without this layer, the Scheduler would be flooded with low-priority data, diluting its context and slowing its reaction time. Feeds ensure the Scheduler maintains a high signal-to-noise ratio, allowing it to focus only on what truly matters.

The sophistication of a Feed's processing is tailored to its source, balancing latency with accuracy. This can range from lightweight normalization to complex, multi-step reasoning.

- **Lightweight Feed (Example: Important Emails):** A simple Feed might listen to an email inbox. When a new email arrives, it applies heuristics to check for keywords, sender importance, and urgency. It then distills a complex email into a single, actionable signal for the Scheduler:
    
    ```json
    {
      "id": "evt_a1b2c3d4-e5f6-a7b8-c9d0-e1f2a3b4c5d6",
      "content": "Action requested by Alice on 'Project Phoenix', due tomorrow.",
      "created_at": "2025-09-29T16:46:15Z",
      "metadata": ...
    }
    ```
    
- **Sophisticated Feed (Example: Agentic Processing):** A more advanced Feed could be an entire agentic system itself. For instance, a Feed monitoring team communications might analyze conversation transcripts to detect blockers or shifts in project sentiment, emitting a ticket/task only when a critical threshold is met.

**A Scalable Architecture for Signal Ownership**

Architecturally, Feeds create a crucial "seam" in the system. They empower different engineering teams to take full ownership of signal quality for their domains without needing to understand the complexities of the central Scheduler's orchestration logic. Each team can define:

- **What to listen to:** Subscribing to specific data sources like new emails, calendar changes, booking confirmations, or analytics events.
- **When to emit:** Establishing clear policies for what constitutes an actionable event versus something to be logged, complete with priority levels to ensure urgent items are handled first.
- **How to emit:** Guaranteeing signals are idempotent, deduplicated, and follow a stable schema, which allows the Scheduler and its models to operate on them reliably and safely.

This modularity is made possible by our streamlined context store, which allows developers to easily subscribe to event sources, apply custom processing, and emit signals to a target namespace. Implementing a Feed is as simple as defining a processor function:

```python
# A feed that listens to source namespaces, processes the data, 
# and emits a new signal to the "target" namespace.
@subscriber.on_context_update(namespaces=["source1", "source2"])
async def process_data_into_signal(update):
    raw_data = update.context.data
    # Processing can be a simple function or a complex agentic system
    processed_signal = agentic_system(raw_data)
    
    await context_store.store(
        processed_signal,
        namespace="target"
    )
```

The true power of this architecture extends beyond reacting to events. The next evolution is predictive Feeds that anticipate user needs. By analyzing patterns and learning from behavior, a Feed could proactively schedule a meeting preparation task *before* a calendar invite is even accepted, transforming the assistant from a reactor to a proactive partner.

### 3. Worker: The Powerful Execution Layer

If Feeds are the senses and the Scheduler is the brain, **Workers are the hands**—specialized agents that get things done. When the Scheduler decides on an action, it delegates to a Worker.

Each Worker is a capable agent, equipped with a suite of tools through MCP. These tools grant them the ability to perform real-world actions like reading and sending emails, managing calendar events, searching for information, or setting reminders.

We support both stateful and stateless worker to facilitate different use cases.

- **Stateful Workers: Specialists with Memory**
    
    Designed for long-running tasks where context matters, Stateful Workers maintain their own “mini-world” with dedicated memory. They are ideal for managing an email thread, tracking a task list, or coordinating reminders. Memory is persisted in the Context Store as an `AgentState`, which stores event history and a running summary. When resuming, the Worker reloads its state to continue seamlessly:
    
    The stateful worker retrieve it’s past agent state via:
    
    ```python
    # The worker's memory is defined by its state, containing its event history.
    class AgentState:
        events: List[Event]
        summary: Optional[str] = None
    
    # Before acting, the worker retrieves its last known state from the Context Store.
    latest_contexts = await context_store.query(
        ContextFilter(namespaces=["memory-email-agent"], limit=1)
    )
    if latest_contexts:
        # Load the previous state to resume the conversation where it left off.
        self.state = AgentState(**latest_contexts[0].data)
    ```
    
- **Stateless Workers: On-Demand Executors**
    
    In contrast, Stateless Workers are ideal for discrete, single-use jobs that require a definitive result. They are ephemeral and highly efficient for tasks like answering a one-off question or performing a quick data lookup.
    

These workers use their tools to achieve the goal and, upon completion, reports its conclusion back to the Scheduler using a dedicated `return_final_answer` tool. 

### 4. Orchestrating Everything with Context Store

The **Context Store** is the real-time data layer that ties everything together. Think of it as the backbone of Pepper—similar to how [feature stores](https://www.uber.com/blog/michelangelo-machine-learning-platform/) power predictive ML systems, but designed for **real-time multi-agent orchestration**.

It handles three key responsibilities:

- **State management**: Agents can persist and share state, enabling coordination and collaboration.
- **Real-time serving**: Feeds publish fresh data that agents can immediately act on.
- **Event orchestration**: Updates trigger downstream actions across components.

At its core, the Context Store offers a simple API to **store**, **retrieve**, and **subscribe** to events—capabilities that are essential for building proactive agents.

Here’s how it all comes together in Pepper:

```python
# 1. Ingestion: A webhook or other trigger stores raw data into a namespace.
@app.on_event("new_email")
async def ingest_raw_event(data: dict):
    await context_store.store(
		    context_id=data.get("id", None) or uuid.uuid(),
        data=data,
        namespace="raw.email"
    )

# 2. Feed: Subscribes to raw events, enriches them, and emits a new signal.
@subscriber.on_context_update(namespaces=["raw.email"])
async def email_feed(update: ContextUpdate):
    # Find related context using semantic search.
    related_docs = await context_store.search(text=update.context.text)
    processed_signal = process_email_signal(update.context, related_docs)

    # Emit the processed signal to a new namespace.
    await context_store.store(
		    context_id="processed_" + update.context.context_id, 
        data=processed_signal,
        namespace="signals.processed"
    )

# 3. Scheduler: Listens for processed signals and adds them to a priority queue.
@subscriber.on_context_update(namespaces=["signals.*"])
async def add_to_queue(self, update: ContextUpdate):
    priority = determine_priority(update.context.data)
    await self.priority_queue.put((priority, update))

# The scheduler's main loop continuously processes tasks from its queue.
async def run_scheduler_loop(self):
    while True:
        await self.scheduler_step()
```

## Conclusion

In this blog post, we present Pepper, a real-time, event-driven architecture that can power a new suite of proactive agentic systems. With Pepper, we envision a future where agents don’t just wait for instructions—they anticipate needs, take action, and drive outcomes.

We’re open-sourcing Pepper and committed to growing it with the community to push forward the next generation of proactive AI.

### Acknowledgement

This work is done with the Agentica team as part of Berkeley Sky Computing Lab. Agentica is supported by Laude Institute, as well as compute grants from AWS and Hyperbolic.

### References

[1] [Introducing ChatGPT Pulse](https://openai.com/index/introducing-chatgpt-pulse/)

[2] [Event-Driven Architecture: How to Implement in Distributed Systems](https://miladezzat.medium.com/event-driven-architecture-how-to-implement-in-distributed-systems-29bd82b02ace)

[3] [Meet Michelangelo: Uber’s Machine Learning Platform](https://www.uber.com/blog/michelangelo-machine-learning-platform/)
