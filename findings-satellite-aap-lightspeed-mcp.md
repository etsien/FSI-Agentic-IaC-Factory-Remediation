# Feasibility Report: Agentic Compliance Remediation via Satellite, AAP, and Lightspeed

**Date:** 2026-08-19  
**Scope:** Self-hosted, air-gapped RHEL environments (RHEL 7-10) with optional OpenShift

---

## 1. Objective

Build a single-interface workflow where an operator:

1. Retrieves a Satellite compliance report (from custom SCAP policies).
2. Reviews failed rules and their Ansible remediations.
3. Selects which remediations to apply.
4. Executes the selected fixes through AAP.
5. Re-runs the Satellite scan to confirm compliance.

All steps occur through the Lightspeed agent, backed by a custom skill that calls a Satellite MCP server and an AAP MCP server. The entire system runs in a disconnected environment.

---

## 2. Component Maturity

| Component | Version | Relevant Status |
|---|---|---|
| Red Hat Satellite | 6.19 (GA May 2026) | SCAP compliance scanning: GA. Satellite MCP server: **Tech Preview** (6.18+). |
| Ansible Automation Platform | 2.7 (GA June 2026) | Controller API: GA. AAP MCP server: **Tech Preview** (2.6.4+, carried into 2.7). |
| OpenShift Lightspeed (OLS) | 1.1.2 (GA) | Custom MCP server registration: **Tech Preview** (feature gate). REST API: GA. |
| Ansible Lightspeed Intelligent Assistant | GA (AAP 2.6+) | Chat in AAP UI. Supports self-hosted LLM. No custom tool/skill extensibility. |
| InstructLab / RHEL AI | 1.5 (GA) | Self-hosted LLM fine-tuning and serving via vLLM. |
| MCP Specification | 2026-07-28 | Stateless Streamable HTTP transport. No session state required. |

**Key finding:** Red Hat already ships MCP servers for both Satellite and AAP. Neither is GA yet, but both are functional and testable. OpenShift Lightspeed already accepts custom MCP servers behind a feature gate.

---

## 3. Architecture Options

There are two viable host platforms. The skill and MCP servers work the same way in both.

### 3a. OpenShift Deployment

```
┌─────────────────────────────────────────────────────────────┐
│  OpenShift Cluster                                          │
│                                                             │
│  ┌──────────────────────┐    ┌────────────────────────────┐ │
│  │ OpenShift Lightspeed │───▶│ Satellite MCP Server (Pod) │ │
│  │ Operator (OLS)       │    │ registry.redhat.io/        │ │
│  │                      │    │ satellite/foreman-mcp-     │ │
│  │ OLSConfig:           │    │ server-rhel9               │ │
│  │   mcpServers:        │    └────────────┬───────────────┘ │
│  │   - satellite-mcp    │                 │ HTTPS           │
│  │   - aap-mcp          │                 ▼                 │
│  └──────────┬───────────┘    ┌────────────────────────────┐ │
│             │                │ Satellite Server (ext/int) │ │
│             │                │ Foreman API :443           │ │
│             │                └────────────────────────────┘ │
│             │                                               │
│             │                ┌────────────────────────────┐ │
│             └───────────────▶│ AAP MCP Server (Pod)       │ │
│                              │ Built into AAP install     │ │
│                              └────────────┬───────────────┘ │
│                                           │ HTTPS           │
│                                           ▼                 │
│                              ┌────────────────────────────┐ │
│                              │ AAP Gateway :443           │ │
│                              └────────────────────────────┘ │
│                                                             │
│  ┌──────────────────────┐                                   │
│  │ RHEL AI / vLLM       │  (LLM inference for OLS)         │
│  │ Granite model on PVC │                                   │
│  └──────────────────────┘                                   │
└─────────────────────────────────────────────────────────────┘
```

- OLS connects to MCP servers via in-cluster Service URLs.
- MCP servers connect to Satellite and AAP APIs over HTTPS.
- LLM served by RHEL AI (vLLM) on the same cluster or a dedicated node.

### 3b. Bare RHEL Deployment (No OpenShift)

