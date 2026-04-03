---
name: vessl
description: Manage VESSL AI Kubernetes runs, workspaces, and clusters from Claude Code. Use when the user says "vessl", "submit job", "GPU job", "run on cluster", "launch training", "create workspace", "check run status", or wants to execute ML workloads on remote GPU infrastructure.
---

# VESSL AI Job Manager

Manage the full lifecycle of VESSL runs and workspaces: authenticate, submit jobs, monitor, stream logs, SSH, and terminate — all from this machine.

## Prerequisites

1. **CLI installed**: `pip install --upgrade vessl`
2. **Authenticated**: `vessl configure` (sets access token, organization, project)
3. **Verify**: `vessl whoami` must return user info

If `vessl whoami` fails, tell the user to run `! vessl configure` interactively.

## Core Workflows

### 1. Check Status / Auth

```bash
vessl whoami                    # current user + org + project
vessl cluster list              # available clusters
vessl run list                  # recent runs in current project
vessl workspace list            # active workspaces
```

### 2. Submit a Run (GPU Job)

Runs are the primary unit of work. Always use YAML files for reproducibility.

**Step-by-step:**
1. Gather from user: task description, GPU requirements, Docker image, commands, code repo, data volumes
2. Generate a YAML config file (see template below)
3. Submit: `vessl run create -f <yaml_file> -w true`
4. The `-w` flag streams logs after scheduling

**Minimal YAML template:**
```yaml
name: <descriptive-name>
description: <what-this-run-does>
tags:
  - <project-tag>

resources:
  cluster: <cluster-name>        # from `vessl cluster list`
  preset: <resource-preset>      # e.g., gpu-a10-small, gpu-l4-small

image: quay.io/vessl-ai/torch:2.3.1-cuda12.1

volumes:
  /root/code:
    import: git://<repo-url>     # clone repo into container
    ref: main                    # branch/tag/commit
  /root/data:
    import: vessl-dataset://<dataset-name>
  /root/output:
    export: vessl-artifact://<artifact-name>

env:
  WANDB_API_KEY:
    value: <key>
    secret: true
  EPOCHS:
    value: "100"

run:
  - workdir: /root/code
    command: |
      pip install -r requirements.txt
      python scripts/train.py --config configs/default.yaml

ports:
  - name: tensorboard
    type: http
    port: 6006
```

