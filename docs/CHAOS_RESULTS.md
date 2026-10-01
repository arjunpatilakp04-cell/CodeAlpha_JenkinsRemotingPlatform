# Phase 8 - Failure Simulation / Chaos Lab Results

## Test 1: Agent crash and recovery
Action: docker kill agent-linux-x86 (SIGKILL from host, terminates the container)
Result: Container disappeared from `docker ps`. `docker compose up -d` recreated it
with a fresh uptime. Agent log showed INFO: Connected shortly after.
Verdict: PASS - agent crash is recoverable via the compose supervisor.

## Test 2: Controller crash and recovery
Action: docker kill jenkins-controller
Result: Container disappeared from `docker ps`. `docker compose up -d` recreated it,
returned to (healthy) status within ~90s. All three agents (linux-x86, linux-x86-2,
linux-arm64) independently reconnected (INFO: Connected in each log) with zero manual
intervention. All jobs, build history and node configuration were intact afterward,
confirming the jenkins_home volume survives container recreation.
Verdict: PASS - controller crash is recoverable with no data loss and automatic
agent reconnection.

## Test 3: Node unavailable / job rerouting
Action: docker stop agent-linux-x86-2, then triggered spread-test (label: x86) three
times via Build Now.
Result: Builds #9 and #10 ran on linux-x86 (confirmed via build log grep). Build #11
queued with "Waiting for next available executor on x86" until an executor freed up,
then also ran on linux-x86. No build was misrouted or silently dropped. Agent was
then restarted (docker start) and reconnected cleanly (INFO: Connected).
Verdict: PASS - jobs correctly reroute to available nodes matching the label; an
unavailable node is excluded from scheduling without causing failures elsewhere.

## Observed anomaly: in-namespace SIGKILL to PID 1
Action: docker exec <container> sh -c "kill -9 1"
Result: The kill command reported exit code 0 (success) in both the agent and
controller containers, but PID 1 did not actually terminate - container uptime was
unaffected even after waiting 100+ seconds. By contrast, docker kill <container> from
the host reliably terminated the container every time.
Verdict: UNRESOLVED / DOCUMENTED - the exact cause was not conclusively identified
(possible causes include container runtime signal handling specifics under Docker
Desktop's backend). This is not a security control and should not be relied upon;
it is noted here as an observed platform quirk for anyone reproducing these tests.
Practical implication: use docker kill (host-level) rather than in-container kill -9 1
when testing crash recovery.

## Key takeaway
Docker's `restart: unless-stopped` policy does NOT auto-restart a container after a
deliberate `docker kill` or `docker stop` - Docker treats these as intentional actions.
Recovery in this lab came from manually re-running `docker compose up -d`. In a
production deployment, this role would be filled by an orchestrator's liveness and
readiness probes (Kubernetes, Docker Swarm) or an external supervisor/health-check
script, not by the restart policy alone.
