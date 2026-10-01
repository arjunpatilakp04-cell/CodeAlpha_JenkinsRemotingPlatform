# CodeAlpha_JenkinsRemotingPlatform

A self-hosted, multi-architecture Jenkins controller/agent platform built with Docker Compose and Jenkins Configuration as Code (JCasC). Built for **CodeAlpha DevOps Internship — Task 2: Jenkins Remoting Project**.

## What this demonstrates

The CodeAlpha brief for this task asks for: remote node connection, distributed build load, multi-architecture jobs, node isolation for security, and hands-on remote execution experience. This project meets and extends every one of those:

- **Remote node connection** — a Jenkins controller running in its own container, with separate inbound agent containers connecting to it over WebSocket, configured declaratively via JCasC rather than clicked together by hand.
- **Multi-architecture execution** — two architecture families of agents (amd64 and arm64, with a second x86 node for load distribution), proven with a real pipeline that runs parallel stages labeled by architecture and prints `uname -m` to confirm each stage actually executed on the correct hardware/emulated platform.
- **Distributed build load** — a real pipeline (not a toy example) that builds in parallel across nodes, hands artifacts between them via `stash`/`unstash`, runs a security scan stage, computes checksums, and archives the results.
- **Node isolation for security** — verified directly, not assumed: agent containers cannot read `/var/jenkins_home` even when run from inside the agent container itself, meaning a fully compromised agent has no path to controller secrets, job configs, or credentials.
- **Hands-on remote execution** — every node, secret, and connection in this repo was built, broken, debugged, and fixed manually rather than scaffolded from a template.

Beyond the brief's minimum, this project also includes:
- A full written **threat model** (`docs/THREAT_MODEL.md`) — 13 numbered threats, each mapped to a specific mitigation and a specific piece of evidence gathered while building this
- A **chaos-engineering test suite** (`docs/CHAOS_RESULTS.md`) — deliberately killing the controller and agents to prove real recovery, including an honestly-documented unresolved anomaly rather than a sanitized "everything worked" writeup

## Architecture

```text
                     ┌─────────────────────┐
                     │  Jenkins Controller  │
                     │  (JCasC-configured)  │
                     └──────────┬───────────┘
                                │ WebSocket
          ┌─────────────────────┼─────────────────────┐
          │                     │                      │
   ┌──────▼──────┐      ┌───────▼───────┐      ┌───────▼───────┐
   │ linux-x86   │      │ linux-x86-2   │      │ linux-arm64   │
   │ agent       │      │ agent         │      │ agent         │
   │ (amd64)     │      │ (amd64)       │      │ (aarch64)     │
   └─────────────┘      └───────────────┘      └───────────────┘
```

All services run via Docker Compose on a shared bridge network (`jenkins-net`). Agent secrets are sourced from a gitignored `.env` file, never hardcoded.

## Repository structure

```text
.
├── agent/                  Dockerfile for building the Jenkins inbound agent image
├── controller/
│   ├── Dockerfile           Controller image, extending the official Jenkins image
│   ├── casc/jenkins.yaml    Jenkins Configuration as Code — security realm, authorization,
│   │                        system message, and declared nodes
│   └── plugins.txt          Pinned plugin list (CasC, job-dsl, matrix-auth, credentials-binding,
│                            docker-plugin, prometheus, role-strategy, etc.)
├── docker/
│   └── docker-compose.yml   Defines the controller and all agent services
└── docs/
    ├── THREAT_MODEL.md      Security threat/mitigation/evidence table
    └── CHAOS_RESULTS.md     Chaos-engineering test results (crash + recovery testing)
```

## Running it locally

1. Create a `.env` file (not committed — see `.gitignore`) with:
   ```
   JENKINS_ADMIN_USER=admin
   JENKINS_ADMIN_PASSWORD=<your choice>
   AGENT_X86_SECRET=<generated after first controller boot, from the node's page>
   AGENT_X86_2_SECRET=<same>
   AGENT_ARM64_SECRET=<same>
   ```
2. From `docker/`, run:
   ```bash
   docker compose up -d
   ```
3. Visit `http://localhost:8081`, log in with the admin credentials above.
4. Agent secrets are generated per-node on first boot — copy each from **Manage Jenkins → Nodes → [node name]**, update `.env`, then restart the agent containers.

## Proven, not just configured

Everything in this repo was validated against a running stack, not just written and assumed correct:

- A real pipeline ran parallel `amd64`/`arm64` stages and confirmed correct architecture routing via console output
- The controller was killed (`docker kill`) and recovered automatically via `docker compose up -d`, with all agents self-reconnecting with zero manual intervention
- An agent was stopped mid-test and Jenkins correctly rerouted queued builds to the remaining node of the same label, confirmed via build history and logs
- Agent-to-controller filesystem isolation was tested directly from inside a running agent container, not assumed from configuration alone
- CSRF protection and anonymous-access lockdown were both verified with `curl` (no browser cookies), after an initial false alarm was caught and corrected

See `docs/THREAT_MODEL.md` and `docs/CHAOS_RESULTS.md` for the full detail and evidence behind each of these.

## Note on scope

Prometheus and Grafana monitoring, and ephemeral Docker Cloud agents, were explored as self-imposed extension goals beyond this task's requirements and are not included in this repository's scope — this repo focuses specifically on what CodeAlpha Task 2 asks for, fully evidenced.