**Full YAML Reference (all fields):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Run name |
| `description` | string | no | Run description |
| `tags` | list | no | Filtering tags |
| `resources.cluster` | string | no | Target cluster (default: VESSL managed) |
| `resources.preset` | string | yes* | Resource preset name |
| `resources.node_names` | list | no | Specific node targets |
| `image` | string/map | yes | Container image URL |
| `image.url` | string | - | Image URL (if using map form) |
| `image.credential_name` | string | - | For private registries |
| `volumes.<path>.import` | string | no | Download source to path (git://, s3://, gs://, vessl-dataset://, vessl-model://, vessl-artifact://) |
| `volumes.<path>.mount` | string | no | Mount filesystem (vessl-dataset://, hostpath://, nfs://) |
| `volumes.<path>.export` | string | no | Upload path after run (vessl-artifact://, s3://, gs://) |
| `volumes.<path>.ref` | string | no | Git ref for git:// imports |
| `volumes.<path>.readonly` | bool | no | Read-only mount (default: true) |
| `run[].workdir` | string | no | Working directory |
| `run[].command` | string | yes | Shell command to execute |
| `run[].wait` | string | no | Pause after command (e.g., "10s") |
| `env.<KEY>.value` | string | - | Environment variable value |
| `env.<KEY>.secret` | bool | - | Mark as secret |
| `ports[].name` | string | no | Port label |
| `ports[].type` | string | no | Protocol: http or tcp |
| `ports[].port` | int | no | Port number |
| `interactive.max_runtime` | string | no | Max runtime (0=infinite) |
| `interactive.jupyter.idle_timeout` | string | no | Jupyter idle cutoff |

### 3. Monitor a Run

```bash
vessl run list                           # list all runs
vessl run read <RUN_ID>                  # detailed status
vessl run logs <RUN_ID>                  # full logs
vessl run logs --tail 50 -f <RUN_ID>    # tail + follow logs
```

### 4. Manage Runs

```bash
vessl run terminate <RUN_ID>    # stop a running job
vessl run delete <RUN_ID>       # remove from history
vessl run ssh <RUN_ID>          # SSH into running container
vessl run update <RUN_ID> "new description"
```

### 5. Create a Workspace (Interactive GPU Session)

Workspaces are persistent interactive environments (Jupyter + SSH).

```bash
vessl workspace create \
  --cluster <cluster> \
  --resource <preset> \
  --image-url quay.io/vessl-ai/torch:2.3.1-cuda12.1 \
  --max-hours 24 \
  --port 8888 \
  --init-script "pip install -r requirements.txt" \
  <workspace-name>
```

**Workspace lifecycle:**
```bash
vessl workspace list               # list all
vessl workspace read <ID>          # details
vessl workspace start <ID>         # resume stopped workspace
vessl workspace stop <ID>          # pause (preserves state)
vessl workspace terminate <ID>     # destroy
vessl workspace ssh <ID>           # SSH in
vessl workspace logs <ID>          # view logs
vessl workspace vscode <ID>        # open in VS Code
```

### 6. Cluster & Resource Management

```bash
vessl cluster list                 # available clusters
vessl cluster list-nodes <ID>     # nodes in a cluster
vessl cluster read <ID>           # cluster details
```

### 7. Data & Model Management

```bash
vessl dataset list                 # list datasets
vessl model list                   # list models
vessl volume list                  # list volumes
vessl storage list                 # list storage backends
```

## Common Image Tags

| Image | Use Case |
|-------|----------|
| `quay.io/vessl-ai/torch:2.3.1-cuda12.1` | PyTorch + CUDA 12.1 |
| `quay.io/vessl-ai/torch:2.1.0-cuda11.8` | PyTorch + CUDA 11.8 |
| Custom Docker Hub/ECR | Specify full URL |

## Autonomous Job Submission Protocol

When the user asks to "submit a job" or "run training on VESSL":

1. **Verify auth**: `vessl whoami` — if fails, prompt user to configure
2. **Discover resources**: `vessl cluster list` to find available clusters
3. **Gather requirements** from user or infer from project:
   - What code to run (git repo or local files)
   - GPU type/count needed
   - Docker image / dependencies
   - Data inputs and output destinations
   - Environment variables / secrets
4. **Generate YAML** in the project directory (e.g., `vessl-runs/<name>.yaml`)
5. **Review with user** before submitting (show the YAML)
6. **Submit**: `vessl run create -f <yaml> -w true`
7. **Monitor**: Stream logs, report completion status
8. **Export results**: Check output artifacts

## YAML File Convention

Save YAML configs to `vessl-runs/` directory in the project for version control:
```
project/
  vessl-runs/
    train-baseline.yaml
    train-sweep-lr.yaml
    eval-final.yaml
```

## Experiment Sweep Integration

For hyperparameter sweeps, use VESSL's built-in sweep:
```bash
vessl sweep --help    # check sweep CLI
```

Or generate multiple YAML files programmatically and submit in parallel.

## Troubleshooting

| Issue | Fix |
|-------|-----|
| "Please run `vessl configure`" | Run `! vessl configure` in terminal |
| "Invalid access token" | `vessl configure --renew-token` |
| Run stuck in "pending" | Check cluster capacity: `vessl cluster list-nodes <id>` |
| OOM in container | Use larger preset or reduce batch size |
| Can't find dataset | Check org: `vessl configure list` |

## Safety

- **Always show YAML to user before submitting** — GPU time costs money
- **Never hardcode secrets** in YAML — use `secret: true` or env vars
- **Tag runs** for cost tracking
- **Set `max_runtime`** on interactive workspaces to avoid runaway costs
