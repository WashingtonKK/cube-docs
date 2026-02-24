---
id: private-model-upload
title: Private Model Upload
sidebar_position: 3
---

## Uploading Private Models to Cube AI

This guide explains how to upload and deploy private or custom models into a Cube AI Confidential VM (CVM). Private models are models that are not available in public registries (Ollama library, HuggingFace) — for example, fine-tuned models, proprietary weights, or models with restricted access.

---

## Ollama Backend

### Upload Model Files to a Running CVM

#### 1. Package Model Files

Prepare your model weights and any associated files into an archive:

```bash
tar -czvf my-model.tar.gz /path/to/model/files
```

#### 2. Transfer to the CVM

Copy the archive into the CVM via SCP using the forwarded SSH port:

```bash
# Buildroot CVM
scp -P 6190 my-model.tar.gz root@localhost:/var/lib/ollama/

# Ubuntu cloud-init CVM
scp -P 6190 my-model.tar.gz ultraviolet@localhost:/var/lib/ollama/
```

#### 3. Extract and Register the Model

SSH into the CVM and create an Ollama model from the uploaded files using a [Modelfile](https://github.com/ollama/ollama/blob/main/docs/modelfile.md):

```bash
ssh -p 6190 root@localhost

cd /var/lib/ollama
tar -xzvf my-model.tar.gz
```

Create a Modelfile that references the uploaded weights:

```bash
cat > /tmp/Modelfile << 'EOF'
FROM /var/lib/ollama/my-model/weights.gguf
PARAMETER temperature 0.7
PARAMETER top_p 0.9
SYSTEM "You are a helpful assistant."
EOF

ollama create my-custom-model -f /tmp/Modelfile
```

#### 4. Verify the Model

```bash
ollama list
```

Test inference:

```bash
curl http://localhost:7001/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-custom-model",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

### Embed a Private Model in a Buildroot HAL Image

To include a private model directly in the HAL image at build time, use the Buildroot filesystem overlay:

#### 1. Place Model Files in the Overlay

Create a directory for the model in the overlay structure:

```bash
mkdir -p cube/hal/buildroot/linux/board/cube/overlay/var/lib/ollama/custom-models/
cp /path/to/weights.gguf cube/hal/buildroot/linux/board/cube/overlay/var/lib/ollama/custom-models/
```

#### 2. Add a Modelfile to the Overlay

```bash
mkdir -p cube/hal/buildroot/linux/board/cube/overlay/etc/cube/modelfiles/
cat > cube/hal/buildroot/linux/board/cube/overlay/etc/cube/modelfiles/my-model.Modelfile << 'EOF'
FROM /var/lib/ollama/custom-models/weights.gguf
PARAMETER temperature 0.7
SYSTEM "You are a domain-specific assistant."
EOF
```

#### 3. Register the Model on First Boot

Add a startup script in the overlay that creates the Ollama model after the service starts:

```bash
mkdir -p cube/hal/buildroot/linux/board/cube/overlay/usr/libexec/ollama/
cat > cube/hal/buildroot/linux/board/cube/overlay/usr/libexec/ollama/register-custom-models.sh << 'SCRIPT'
#!/bin/sh
# Wait for Ollama to be ready
for i in $(seq 1 30); do
  if curl -s http://localhost:11434/api/version > /dev/null 2>&1; then
    break
  fi
  sleep 2
done

# Register custom models from Modelfiles
for mf in /etc/cube/modelfiles/*.Modelfile; do
  [ -f "$mf" ] || continue
  name=$(basename "$mf" .Modelfile)
  ollama create "$name" -f "$mf"
done
SCRIPT
chmod +x cube/hal/buildroot/linux/board/cube/overlay/usr/libexec/ollama/register-custom-models.sh
```

#### 4. Build the Image

```bash
cd buildroot
make -j$(nproc)
```

The model weights are embedded in the rootfs and registered automatically on first boot.

### Embed a Private Model via Cloud-Init

To deploy a private model in an Ubuntu cloud-init CVM, modify the `user-data` section in `hal/ubuntu/qemu.sh`:

#### 1. Pre-Stage Model Files on the Host

Place model files in a directory accessible to the QEMU VM. The simplest approach is to transfer them after boot via the `runcmd` section.

#### 2. Add a Modelfile and Registration to Cloud-Init

Add the Modelfile and a registration command to the `write_files` and `runcmd` sections:

```yaml
write_files:
  - path: /etc/cube/modelfiles/my-model.Modelfile
    content: |
      FROM /var/lib/ollama/custom-models/weights.gguf
      PARAMETER temperature 0.7
      SYSTEM "You are a domain-specific assistant."
    permissions: '0644'

runcmd:
  # ... (existing commands)
  # After ollama is installed and running, register the custom model
  - |
    for i in $(seq 1 60); do
      if curl -s http://localhost:11434/api/version > /dev/null 2>&1; then
        break
      fi
      sleep 2
    done
    ollama create my-model -f /etc/cube/modelfiles/my-model.Modelfile
```

If the model weights need to be downloaded from a private source during provisioning, add a download step before registration:

```yaml
runcmd:
  # Download private model weights (e.g., from a private S3 bucket or internal server)
  - mkdir -p /var/lib/ollama/custom-models
  - curl -o /var/lib/ollama/custom-models/weights.gguf https://internal-server/models/weights.gguf
  # Then register
  - ollama create my-model -f /etc/cube/modelfiles/my-model.Modelfile
```

---

## vLLM Backend

### Upload Custom Model Files to a Running CVM

#### 1. Transfer Model Directory

vLLM expects a HuggingFace-format model directory. Transfer the entire directory:

```bash
scp -r -P 6190 /path/to/my-hf-model/ root@localhost:/var/lib/vllm/models/
```

#### 2. Update vLLM Configuration

SSH into the CVM and update the vLLM environment to point to the uploaded model:

```bash
ssh -p 6190 root@localhost

# Edit the vLLM config
sed -i 's|^VLLM_MODEL=.*|VLLM_MODEL=/var/lib/vllm/models/my-hf-model|' /etc/vllm/vllm.env

# Restart vLLM
systemctl restart vllm
# or for SysV init:
/etc/init.d/S96vllm restart
```

#### 3. Verify

```bash
curl http://localhost:8000/v1/models
```

### Embed a Custom Model in a Buildroot HAL Image

Use the `BR2_PACKAGE_VLLM_CUSTOM_MODEL_PATH` option to embed model files at build time.

#### 1. Configure the Model Path

In `menuconfig`, navigate to **Target packages → Cube packages → vllm** and set:

- **Custom model path** — Absolute path to the model directory on your build machine

Or set it in the defconfig:

```bash
BR2_PACKAGE_VLLM_CUSTOM_MODEL_PATH="/path/to/my-hf-model"
```

#### 2. Build

```bash
make -j$(nproc)
```

The build system copies the model files into `/var/lib/vllm/models/` in the image and configures vLLM to use the local path automatically.

### Embed a Custom Model via Cloud-Init

Add a model download or transfer step to the `runcmd` section in `hal/ubuntu/qemu.sh`:

```yaml
runcmd:
  # Install vLLM
  - pip install vllm
  # Download private model
  - mkdir -p /var/lib/vllm/models
  - |
    # Option A: Download from a private registry (requires HF token for gated models)
    HF_TOKEN="your-token-here"
    huggingface-cli download my-org/my-private-model \
      --local-dir /var/lib/vllm/models/my-private-model \
      --token "$HF_TOKEN"
  # Configure and start vLLM
  - |
    cat > /etc/vllm/vllm.env << 'ENVEOF'
    VLLM_MODEL=/var/lib/vllm/models/my-private-model
    VLLM_GPU_MEMORY_UTILIZATION=0.85
    VLLM_MAX_MODEL_LEN=2048
    ENVEOF
  - systemctl restart vllm
```

---

## Verifying Model Availability Through the Proxy

After deploying a custom model, verify it is accessible end-to-end through the Cube Agent:

```bash
# List available models
curl http://localhost:6193/v1/models

# Test chat completions
curl http://localhost:6193/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-custom-model",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

Port `6193` is the default host-side forwarded port for the Cube Agent (maps to port `7001` inside the CVM).
