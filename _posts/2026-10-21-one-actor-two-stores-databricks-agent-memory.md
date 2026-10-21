---
layout: post
tags:
  - databricks
  - lakehouse
  - architecture
  - agents
---

# One Actor, Two Stores: What Databricks Agent Memory Changes for an Architect

On 20 October 2026, Sirui Sun and Nihit Desai published agent memory on Agent Bricks and called it Beta, on Lakebase. The same day the research team published MemCurrent and said that design now powers Agent Bricks Memory and Genie. The 16 June summit post had already named a Lakebase memory service. That sentence has no preview label and no pipeline. The October notes are the other clock. On 12 October they mark memory pipelines as Beta. The memory, session, and pipeline pages I opened on 21 October were last updated on 17 October, and each says Beta. Azure prints the same banner. The 20 October notes are three Lakeflow Connect connectors, and they have no memory bullet. I did not find a Hacker News thread for either post.

## The actor is a partition. The store is the lock.

Creating either store provisions Lakebase. I wrote about that Postgres engine on 9 October. During the preview the instance is the bill: no extra charge for the feature, and the price can change. The product post says you stand up no new infrastructure. The docs still provision that instance. I would write the meter beside that sentence.

A session is one interaction. `actor_id` is required and has to be set in trusted application code. `session_id` is optional. Items are opaque JSON. An append does not change, and pop, clear, and a parentless fork are still allowed. Deleting a session deletes its children and leaves memory in place.

A memory entry is that same `actor_id`, an optional `session_id`, and a path such as `/preferences/communication.md`. The same path in two sessions is two entries. The store id is 3 to 56 characters, lowercase letters, numbers, and hyphens, starting with a letter and ending with a letter or number. Only the description can change later.

Every list and every search takes one `actor_id`, and that field does not grant access. Any principal that can open the store can read and write every actor. Any workspace user can list the store ids. Only the creator or a workspace admin can grant it. A shared `actor_id` can hold a team glossary. A second store is what the page gives you for a tenant boundary. The page says to reject a caller with no end-user identity, rather than fall back to a shared actor.

The older store is still documented, updated 13 October, and it is a different object. It is a Unity Catalog securable on `/api/2.1/unity-catalog/memory-stores`, with `CREATE MEMORY STORE`, `READ MEMORY STORE`, `WRITE MEMORY STORE`, and `MANAGE`. Paths there start with `/memories/`. The API page says that store will be turned down soon, and the 14 October overview calls it legacy. A new agent belongs on `/api/2.0/agents/memory-stores`. The examples are Python. The same paths are REST.

## Three writers, and four search sentences

MemCurrent names three ways to attach memory. A plugin reads and writes at fixed points. Tools let the agent decide. An offline pipeline consolidates later, and the update arrives late. The product is that split, on the two stores.

The session append is the log. The product post says the agent loads an index of memory at session start, then searches. `add()` fails with `ALREADY_EXISTS` when the path is taken. `put()` creates or replaces, and it is not atomic. `put()` and a path list need `databricks-agentbricks` 0.6.0 or above. Under `agentbricks dev`, sessions stay in the process and memory is off. A call with no actor scopes memory to that one conversation.

The pipeline is the late writer, and it is not a Lakeflow pipeline. One session store feeds one memory store, and a second pipeline on that pair fails with `ALREADY_EXISTS`. The run uses its creator's identity, calls a Unity Gateway model (`system.ai.claude-sonnet-5` in the docs' example, or a default), and merges under the same `actor_id`. Maintenance can delete an entry the agent saved. Actors with no sessions are left alone. A new pipeline does not reuse the old checkpoints, so its first run rereads full history. The schedule is about every 24 hours, with no cron. Past about 1,200 UUID actors the run fails, at 48 KiB of serialized ids. The page says it is designed to skip secrets and live counts. I would still read the store. The run bills serverless general compute and Foundation Model Serving, beside Lakebase.

Search is where the pages do not share a sentence. The 20 October product post says hybrid keyword and semantic search. The 14 October overview says semantic search. The AWS, Azure, and Google Cloud memory pages say BM25 plus semantic matching, at most 100 entries, and no pagination. The developer overview says BM25, and that the rank is not vector similarity. I would not draw a vector index, and I would not let one sentence overwrite the others.

The benchmark stays in the research post. No memory passes 22.3 percent, MemCurrent passes 68.4 percent, and gold injection passes 78.6 percent, on 114 cases from 29 users and 2,233 history sessions. That opening calls this a triple, with 31 percent fewer tokens and 26 percent faster. The product post prints the triple, the 31 percent, and a session that fell from 11 tool calls and nearly six minutes to 2 tool calls and 41 seconds. The research account of that session also counts 8 subagents down to 1. I did not rerun it. That post calls episodic memory append-only. The sessions page also allows pop and clear.

