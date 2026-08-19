# AI Accelerated Engineering Bootcamp — Session PE: Ship a Mini Golden Path

> [!IMPORTANT]
> **When using this template, create the new repository under **your Slalom GitHub account** (not the org) and set it to **Private** — the org does not allow Codespaces**

<table><tr><td><img src="image.png" width="48" alt="Session PE banner" /></td><td><strong>Use AI-assisted development to pave a production-ready "golden path" for a new service — IaC, CI/CD, and Observability, all as code.</strong></td></tr></table>

## Purpose

This lab puts Platform Engineering practices in your hands. Starting from a running Todo API (Express + React), you will use GitHub Copilot in Agent mode to build every layer of a paved road:

| Layer     | What you build                                                      |
| --------- | ------------------------------------------------------------------- |
| **IaC**   | Dev stack calling the Slalom golden-path module + policy checks     |
| **CI/CD** | Reusable GitHub Actions workflow + required status checks           |
| **PR**    | Clean description, evidence, and a "what changed / why" summary     |

By the end, a new service team can bootstrap production-ready infrastructure, pipelines, and dashboards by adopting a single template.

---

## Environment Setup

This lab runs in **GitHub Codespaces** (recommended — zero setup) or on your **local machine**.

| Option         | How                                                                                                                                                                                                        |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Codespaces** | Click **Code** → **Codespaces** → **Create codespace on main**. Everything is provisioned automatically. [Learn how to use Codespaces →](https://docs.github.com/en/codespaces/getting-started/quickstart) |
| **Local**      | Follow the [Local Setup guide](LOCAL-SETUP.md) to install all required tools before starting.                                                                                                              |

---

## Quick Start (Skill Lab)

1. Click **Use this template** → **Create a new repository**.
2. Open the **Actions** tab — a workflow fires automatically within ~30 s.
3. When it completes, a **GitHub Issue** appears with your first step instructions.
4. Follow each Issue with Copilot Agent mode. Issues chain automatically as you complete steps.

> The lab has **4 steps + a review**. Expect ~90 minutes with Copilot doing the heavy lifting.

---

## If Something Doesn't Happen

| Symptom                        | Fix                                                          |
| ------------------------------ | ------------------------------------------------------------ |
| No workflow run after creation | Refresh Actions tab; confirm Actions are enabled in Settings |
| Workflow passes but no Issue   | Open the run log, look for errors, then re-run               |

---

## Lab Steps at a Glance

| Step                    | Goal                                              | Deliverable                       |
| ----------------------- | ------------------------------------------------- | --------------------------------- |
| **1 — IaC Scaffold**    | Terraform module + tflint + checkov               | PR with `infra/`                  |
| **2 — CI/CD Pipeline**  | Reusable GHA workflow + caller pipeline           | PR with `.github/workflows/`      |
| **3 — Terraform Apply** | OIDC auth + `terraform apply` against AWS dev env | PR extending `golden-path-ci.yml` |
| **4 — Open PR**         | Full PR with evidence + "what changed / why"      | Merged PR link                    |

---

## Tech Stack Snapshot

| Layer              | Technology             |
| ------------------ | ---------------------- |
| Frontend           | React                  |
| Backend            | Node.js / Express      |
| IaC                | Terraform              |
| CI/CD              | GitHub Actions         |
| Policy             | tflint · checkov · OPA |
| Testing            | Jest                   |
| Package management | npm workspaces         |

---

## Repository Layout

```
.
├── .devcontainer/          # Codespaces / Dev Container config
├── .github/
│   ├── copilot-instructions.md   # Copilot context — read this first
│   ├── prompts/            # Reusable Copilot prompt files
│   ├── steps/              # Step instruction markdown files
│   └── workflows/          # GitHub Actions (lab automation + golden path CI)
├── docs/                   # Project context documents
├── infra/
│   ├── modules/todo-service/     # Terraform module (you build this)
│   └── stacks/dev/               # Terraform stack for dev env (you build this)
└── packages/
    ├── backend/            # Express API — the service to pave a road for
    └── frontend/           # React UI
```

---

## License and Conduct

© 2026 Slalom • [MIT License](LICENSE)

<div align="center"><img src="image.png" width="48" alt="Session PE banner" /></div>