```
┌────────────────────────┐     ┌────────────────────────┐
│ RHEL Host A            │     │ RHEL Host B            │
│                        │     │                        │
│ Lightspeed Agent       │     │ Satellite Server       │
│ (Podman or systemd)    │     │ Foreman API :443       │
│                        │     └────────────────────────┘
│ Satellite MCP Server   │
│ (Podman container)     │───HTTPS──▶ Satellite API
│                        │
│ AAP MCP Server         │     ┌────────────────────────┐
│ (Podman container)     │     │ RHEL Host C            │
│                        │     │                        │
│ vLLM (Podman or bare)  │     │ AAP (Containerized)    │
│ Granite model on disk  │     │ Gateway API :443       │
└────────────────────────┘     └────────────────────────┘
        │
        └───HTTPS──▶ AAP Gateway API
```

- MCP servers run as rootless Podman containers managed by systemd user services.
- Lightspeed agent interfaces via stdio (local) or Streamable HTTP (remote).
- vLLM runs on a GPU-equipped node or CPU-only with a quantised model.

---

## 4. Workflow Feasibility — Step by Step

### Step 1: Retrieve Compliance Report

| Question | Answer |
|---|---|
| API exists? | Yes. `GET /api/v2/compliance/arf_reports` lists reports with pass/fail counts per host. |
| Filter by policy? | Yes. Search syntax: `compliance_policy_id = <id>`. |
| Get per-rule results? | Partially. The API returns aggregate counts. For rule-level detail, download the ARF XML via `GET /api/v2/compliance/arf_reports/:id/download` and parse the XCCDF `<rule-result>` elements. |
| Satellite MCP server? | Yes. Tech Preview container at `registry.redhat.io/satellite/foreman-mcp-server-rhel9`. Provides inventory and content tools. Compliance-specific tools may need to be added or wrapped. |

**Gap:** The existing Satellite MCP server exposes advisor, inventory, content-view, and diagnostics tools. It may not yet wrap the `/api/v2/compliance/*` endpoints. If it does not, you will need a thin wrapper MCP server (or contribute compliance tools upstream).

**Mitigation:** The Satellite REST API is well-documented. A custom MCP tool that calls the compliance endpoints and parses ARF XML is straightforward to build with the Python MCP SDK (`FastMCP`).

### Step 2: Parse Failed Rules and Remediations

| Question | Answer |
|---|---|
| Does ARF contain remediation data? | Yes. Each failed rule in the SCAP Security Guide data stream includes `<fix>` elements with Ansible or Bash remediation snippets. |
| Can Satellite generate a playbook? | Not via API. The `oscap xccdf generate fix --fix-type ansible` command generates a playbook from scan results, but runs client-side. |
| Pre-built playbooks available? | Yes. SSG ships full-profile playbooks (e.g., `/usr/share/scap-security-guide/ansible/rhel9-playbook-stig.yml`). Individual rule remediations are embedded in the data stream. |

**Approach:** Parse the downloaded ARF XML, extract `<rule-result>` entries with `result=fail`, match each rule ID to its `<fix system="urn:xccdf:fix:script:ansible">` content from the data-stream XML. Present the list to the user.

### Step 3: User Selects Remediations

| Question | Answer |
|---|---|
| OLS supports human-in-the-loop? | Yes. The MCP spec and OLS both support `InputRequiredResult` for pausing execution and requesting confirmation. OLS 1.1 has `toolsApprovalConfig` for write operations. |
| Ansible Lightspeed supports selection? | The chat UI supports conversational interaction but has no structured selection widget. |

**Approach:** The Lightspeed agent presents the failed rules as a numbered list. The user replies with their selections. The skill validates and proceeds.

### Step 4: Execute Remediations via AAP

| Question | Answer |
|---|---|
| API exists? | Yes. `POST /api/controller/v2/job_templates/{id}/launch/` with `extra_vars` for target hosts and rule list. |
| Pass dynamic host list? | Yes. Use `limit` parameter or `extra_vars` with host list. Requires `ask_limit_on_launch` enabled on the template. |
| Monitor job? | Yes. Poll `GET /api/controller/v2/jobs/{id}/` until `status` is terminal (`successful`, `failed`, `error`, `canceled`). |
| Get per-host results? | Yes. `GET /api/controller/v2/jobs/{id}/job_host_summaries/` returns per-host pass/fail/changed counts. |
| AAP MCP server? | Yes. Tech Preview in AAP 2.6.4+. Provides job management, inventory, and compliance tools. Supports PAT and OAuth auth. |

