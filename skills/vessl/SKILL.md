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
  cluster: umn-choilab-cluster   # ALWAYS use this cluster
  preset: rtx-6000-1-30cpu-100gbram   # see preset table below

image: quay.io/vessl-ai/torch:2.3.1-cuda12.1

import:
  /root/code: git://github.com/<org>/<repo>
  /root/data: vessl-dataset://<org>/<dataset>

export:
  /root/output: vessl-artifact://<org>/<project>/<artifact>

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

**Full YAML example (all features):**
```yaml
name: stable-diffusion-finetune
description: Fine-tune stable diffusion on custom dataset
tags: ["A100", "finetune"]

resources:
  cluster: umn-choilab-cluster    # ALWAYS use this cluster
  preset: rtx-6000-2-60cpu-200gbram   # see preset table below
  # node_names: ["n01", "n03"]    # optional: pin to specific nodes

image: quay.io/vessl-ai/torch:2.3.1-cuda12.1
# Private registry:
# image:
#   url: my-docker-account/private-repo:tag
#   credential_name: docker_hub_cred

# --- Import: download into container at start ---
import:
  /root/code: git://github.com/org/repo          # shorthand
  # /root/code:                                    # verbose git import
  #   git:
  #     url: https://github.com/org/repo
  #     ref: c0ffee                                # branch/tag/commit
  #     credential_name: my-git-cred
  /root/data: vessl-dataset://<org>/<dataset>
  /root/model: vessl-model://<org>/<model_repo>/<model_number>
  /root/s3data: s3://<bucket>/<path>
  /root/gsdata: gs://<bucket>/<path>

# --- Mount: persistent direct filesystem access ---
mount:
  /mnt/dataset: vessl-dataset://<org>/<dataset>
  /mnt/hostpath: hostpath://<path>
  /mnt/nfs: nfs://<server>/<path>

# --- Export: upload from container after run ---
export:
  /root/output: vessl-artifact://                  # auto-create artifact
  /root/model-out: vessl-model://<org>/<model_repo>
  /root/s3out: s3://<bucket>/<prefix>

run:
  - workdir: /root/code
    command: pip install -r requirements.txt
  - wait: 5s
  - workdir: /root/code
    command: python train.py --lr=$learning_rate --bs=$batch_size

# --- Interactive mode (Jupyter + SSH, no run commands) ---
# interactive:
#   max_runtime: 24h              # 0 = infinite
#   jupyter:
#     idle_timeout: 120m

ports:
  - name: tensorboard
    type: http
    port: 6006

env:
  learning_rate: "0.001"
  batch_size: "64"
  postgres_password:
    value: OUR_DB_PW
    secret: true
```

**Available resource presets on `umn-choilab-cluster`:**

| Preset | Processor | GPU | CPU | Memory |
|--------|-----------|-----|-----|--------|
| `cpu-2core-4gbram` | CPU | — | 2 | 4 Gi |
| `cpu-4core-8gbram` | CPU | — | 4 | 8 Gi |
| `rtx-6000-1-30cpu-100gbram` | GPU | 1x RTX 6000 Ada | 30 | 100 Gi |
| `rtx-6000-2-60cpu-200gbram` | GPU | 2x RTX 6000 Ada | 60 | 200 Gi |
| `rtx-6000-3-90cpu-300gbram` | GPU | 3x RTX 6000 Ada | 90 | 300 Gi |
| `rtx-6000-4-120cpu-400gbram` | GPU | 4x RTX 6000 Ada | 120 | 400 Gi |

Default for training: `rtx-6000-1-30cpu-100gbram`. Use multi-GPU presets for distributed training or large models.

**Run status lifecycle:** Pending → Initializing → Running → Stopping → Completed/Failed/Idle/Terminated

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

VESSL has a built-in hyperparameter sweep engine:

```bash
vessl sweep create \
  -T maximize -M accuracy -G 0.95 \
  --num-experiments 20 --num-parallel 4 \
  -a bayesian \
  -p learning_rate double space 0.0001 0.01 \
  -p batch_size int list 32 64 128 \
  -p dropout categorical list 0.1 0.2 0.3 \
  -c <cluster> -r <preset> \
  -i quay.io/vessl-ai/torch:2.3.1-cuda12.1 \
  -x "python train.py --lr=\$learning_rate --bs=\$batch_size --dropout=\$dropout"

vessl sweep list                    # list sweeps
vessl sweep read <NAME>             # details
vessl sweep logs <NAME>             # logs
vessl sweep best-experiment <NAME>  # winner
vessl sweep terminate <NAME>        # stop
```

Algorithms: `grid`, `random`, `bayesian`.

Alternatively, generate multiple YAML files and submit in parallel.

## Non-Interactive Auth

For scripts/automation, configure without prompts:
```bash
vessl configure -t <ACCESS_TOKEN> -o <ORGANIZATION> -p <PROJECT>
```

Environment variables (highest priority):
- `VESSL_ACCESS_TOKEN` — API token
- `VESSL_DEFAULT_ORGANIZATION` — default org
- `VESSL_CREDENTIALS_FILE` — path to credentials file

Default credentials stored at `~/.vessl/config`.

## Service Deployment

Deploy models as endpoints:
```bash
vessl service create -f serve.yaml -s <service-name>
vessl service list
vessl service read --service <NAME> -d
vessl service scale --service <NAME> --min 1 --max 3 --metric gpu --target 50
vessl service terminate --service <NAME>
```

## Claude-Ready Containers

New VESSL containers can auto-bootstrap Claude Code + full config (skills, hooks, MCP servers) with a single init command.

**Bootstrap** — the `setup-claude.sh` script lives in the `dot-claude` repo (private). Since it's private, the VESSL run must clone the repo using a GitHub token, then run the script:

```yaml
run:
  - command: |
      git clone https://${GITHUB_TOKEN}@github.com/benchoi93/dot-claude.git ~/.claude
      bash ~/.claude/setup-claude.sh
```

The script is idempotent (safe to re-run) and does:
1. Installs Claude Code CLI via npm
2. Pulls latest `dot-claude` if already cloned (all skills, hooks, settings)
3. Sets up refcheck MCP server (venv + deps)
4. Generates `.mcp.json` from template
5. Symlinks VESSL auth if `$HOME != /root`
6. Validates `ANTHROPIC_API_KEY`

**Required env vars** (set in VESSL YAML with `secret: true`):
```yaml
env:
  ANTHROPIC_API_KEY:
    secret: true
  GITHUB_TOKEN:
    secret: true
  CROSSREF_EMAIL:
    value: "chois@umn.edu"
```

- `ANTHROPIC_API_KEY`: Claude Code reads this from environment — no interactive login needed
- `GITHUB_TOKEN`: GitHub personal access token for cloning private `dot-claude` repo
- `CROSSREF_EMAIL`: For the refcheck MCP server (optional)

**Templates** are in `skills/vessl/templates/`:
- `claude-workspace.yaml` — interactive GPU workspace with Jupyter + Claude
- `claude-run.yaml` — training run with Claude available

When the user asks for a "Claude-ready workspace" or "workspace with Claude", use these templates as the base and customize the `import`, `run`, and `preset` fields.

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