<img src="/assets/images/agent-memory-one-actor-two-stores.svg" alt="Session store and memory store on Lakebase, joined by actor id. A pipeline distills sessions later and can delete memory the agent wrote. The store grant is the access boundary." title="One actor, two stores. The pipeline is the late writer." style="width:100%; height:auto;">

## Where it fits, and where it does not

I would use both stores when a conversation must resume and a preference must outlive it, with `actor_id` fixed in server code and a store per tenant where the review asks for isolation. I would run the pipeline once by hand before trusting the 24-hour clock. The Unity Catalog store stays for an agent that already calls `/api/2.1`.

A secret, a live count, or a warehouse row does not belong in the memory store. The actor stays in server code. `put()` is not a transaction, and a pipeline run can drop a line the agent wrote. Search stops at 100 hits. `agentbricks dev` does not open the stores. The 9 October post is still the database architecture. This is a Beta instance, plus a second meter when the dream runs.

## What goes in the design doc

Write Beta. Pipelines on 12 October, the stores on the 17 October pages, and the 20 October posts are not a general-availability date. Draw two Lakebase stores. The actor partitions. The grant locks. Name the three writers, the 24-hour schedule, the 48 KiB cap, and that maintenance may delete. Keep the four search sentences. Leave the benchmark in the blog. A new agent uses `/api/2.0`.

## References

- Sirui Sun and Nihit Desai, "Introducing Agent Memory on Databricks," Databricks Blog, 20 October 2026. [https://www.databricks.com/blog/introducing-agent-memory-databricks](https://www.databricks.com/blog/introducing-agent-memory-databricks)
- The Databricks AI Research Team, "MemCurrent: Designing Persistent Memory for Production Agents," Databricks Blog, 20 October 2026. [https://www.databricks.com/blog/memcurrent-designing-persistent-memory-production-agents](https://www.databricks.com/blog/memcurrent-designing-persistent-memory-production-agents)
- Hanlin Tang, Kasey Uhlenhuth, Akhil Gupta, and Patrick Wendell, "Agent Bricks: Data + AI Summit 2026," Databricks Blog, 16 June 2026. [https://www.databricks.com/blog/agent-bricks-dais-2026](https://www.databricks.com/blog/agent-bricks-dais-2026)
- Databricks, "October 2026" release notes (AWS). Pipelines are marked Beta on 12 October. The 20 October entries are the SurveyMonkey, Oracle Eloqua, and Asana connectors. [https://docs.databricks.com/aws/en/release-notes/product/2026/october](https://docs.databricks.com/aws/en/release-notes/product/2026/october)
- Databricks, "Agent memory" (AWS), last updated 17 October 2026. [https://docs.databricks.com/aws/en/agents/agent-memory/managed-memory](https://docs.databricks.com/aws/en/agents/agent-memory/managed-memory)
- Databricks, "Agent memory" (Google Cloud). [https://docs.databricks.com/gcp/en/agents/agent-memory/managed-memory](https://docs.databricks.com/gcp/en/agents/agent-memory/managed-memory)
- Microsoft, "Agent memory" (Azure Databricks). [https://learn.microsoft.com/en-us/azure/databricks/agents/agent-memory/managed-memory](https://learn.microsoft.com/en-us/azure/databricks/agents/agent-memory/managed-memory)
- Databricks, "Agent sessions" (AWS), last updated 17 October 2026. [https://docs.databricks.com/aws/en/agents/agent-memory/managed-sessions](https://docs.databricks.com/aws/en/agents/agent-memory/managed-sessions)
- Databricks, "Agent memory pipelines" (AWS), last updated 17 October 2026. [https://docs.databricks.com/aws/en/agents/agent-memory/memory-pipelines](https://docs.databricks.com/aws/en/agents/agent-memory/memory-pipelines)
- Databricks, "Agent memory and sessions" (AWS), last updated 14 October 2026. [https://docs.databricks.com/aws/en/agents/custom-agents/stateful-agents](https://docs.databricks.com/aws/en/agents/custom-agents/stateful-agents)
- Databricks, "Agent memory and sessions," developer overview. [https://developers.databricks.com/docs/agents/memory](https://developers.databricks.com/docs/agents/memory)
- Databricks, "Managed agent memory (legacy)" (AWS), last updated 13 October 2026. [https://docs.databricks.com/aws/en/agents/agent-memory/managed-memory-stores](https://docs.databricks.com/aws/en/agents/agent-memory/managed-memory-stores)
- Databricks, "Memory API reference (legacy)" (AWS), last updated 13 October 2026. [https://docs.databricks.com/aws/en/agents/agent-memory/memory-store-api](https://docs.databricks.com/aws/en/agents/agent-memory/memory-store-api)
