---
author: Paul Elias Sojan
title: Building a multi agent system using Actor model from scratch
date: 2026-01-12
tags:
  - python
  - llm

readingSpeed: 20
readingSpeedMin: 50
readingSpeedMax: 100
---

When I first started working with multi-agent systems, I was fascinated by how independent agents coordinate with each other, how did independent entities communicate? Then I came across a 50-year-old mathematical framework for concurrent computation called Actor model. This blog I'll walk you through how I built a multi-agent system using the Actor Model from scratch.

## Introduction

This project implements a Multi-Agent System (MAS) designed to orchestrate multiple AI agents to solve complex, multi-step problems. By leveraging specialized agents with distinct roles, the system can handle tasks that require research, reasoning, and review more effectively than a single model.

Sometimes handing complex tasks involves multiple steps, and handling all of them with a single model can be inefficient and prone to errors. A MAS addresses this by dividing the workload among specialized agents, each responsible for a specific part of the problem.

> 📍Note:
> **I have implemented the actor model from scratch to understand how it works within a MAS.**

## About Actor model

The Actor Model is the foundation of this MAS. It is a model for designing concurrent and distributed systems. An actor is like a tiny autonomous worker. It has a state, behavior and a mailbox (queue of incoming messages).

Communication in the Actor Model is asynchronous. They never directly access each other’s state, they communicate by sending messages. When an actor receives a message, it can do only three things, send messages to other actors, create new actors, change its own behavior/state. This strict separation and asynchronous communication naturally eliminates many concurrency issues, making the system robust and scalable.

![Actor model](/images/multi_agent/actor_model.png)

Actor model follows `"Let it crash"` approach. So instead of handling every possible error within an actor's logic it let the actor to fail when it encounters an unexpected error. Since each actor is isolated, failure in one actor does not directly affect others. Errors can be managed using a supervision strategy, where a supervisor actor monitors its child actors and decides how to respond on failures.

## Architecture

![Architecture](/images/multi_agent/architecture.png)

Here I followed `Hierarchical Architecture` pattern. It is built around one central controller (the `Orchestrator`) and several specialized worker agents.

Each agent is an independent actor. At the top of the hierarchy is the Orchestrator. It acts as the supervisor. It receives the user’s input, decides which agent should handle the next step, sends the message to that agent, and agents send their results back up to their direct supervisor. All agents use a shared `ModelAdapter`, which has access to the underlying LLM.

The hierarchical approach works best for problems with natural decomposition into sub-problems.

There are many architecture available need to choose architecture that that best matches your primary use case.

![Comparison](/images/multi_agent/comparison.avif)

Checkout my github repo: https://github.com/paulsojan/multi-agent-ai

## Design overview

[actor.py](https://github.com/paulsojan/multi-agent-ai/blob/1-created-multi-agent-ai/core/actor.py)

```python
class Actor:
    def __init__(self):
        self.mailbox = Queue()
        start(self.run)

    def send(self, msg):
        self.mailbox.put(msg)

    def run(self):
        while True:
            self.receive(self.mailbox.get())

    def receive(self, msg):
        pass
```

The `Actor` class represents an independent, self-contained unit that has its own `mailbox` and processes messages in FIFO from its mailbox. It communicates exclusively through the `send()` method, ensuring complete isolation from other actors.

---

[registry.py](https://github.com/paulsojan/multi-agent-ai/blob/1-created-multi-agent-ai/registry/agent_registry.py)

```python
class Registry:
    agents = {}

    def register(self, name, agent):
        self.agents[name] = agent

    def get(self, name):
        return self.agents[name]
```

The `Registry` is a directory for all agents in the system. Agents are registered with a name using `register()`, and the orchestrator or other actors can retrieve them dynamically via get().

---

```python
class Agent(Actor):
    def __init__(self, orchestrator):
        self.orchestrator = orchestrator

    def receive(self, msg):
        result = process(msg)
        self.orchestrator.send(result)
```

`Agent` class is an actor that performs a specific task. It receives a message, processes it (e.g., via an LLM), and then puts the message into the orchestrator’s mailbox.

---

[orchestrator.py](https://github.com/paulsojan/multi-agent-ai/blob/1-created-multi-agent-ai/orchestrator/__init__.py)

```python
class Orchestrator(Actor):
    def __init__(self, registry, workflow):
        super().__init__("Orchestrator")
        self.registry = registry
        self.workflow = workflow   # ordered list of agents
        self.step = 0

    def receive(self, message):
        agent_name = self.workflow[self.step]
        agent = self.registry.get(agent_name)
        self.step += 1
        agent.send(message)

```

The `Orchestrator` is a supervisor actor that controls the workflow of the system. It takes a message from its mailbox, looks up the next agent in the predefined workflow using the Registry, forwards the message to that agent, and advances the step. Ensuring a hierarchical execution of tasks.

---

[main.py](https://github.com/paulsojan/multi-agent-ai/blob/main/main.py)

```python
registry = Registry()

registry.register("TaskNormalizationAgent", task_agent)
registry.register("PlannerAgent", planner_agent)
registry.register("ResearchAgent", research_agent)
registry.register("ExecutorAgent", executor_agent)

workflow = ["TaskNormalization", "Planner", "Research", "Executor"]

orchestrator = Orchestrator(registry, workflow)

orchestrator.send(UserQuery("Self evolving AI"))
```

First, register all agents to the `Registry`. Then, a workflow list defines the hierarchical order in which agents to process. The `Orchestrator` is created with the registry and workflow, acting as the supervisor that routes messages through the agents. Finally, sending a `UserQuery` to the Orchestrator.

## Conclusion

Multi‑Agent Systems represent a powerful paradigm shift in AI. By orchestrating multiple specialized agents, MAS can tackle complex problems that single agents struggle with. While this implementation demonstrates the core concepts, there is still much more to explore—from dynamic agent creation and adaptive workflows to multi‑orchestrator hierarchies and emergent agent behaviors.
