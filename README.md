# vessl-skill

A [Claude Code](https://claude.ai/code) skill for managing [VESSL AI](https://vessl.ai) Kubernetes runs, workspaces, and clusters autonomously.

## What it does

- Submit GPU training jobs via YAML configs
- Create and manage interactive workspaces (Jupyter + SSH)
- Monitor runs, stream logs, and manage the full job lifecycle
- Discover clusters, resources, datasets, and models
- Generate reproducible YAML configs for ML workloads

## Installation

### As a Claude Code skill

Copy the `skills/vessl/` directory into your Claude Code skills directory:

```bash
cp -r skills/vessl ~/.claude/skills/vessl
```

Or clone this repo and symlink:

```bash
git clone https://github.com/benchoi93/vessl-skill.git
ln -s $(pwd)/vessl-skill/skills/vessl ~/.claude/skills/vessl
```

### Prerequisites

1. Install the VESSL CLI:
   ```bash
   pip install --upgrade vessl
   ```

2. Authenticate:
   ```bash
   vessl configure
   ```

3. Verify:
   ```bash
   vessl whoami
   ```

## Usage

Once installed, Claude Code will automatically use this skill when you say things like:

- "Submit a training job on VESSL"
- "Create a GPU workspace"
- "Check my VESSL run status"
- "Launch training on the cluster"

The skill guides Claude through:
1. Verifying authentication
2. Discovering available clusters and resources
3. Generating a YAML config for your workload
4. Submitting and monitoring the run

## Example YAML

```yaml
name: train-model
description: Fine-tune transformer on traffic data
resources:
  cluster: vessl-oci-sanjose
  preset: gpu-a10-small
image: quay.io/vessl-ai/torch:2.3.1-cuda12.1
volumes:
  /root/code:
    import: git://github.com/your-org/your-repo
    ref: main
  /root/output:
    export: vessl-artifact://train-results
run:
  - workdir: /root/code
    command: |
      pip install -r requirements.txt
      python train.py --config configs/default.yaml
```

## License

MIT
