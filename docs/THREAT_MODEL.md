# Threat and Mitigation Document

Platform: Secure Multi-Architecture Jenkins Remoting Platform
Scope: Jenkins controller, three agents (2x amd64, 1x arm64 via QEMU), docker-socket-proxy, all run with Docker Compose.

## Threat model summary

| # | Threat | Impact | Mitigation | Evidence |
|---|--------|--------|------------|----------|
| 1 | Unauthenticated access to UI, API or build trigger | Data leak, unauthorized builds | Signup disabled, role-based authorization | HTTP 403 for /, /api/json, /computer/api/json, POST /job/*/build |
| 2 | Default or weak admin credentials | Full takeover | Random 20-char password in gitignored .env, injected via JCasC | Old password rejected, new password authenticates |
| 3 | CSRF against Jenkins | Forged actions by a logged-in admin | Crumb issuer enabled | /crumbIssuer/api/json returns a DefaultCrumbIssuer crumb |
| 4 | Malicious build gains root on agent | Agent and host compromise | Container runs as uid 1000, cap_drop ALL, no-new-privileges | whoami = jenkins, CapEff = 0000000000000000, NoNewPrivs = 1 |
| 5 | Malicious build modifies agent system files | Persistence, tampering | Non-root user, write to /etc denied | touch /etc/testfile: Permission denied |
| 6 | Resource exhaustion (memory bomb, fork bomb) | Host slowdown, other agents disrupted | cgroup limits: 1 GiB memory, 1 CPU, 512 pids | memory.max = 1073741824, pids.max = 512, oom_kill = 1; other agents stayed up |
| 7 | Persistent malware or leftover data on agent | Cross-build contamination | tmpfs workspace wiped on container restart | Marker file present between builds, gone after restart |
| 8 | Compromised agent reads controller data | Secrets and job config exposure | Agents run in separate containers with no controller volume | ls /var/jenkins_home on agent: No such file or directory |
| 9 | Exposed Docker socket | Host takeover | Socket mounted read-only into a filtering proxy only; controller talks to the proxy | Only CONTAINERS, IMAGES, NETWORKS, POST enabled; EXEC, VOLUMES, SYSTEM denied |
| 10 | Unused network attack surface | Extra entry point | Legacy agent port 50000 removed; agents use WebSocket over 8080 | docker port shows only 8080 -> 8081 |
| 11 | Stolen agent secret | Rogue agent joins | Secrets kept in .env (gitignored), unique per node, node names fixed in JCasC | .env excluded from git; wrong secret gives incorrect secret warning |
| 12 | Supply-chain drift from floating image tags | Untested or malicious image | Proxy pinned by digest; agent base pinned to inbound-agent:3384.v60d89463d9e0-1-jdk17 | Compose and Dockerfile show pinned references |
| 13 | Misrouted jobs (wrong architecture or node) | Failed or unsafe builds | Label-based routing, exclusive nodes | routing-test and arch-test outputs; unmatched label stays queued |

## Known limitations and residual risk

- ARM64 agent runs under QEMU emulation: slow, and not a substitute for real ARM hardware.
- Traffic between agents and controller is plain HTTP on a local Docker network. Use TLS for any non-local deployment.
- Jenkins is exposed on localhost:8081 only for local use; put a TLS reverse proxy in front for real use.
- The Docker socket proxy still allows container creation (POST); a compromised controller could start containers.
- The admin password was displayed in a screenshot during setup and should be rotated.
- Secrets in .env are plaintext on disk. Use a secrets manager for production.
- tmpfs workspaces limit build size to available tmpfs space.

## Verification commands

    curl.exe -s -o NUL -w "%{http_code}" http://localhost:8081/api/json
    docker exec agent-linux-x86 id
    docker exec agent-linux-x86 cat /sys/fs/cgroup/memory.events
    docker port jenkins-controller
    docker inspect agent-linux-x86 --format "{{.Config.Image}}"