**Prerequisite:** A job template (or workflow template) must exist in AAP that accepts a list of remediation rule IDs and target hosts as `extra_vars`, then runs the corresponding SSG Ansible tasks. This template is a one-time setup item.

### Step 5: Re-run Satellite Scan and Confirm

| Question | Answer |
|---|---|
| Trigger scan via API? | Yes. Use Remote Execution: `POST /api/v2/job_invocations` with the OpenSCAP scan job template. Or invoke the policy's scheduled scan via `PUT /api/v2/compliance/policies/:id` to force a run. |
| Wait for results? | Poll `GET /api/v2/compliance/arf_reports?search=host=<host>&order=created_at+DESC` until a new report appears with a timestamp after the scan was triggered. |
| Compare before/after? | Yes. Retrieve both ARF reports, diff the `<rule-result>` entries. |

---

## 5. What Already Exists

| Item | Status | Notes |
|---|---|---|
| Satellite MCP server container | Tech Preview | `registry.redhat.io/satellite/foreman-mcp-server-rhel9`. May not include compliance tools yet. |
| AAP MCP server | Tech Preview | Deployed within AAP. Job management, inventory, compliance tools included. |
| OpenShift Lightspeed with custom MCP | Tech Preview | Feature gate `MCPServer`. Accepts any MCP-compliant server URL in `OLSConfig`. |
| RHEL AI / InstructLab | GA | Self-hosted LLM. Granite models. vLLM serving. Air-gapped supported. |
| Ansible Lightspeed Intelligent Assistant | GA | Chat in AAP UI. Cannot register custom MCP servers (not extensible today). |
| MCP Python SDK (FastMCP) | Stable | `pip install "mcp[cli]"`. Streamable HTTP and stdio transports. |
| SCAP Security Guide Ansible remediations | Shipped with SSG | Per-rule Ansible fix snippets in data-stream XML. Full-profile playbooks on disk. |

---

## 6. What Needs To Be Built

### 6a. Must Build

| Deliverable | Description | Effort Estimate |
|---|---|---|
| **Compliance MCP tools** | If the Satellite MCP server lacks compliance endpoints: a small Python MCP server (or tool additions) that wraps `/api/v2/compliance/arf_reports`, `/api/v2/compliance/policies`, and ARF XML parsing. ~5-8 MCP tools. | Small-medium |
| **Remediation-extraction logic** | Parse ARF XML + data-stream XML to extract failed rules and their Ansible fix content. Package as an MCP tool or a library called by the skill. | Small |
| **AAP job template** | A parameterised job template (or workflow) in AAP that accepts `rule_ids` and `target_hosts` as extra vars and runs the matching SSG Ansible remediation tasks. | Small |
| **Lightspeed skill / prompt chain** | The orchestration logic that ties the five workflow steps together. In OLS, this is the prompt engineering and MCP tool registration in `OLSConfig`. For a standalone agent, this is a skill definition. | Medium |
| **Deployment automation** | Ansible roles or Helm charts to deploy the MCP servers and configure OLS/agent in both OpenShift and bare-RHEL modes. | Medium |

### 6b. Should Build

| Deliverable | Description |
|---|---|
| **Before/after compliance diff tool** | MCP tool that compares two ARF reports and returns a structured delta (rules fixed, rules still failing, new failures). |
| **Scan-trigger tool** | MCP tool that triggers a Satellite remote-execution OpenSCAP scan and polls for completion, rather than waiting for the next scheduled scan. |
| **Auth integration** | If using OpenShift: leverage Kubernetes ServiceAccount tokens for MCP-to-API auth. If bare RHEL: manage PATs/OAuth tokens via Vault or environment variables. |

### 6c. Nice To Have

| Deliverable | Description |
|---|---|
| **EDA integration** | Event-Driven Ansible rulebook that listens for Satellite compliance-scan-complete webhooks and auto-triggers the remediation workflow without manual initiation. |
| **Fine-tuned model** | Use InstructLab to fine-tune a Granite model on SCAP/compliance domain knowledge for better natural-language interaction. |

