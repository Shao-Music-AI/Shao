# Deployment Guide: Non-NGC Environments

This guide covers how to run Khala on standard GPU cloud platforms (e.g., AutoDL, Vast.ai) that do not support custom Docker images or NGC containers.

> **Note**: The official recommended environment is the NGC `nvcr.io/nvidia/pytorch:25.02-py3` image. Running outside of NGC will result in significantly slower inference speed due to the absence of Flash Attention and CUDA Graph support. Expect approximately 1 tok/s instead of 10+ tok/s.

---

## Environment

Tested on:

- **Platform**: AutoDL
- **GPU**: NVIDIA RTX 4090D (24GB VRAM)
- **Base Image**: PyTorch 2.8.0 + Python 3.12 + CUDA 12.8 (Ubuntu 22.04)

---

## Step 1: Install System Dependencies

```bash
apt-get update && apt-get install -y ffmpeg curl xz-utils tmux
```

## Step 2: Install Node.js

```bash
mkdir -p /usr/local/lib/nodejs
curl -fsSLO https://nodejs.org/dist/v24.15.0/node-v24.15.0-linux-x64.tar.xz
tar -xJf node-v24.15.0-linux-x64.tar.xz -C /usr/local/lib/nodejs
ln -sf /usr/local/lib/nodejs/node-v24.15.0-linux-x64/bin/node /usr/local/bin/node
ln -sf /usr/local/lib/nodejs/node-v24.15.0-linux-x64/bin/npm /usr/local/bin/npm
ln -sf /usr/local/lib/nodejs/node-v24.15.0-linux-x64/bin/npx /usr/local/bin/npx
rm -f node-v24.15.0-linux-x64.tar.xz
```

## Step 3: Clone the Repository

```bash
git clone https://github.com/Khala-Music-AI/Khala.git /root/Khala
cd /root/Khala
```

## Step 4: Install Python Dependencies

```bash
pip install -r requirements.txt
pip install pybind11
pip install --no-build-isolation "transformer-engine[pytorch]==2.1.0"
```

> `pybind11` is required to compile Megatron's C++ dataset helpers.  
> `transformer-engine` must match your PyTorch version. For PyTorch 2.8, use version `2.1.0`.

## Step 5: Install Frontend Dependencies

```bash
cd /root/Khala/frontend
npm install
```

## Step 6: Download Model Checkpoints

The model is hosted on HuggingFace. Use the mirror for faster downloads in China:

```bash
pip install huggingface_hub
# Use tmux to prevent interruption on SSH disconnect
tmux new -s download
HF_ENDPOINT=https://hf-mirror.com hf download liujiafeng/Khala-MusicGeneration-v1.0 --local-dir /path/to/checkpoints
```

> `hf download` supports resume on reconnect. If interrupted, re-run the same command.  
> The total size is approximately 19GB. Make sure you have enough disk space.

Then create a symlink so the code can find the checkpoints:

```bash
ln -s /path/to/checkpoints /root/Khala/checkpoints
```

---

## Step 7: Apply Required Code Patches

Three modifications are needed to work around incompatibilities between the checkpoint's saved configuration and the non-NGC environment.

### 7.1 Disable `gradient_accumulation_fusion` (run_backend.sh)

The checkpoint was saved with `gradient_accumulation_fusion=True`, which requires the APEX library (`fused_weight_gradient_mlp_cuda`). This is not available outside NGC.

In `backend/run_backend.sh`, add `--no-gradient-accumulation-fusion` to `MEGATRON_ARGS`:

```bash
MEGATRON_ARGS=(
    --tensor-model-parallel-size 1
    --pipeline-model-parallel-size 1
    --tokenizer-type NullTokenizer
    --norm-epsilon 1e-6
    --num-tokens-to-generate 23552
    --inference-max-seq-length 25600
    --stream
    --no-gradient-accumulation-fusion   # <-- add this line
    --bf16
)
```

### 7.2 Disable CUDA Graph (backend_worker.py)

The checkpoint was also saved with `enable_cuda_graph=True`. When loaded via `--use-checkpoint-args`, this triggers a CUDA graph warmup that crashes in non-NGC environments.

In `backend/backend_worker.py`, find the `load_backbone` function and add the following line before the cuda graph check (around line 463):

```python
args.enable_cuda_graph = False  # Force disable — not supported outside NGC
if getattr(args, "enable_cuda_graph", False):
    print("[Worker] Running CUDA graph warmup for backbone.")
    ...
```

### 7.3 Increase API Timeouts (backend_api.py)

Without Flash Attention and CUDA Graph, inference runs at approximately 1 tok/s. Generating 3 minutes of audio requires ~9,800 tokens, which takes around 2–3 hours. The default timeouts are too short for this.

In `backend/backend_api.py`, update the following values:

```python
# Line ~360: job cleanup TTL (increase from 30 min to 4 hours)
JOB_TTL_SECONDS = 14400

# Line ~455: HTTP request timeout (increase from 10 min to 5 hours)
async def _dispatch_to_worker(
    ...
    timeout: float = 18000.0,   # was 600.0
    ...
):
```

---

## Step 8: Start the Services

```bash
# Start backend
tmux new -s backend
cd /root/Khala/backend
bash run_backend.sh

# Start frontend (new terminal)
tmux new -s frontend
cd /root/Khala/frontend
npm run dev
```

Default ports: frontend `30869`, backend API `8889`.

---

## Performance Notes

| Environment | Inference Speed | Time for 3-min audio |
|-------------|----------------|----------------------|
| NGC 25.02 (official) | ~10+ tok/s | ~15 minutes |
| Non-NGC (this guide) | ~1 tok/s | ~2–3 hours |

The speed difference is due to the absence of:
- **Flash Attention** — faster attention computation
- **CUDA Graph** — eliminates Python overhead per token

These features are bundled in the NGC image and are non-trivial to install separately.
