# Jenkins Agent Test — Build Survival During Controller Drain

This test goes further than the basic drain test: a Jenkins build is actively running on a WSL agent when the controller pod is drained. The build survives the controller restart because:

1. The `workflow-durable-task-step` plugin writes `sh` step output to a durable file on the agent side.
2. The Jenkins controller restores build state from its home PVC on restart.
3. The WSL agent reconnects automatically and the build resumes from where it left off.

This simulates a real maintenance scenario: someone patches the OS on the server running the Jenkins controller node during a running CI job.

## Prerequisites

1. k3d dev cluster running with all on-premises apps healthy.
2. Jenkins accessible at http://jenkins.localhost:8080.
3. Docker available on WSL (used by k3d, so it already is).

## Step 1 — Connect the WSL Agent

### 1a. Get the Agent Secret

Retrieve it via the API (no UI needed):

```bash
JENKINS_PASS=$(kubectl exec --namespace jenkins svc/jenkins -c jenkins \
  -- /bin/cat /run/secrets/additional/chart-admin-password)

AGENT_SECRET=$(curl -s -u "admin:${JENKINS_PASS}" \
  "http://jenkins.localhost:8080/computer/wsl-agent/slave-agent.jnlp" \
  | grep -oP '(?<=<argument>)[0-9a-f]{64}(?=</argument>)' | head -1)

echo "Secret: ${AGENT_SECRET:0:8}..."
```

Or navigate in the Jenkins UI: **Manage Jenkins → Nodes → wsl-agent → Status**.

### 1b. Start the Agent as a Docker Container

The `jenkins/inbound-agent` image includes Java and the agent JAR — no local Java needed.
Run in a dedicated terminal; keep it running throughout the test.

```bash
docker run --rm --name jenkins-wsl-agent \
  --network host \
  jenkins/inbound-agent:latest \
  -url http://jenkins.localhost:8080 \
  -secret "${AGENT_SECRET}" \
  -name wsl-agent \
  -webSocket
```

`--network host` lets the container reach `jenkins.localhost:8080` via the host's `/etc/hosts`.

The agent reconnects automatically if the controller is temporarily unavailable: the `inbound-agent` image retries until it can re-establish the WebSocket connection.

### 1c. Verify the Agent Is Online

```bash
JENKINS_PASS=$(kubectl exec --namespace jenkins svc/jenkins -c jenkins \
  -- /bin/cat /run/secrets/additional/chart-admin-password)

curl -s -u "admin:${JENKINS_PASS}" \
  "http://jenkins.localhost:8080/computer/wsl-agent/api/json?tree=offline" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print('online' if not d['offline'] else 'offline')"
```

Expected output: `online`

Or check the Jenkins UI: **Manage Jenkins → Nodes → wsl-agent** — status should show **Connected** (green).

## Step 2 — Create the Test Pipeline Job

1. In Jenkins, click **New Item**.
2. Name it `agent-drain-test`, select **Pipeline**, click **OK**.
3. In the Pipeline section, paste this script:

```groovy
pipeline {
  agent { label 'wsl-agent' }

  stages {
    stage('Pre-drain') {
      steps {
        // Use single-quoted strings for sh steps — Groovy does not interpolate them.
        // Double-quoted strings with $(cmd) cause a compile error.
        sh 'echo "Build started at: $(date)"'
        sh 'echo "Running on host: $(hostname)"'
      }
    }

    stage('Long Running Task') {
      steps {
        sh '''
          echo "Starting long task. Drain the Jenkins controller node now."
          echo "You have ~90 seconds..."
          for i in $(seq 1 9); do
            echo "  tick $i / 9  ($(date +%T))"
            sleep 10
          done
          echo "Long task completed successfully at: $(date)"
        '''
      }
    }

    stage('Post-drain') {
      steps {
        sh 'echo "Build survived the drain and completed! $(date)"'
      }
    }
  }

  post {
    success { echo 'SUCCESS — the build survived the drain.' }
    failure { echo 'FAILURE — something went wrong.' }
  }
}
```

4. Click **Save**.

## Step 3 — Start the Build

Click **Build Now** on the `agent-drain-test` job.

Wait until the console output shows:
```
Starting long task. Drain the Jenkins controller node now.
You have ~90 seconds...
  tick 1 / 9  (HH:MM:SS)
```

You now have approximately 80 seconds to drain the Jenkins controller node.

## Step 4 — Drain the Jenkins Controller Node

In a new terminal, find which node Jenkins is running on:

```bash
kubectl get pod -n jenkins -o wide
```

Drain it:

```bash
# Replace k3d-dev-agent-0 with the actual node name
kubectl drain k3d-dev-agent-0 \
  --ignore-daemonsets \
  --delete-emptydir-data
```

Watch the controller pod move:

```bash
kubectl get pod -n jenkins -w
```

You should see the Jenkins controller pod terminate and be recreated on a different node. The startup takes about 30–60 seconds.

## Step 5 — Observe the Agent Reconnect

While Jenkins is restarting, look at the WSL terminal running the agent. You will see:

```
==> Agent disconnected. Retrying in 10 seconds...
==> Connecting agent to Jenkins at http://jenkins.localhost:8080...
```

Once Jenkins is back online, the agent reconnects:
```
INFO: Connected
```

## Step 6 — Verify the Build Completed

1. Open http://jenkins.localhost:8080.
2. Navigate to the `agent-drain-test` job.
3. Open the most recent build (should be `#1`).
4. Check the console output — you should see all ticks printed and the final messages:
   ```
   Long task completed successfully at: ...
   Build survived the drain and completed!
   SUCCESS — the build survived the drain.
   ```

The build status should be **Success** (green).

## Step 7 — Restore the Node

```bash
kubectl uncordon k3d-dev-agent-0
kubectl get nodes   # all nodes should be Ready
```

---

## How Build Survival Works

The key is the interaction between three components:

1. **`workflow-durable-task-step` plugin**: when a Pipeline `sh` step is executing, the plugin writes all output to a durable log file on the **agent's** filesystem (`$WORK_DIR/remoting/...`). The agent process keeps running even when the controller is down.

2. **Jenkins controller restart from PVC**: when the controller pod moves to a new node, it reads its entire state from the Longhorn PVC — including the in-progress build's Pipeline execution state.

3. **Agent reconnection**: the controller knows the build was in progress. When the WSL agent reconnects, the controller resumes reading from the durable log files and continues the build from where it was.

The `sh 'sleep 10'` loop was still running on the WSL agent the whole time. The controller just came back and picked up where it left off.

---

## Troubleshooting

**Build shows as aborted / failed after drain:**
- Check that the controller restarted within the 90-second window. If it took too long, the agent may have stopped retrying. Reduce drain time by pre-pulling the Jenkins image: `docker pull jenkins/jenkins:lts`.
- Verify `workflow-durable-task-step` is installed: **Manage Jenkins → Plugins → Installed**.

**Agent does not reconnect after drain:**
- Confirm the Docker container is still running: `docker ps --filter name=jenkins-wsl-agent`.
- Check Jenkins at http://jenkins.localhost:8080 — wait for it to be fully ready (progress bar gone).
- The controller may need another ~30 seconds after its health check passes before accepting agent connections.

**Build resumes but shows as failed:**
- Check the build console for the actual error.
- A common cause: the `WORKSPACE` directory on the agent was cleaned between reconnects. Add a `checkout scm` or file creation step at the start to verify workspace state.
