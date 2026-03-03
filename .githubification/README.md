# Githubification

### The method by which any repo can execute using GitHub as Infrastructure

<p align="center">
  <picture>
    <img src="https://raw.githubusercontent.com/japer-technology/githubification/main/.githubification/githubification-logo.png" alt="GitHubification" width="500">
  </picture>
</p>

Githubification is the act of converting a repository into GitHub-as-infrastructure. Instead of cloning the repo and running the software elsewhere, the repo becomes something that runs on GitHub itself via GitHub Actions. There’s no separate local runtime to install—GitHub is the runtime.

---

## Three Types of Repo

There are three primary patterns for Githubification, determined by what the repository already contains and how you want it to run.

### Type 1 — AI Agent Repo

The repository already contains an AI agent. Githubification converts that agent's functionality from something that must be installed and run locally into something that **runs natively inside GitHub as an Action**, using GitHub as AI infrastructure.

| Before | After |
|--------|-------|
| Clone the repo, install dependencies, run the agent locally | The agent executes directly on GitHub Actions—no local setup required |

### Type 2 — Non-AI Software Repo

The repository contains software that is **not** an AI agent. Githubification inserts an AI agent into the repo that provides two capabilities:

1. **AI-powered access** — interact with the software's functionality through the AI agent.
2. **GitHub-as-infrastructure execution** — run the software itself on GitHub Actions without local installation.

| Before | After |
|--------|-------|
| Clone the repo, install, configure, and run the software locally | An AI agent exposes the software's capabilities and runs it on GitHub Actions |

### Type 3 — Hybrid / Variations

Not every repo fits neatly into the first two categories. Additional combinations include:

- **AI repo + companion agent** — the repo already has an AI engine, but instead of turning that AI into the execution engine you place a _separate_ AI agent alongside it that can communicate with and orchestrate the existing one.
- **Multi-agent orchestration** — combining several of the above patterns so that multiple agents or software components coordinate through GitHub Actions.
- Any other arrangement where Githubification bridges the gap between "software in a repo" and "software running on GitHub."

---

## Core Idea

> You no longer have to install the software that's in the repo. The repo now has the ability to execute it directly in GitHub using GitHub Actions.

Githubification leverages the same primitives described in this project's root [README](../README.md):

- **GitHub Actions** as compute
- **Git** as storage and memory
- **GitHub Issues** as the user interface
- **GitHub Secrets** as the credential store

By applying these primitives to _any_ repository, Githubification turns GitHub from a place where code is stored into a place where code is **run**.

