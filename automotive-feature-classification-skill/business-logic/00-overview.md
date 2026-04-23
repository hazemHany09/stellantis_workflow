# 00 — Overview

## Purpose of this document set

These files describe the **business problem** the skill must solve. They do not describe the workflow, the prompts, the agent loop, or any implementation detail. The workflow will be designed in a later pass, once every business concept in these files is agreed upon.

The skill is intended to be **portable**. Any system that can read files, run a web-search tool, run a browser-rendering / scraping tool, and interact with a knowledge-base tool (ingestion + semantic retrieval) should be able to execute it. No cloud-provider names, vendor names, or platform-specific terminology are used in the business logic.

## Business context

The client operates in the automotive industry. The client maintains a catalogue of cars and needs to know, for each car, which advanced features the car supports and **at what implementation level** each supported feature is delivered.

Today the client does this research manually: a human analyst opens a browser, runs searches on the open web (sometimes restricted to a whitelist of websites, sometimes not), reads the resulting pages, and marks each feature as *present / absent* and — when present — as *Basic*, *Medium*, or *High*. Each decision must be backed by a cited source.

This skill automates that workflow end-to-end while keeping the client in control at defined approval points.

## Scope restriction: full-option trims only

The skill only evaluates cars in their **full-option trim** (the highest equipment level offered for that model). Lower trims are out of scope. The client is expected to supply, or confirm, that the target of the analysis is the full-option variant.

## Primary actors

| Actor              | Role                                                                                                                                                           |
| :----------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Client**         | Supplies the car(s) to be analysed, optionally supplies the list of source websites to consult, and approves the source list before ingestion.                 |
| **Agent**          | Executes the research and classification work. Has access only to the tools declared in `07-tools-and-constraints.md`.                                         |
| **Knowledge Base** | External system that stores downloaded source documents, parses them, and answers semantic retrieval queries. Treated as a black box with a defined interface. |

## High-level goal

For every car submitted by the client, produce a structured deliverable that, for each of the \~160 reference parameters, states:

1. Whether the car supports the parameter (Yes / No).
2. If supported, which implementation level applies (Basic / Medium / High — constrained to the levels defined for that parameter).
3. A written justification for the presence decision, citing the source evidence.
4. A written justification for the level decision, citing the source evidence.

## What this document set does **not** cover

* The orchestration workflow (ordering of steps, retries, state management).
* The exact prompts issued to the search / retrieval / classification stages.
* The internal mechanism of the classification "harness" that converts retrieved evidence into a parameter verdict. The harness will be specified separately. These files only assert that such a stage exists and what it consumes and produces.