---

## 7. Prerequisites

### Infrastructure

| Requirement | OpenShift Path | Bare RHEL Path |
|---|---|---|
| Satellite 6.18+ | External or on-cluster VM | Dedicated RHEL 9 host |
| AAP 2.6.4+ | AAP Operator on OCP 4.14+ | Containerized install on RHEL 9.6+ (Podman) |
| LLM inference | OpenShift AI or RHEL AI pod with GPU | vLLM on a GPU-equipped RHEL host (or CPU with quantised model) |
| Lightspeed agent | OLS Operator on OCP | Custom agent using OLS REST API or standalone MCP client |
| Container registry | Internal registry or Quay | Podman local storage or Private Automation Hub registry |
| Python 3.11+ | In MCP server container | On MCP server host |

### Content (Transfer Across Air Gap)

| Item | Source | Transfer Method |
|---|---|---|
| Satellite MCP server image | `registry.redhat.io` | `podman save` / `podman load` |
| AAP MCP server | Bundled with AAP install | AAP bundle installer |
| OLS Operator images | `registry.redhat.io` | `oc mirror` or `podman save` |
| LLM model weights | Hugging Face / Red Hat | Download, transfer via media, load to PVC or local disk |
| SCAP Security Guide | Red Hat repos | Sync via Satellite ISS Export Sync |
| Custom MCP server code | Your repo | Git bundle or tarball |
| Execution Environment images | Build on connected side | `podman save` / `podman load` to Private Hub |

### Networking (Internal Only)

| From | To | Port | Protocol |
|---|---|---|---|
| Lightspeed agent | Satellite MCP server | 8080 | HTTP (Streamable HTTP) |
| Lightspeed agent | AAP MCP server | 8080 | HTTP (Streamable HTTP) |
| Lightspeed agent | vLLM | 8000 | HTTP (OpenAI-compatible API) |
| Satellite MCP server | Satellite Foreman API | 443 | HTTPS |
| AAP MCP server | AAP Gateway API | 443 | HTTPS |
| Managed hosts | Satellite Capsule | 443, 8000, 8140, 9090 | HTTPS |
| AAP execution nodes | Managed hosts | 22 | SSH |

---

## 8. Risks and Open Questions

| Risk | Severity | Mitigation |
|---|---|---|
| Satellite MCP server may not expose compliance tools | Medium | Build a thin compliance MCP wrapper. The REST API is well-documented and stable. |
| MCP servers are Tech Preview | Medium | Functional today. Plan for API changes. Pin container image versions. Red Hat is actively investing (five MCP servers shipped in 2025-2026). |
| Ansible Lightspeed Intelligent Assistant is not extensible via MCP | High (if using AAP UI as the interface) | Use OpenShift Lightspeed as the agent interface instead. Or build a standalone MCP client agent. |
| LLM quality for compliance domain | Low | Granite models handle structured tool-calling well. Fine-tune with InstructLab if needed. |
| ARF XML parsing complexity | Low | Well-documented SCAP 1.2/1.3 format. Python libraries exist (`openscap`, `lxml`). |
| GPU availability in air-gapped environments | Medium | Use quantised models (GGUF/AWQ) for CPU-only inference, with degraded performance. |

---

## 9. Conclusion

**This workflow is feasible.** The core building blocks exist and are functional today:

- Satellite has a comprehensive compliance API and ships SCAP content with Ansible remediations.
- AAP has a mature job-execution API and a Tech Preview MCP server.
- OpenShift Lightspeed accepts custom MCP servers and works air-gapped with a self-hosted LLM.
- The MCP protocol supports stateless HTTP transport suitable for enterprise networking.
- Red Hat has already shipped MCP servers for both Satellite and AAP.

The primary build effort is:

1. A compliance-focused MCP tool layer (wrapping Satellite compliance API + ARF parsing).
2. An AAP job template that executes SSG Ansible remediations by rule ID.
3. A Lightspeed skill (prompt chain + tool registration) that orchestrates the five-step workflow.
4. Deployment automation for both OpenShift and bare-RHEL targets.

If OpenShift is available, use OLS as the agent interface. If only bare RHEL is available, build a lightweight MCP client agent that calls the same MCP servers and uses vLLM directly.
