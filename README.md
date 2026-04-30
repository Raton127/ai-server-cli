# ai-server-cli

---

#  vLLM Multi-Model Inference Server

This repository provides a production-ready setup for running multiple LLMs using **vLLM OpenAI-compatible API server**.

Supported models:

* Qwen 3.6 35B
* Llama 3.1 8B
* Mistral 7B
* DeepSeek R1 Distill

---

##  Requirements

* Ubuntu / AlmaLinux
* NVIDIA GPU (CUDA supported)
* Python virtual environment (`/opt/vllm311`)
* Installed:

  * vLLM
  * CUDA drivers
  * NVIDIA Toolkit

---

##  Model Deployment

###  Qwen (Port 8000)

```bash
/opt/vllm311/bin/vllm serve Qwen/Qwen3.6-35B-A3B \
  --host 0.0.0.0 \
  --port 8000 \
  --tensor-parallel-size 1 \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.84 \
  --enable-prefix-caching \
  --max-num-batched-tokens 12288 \
  --max-num-seqs 16 \
  --stream-interval 8 \
  --reasoning-parser qwen3 \
  --enable-auto-tool-choice \
  --tool-call-parser qwen3_coder \
  --language-model-only \
  --enforce-eager \
  --moe-backend triton \
  --no-enable-flashinfer-autotune
```

---

###  Llama (Port 8001)

```bash
/opt/vllm311/bin/vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --host 0.0.0.0 \
  --port 8001 \
  --tensor-parallel-size 1 \
  --max-model-len 16384 \
  --gpu-memory-utilization 0.75 \
  --enable-prefix-caching \
  --max-num-batched-tokens 8192 \
  --max-num-seqs 16
```

---

###  Mistral (Port 8002)

```bash
/opt/vllm311/bin/vllm serve mistralai/Mistral-7B-Instruct-v0.3 \
  --host 0.0.0.0 \
  --port 8002 \
  --tensor-parallel-size 1 \
  --max-model-len 16384 \
  --gpu-memory-utilization 0.75 \
  --enable-prefix-caching \
  --max-num-batched-tokens 8192 \
  --max-num-seqs 16
```

---

###  DeepSeek (Port 8003)

```bash
/opt/vllm311/bin/vllm serve deepseek-ai/DeepSeek-R1-Distill-Qwen-14B \
  --host 0.0.0.0 \
  --port 8003 \
  --tensor-parallel-size 1 \
  --max-model-len 16384 \
  --gpu-memory-utilization 0.8 \
  --enable-prefix-caching \
  --max-num-batched-tokens 8192 \
  --max-num-seqs 12
```

---

##  API Usage

###  List Models

```bash
curl http://127.0.0.1:8000/v1/models
```

---

###  Chat Completion

```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model":"Qwen/Qwen3.6-35B-A3B",
    "messages":[{"role":"user","content":"hello"}],
    "max_tokens":128,
    "temperature":0.7
  }'
```

---

###  Text Completion

```bash
curl http://127.0.0.1:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model":"Qwen/Qwen3.6-35B-A3B",
    "prompt":"Write one short sentence about Dhaka.",
    "max_tokens":64,
    "temperature":0.7
  }'
```

---

###  Health Check

```bash
curl http://127.0.0.1:8000/health
```

---

###  Streaming Response

```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model":"Qwen/Qwen3.6-35B-A3B",
    "messages":[{"role":"user","content":"Tell me a joke"}],
    "max_tokens":128,
    "stream":true
  }'
```

---

##  Systemd Service Setup

Create service file:

```bash
/etc/systemd/system/vllm-qwen.service
```

```ini
[Unit]
Description=vLLM OpenAI API
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
Group=root
Environment="HF_HOME=/var/cache/huggingface"
Environment="HUGGING_FACE_HUB_CACHE=/var/cache/huggingface"
Environment="HF_HUB_DISABLE_XET=1"
Environment="PATH=/opt/vllm311/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin"

ExecStart=/opt/vllm311/bin/vllm serve Qwen/Qwen3.6-35B-A3B \
  --host 0.0.0.0 \
  --port 8000 \
  --tensor-parallel-size 1 \
  --max-model-len 16384 \
  --gpu-memory-utilization 0.8 \
  --enable-prefix-caching \
  --max-num-batched-tokens 8192 \
  --max-num-seqs 16

Restart=always
RestartSec=10
TimeoutStopSec=120

[Install]
WantedBy=multi-user.target
```

---

### Service Commands

```bash
systemctl daemon-reload
systemctl enable vllm-qwen
systemctl restart vllm-qwen
systemctl status vllm-qwen
journalctl -u vllm-qwen -f
```

---

##  Benchmarking

###  Latency Test

```bash
time curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model":"Qwen/Qwen3.6-35B-A3B",
    "messages":[{"role":"user","content":"Write one short sentence about Dhaka."}],
    "max_tokens":64,
    "temperature":0
  }' > /dev/null
```

---

###  Concurrency Test

```bash
seq 1 10 | xargs -I{} -P 5 curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model":"Qwen/Qwen3.6-35B-A3B",
    "messages":[{"role":"user","content":"Say ok"}],
    "max_tokens":16,
    "temperature":0
  }' > /dev/null
```

---

###  Loop Benchmark

```bash
for i in {1..20}; do
  curl -s http://127.0.0.1:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
      "model":"Qwen/Qwen3.6-35B-A3B",
      "messages":[{"role":"user","content":"Say ok"}],
      "max_tokens":16,
      "temperature":0
    }' > /dev/null
done
```

---

###  GPU Monitoring

```bash
watch -n 1 nvidia-smi
```

---

##  Help

```bash
/opt/vllm311/bin/vllm serve --help=all
/opt/vllm311/bin/vllm serve --help=ModelConfig
/opt/vllm311/bin/vllm serve --help=SchedulerConfig
/opt/vllm311/bin/vllm serve --help=CacheConfig
/opt/vllm311/bin/vllm serve --help=KernelConfig
```

---

##  Notes

* Each model runs on a separate port
* OpenAI-compatible API format
* Optimized for GPU inference
* Supports streaming + batching

---
