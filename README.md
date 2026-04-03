# Simple LLM
Simple docker compose setups for running local LLMs with a web-based chat interface (open-webui). Each subdirectory contains an independent docker compose environment for a different inference backend.

> Note a lot of what is done here can just as easily be done without docker and just using python venvs, I just personally find the docker route easier.

This repo is useful for experimenting with local LLMs, different inference backends, and open-webui.

## Backends

| Directory | Backend | Status |
|-----------|---------|--------|
| [vllm/](vllm/) | vLLM | Available |
| [ollama/](ollama/) | Ollama | Available |

## Prerequisites
You will need docker installed on the machine. See the [docker website](https://docs.docker.com/engine/install/) for install instructions for a variety of platforms.

This repo has primarily been tested using debian, but arguably should work anywhere you can install docker.

Additional prerequisites vary by backend -- see the README in each subdirectory for details.

---

## vLLM

Uses vLLM to run Qwen/Qwen3-14B-FP8 (FP8 quant version of Qwen 3 14B model with thinking/non-thinking mode), with open-webui for a web based chat interface. Includes Prometheus and Grafana for monitoring vLLM metrics. The OpenAI compatible API is also exposed via port 8000 on localhost.

In the default config (running Qwen/Qwen3-14B-FP8) the model itself fits within ~15-17GB of VRAM and runs easily on a 5090. The more VRAM above 15GB you have the larger the context that will be available. On a 5090 vLLM has ~16GB remaining for KV cache.  vLLM in this config seems to autoconfigure for ~2 concurrent  connections with ~40k tokens worth of context each. 

Smaller GPUs technically can work but you need to reduce the size of the context and/or use a smaller model, i.e. rather than Qwen/Qwen3-14B-FP8 you could try with Qwen/Qwen3-8B-FP8.

The model cache is stored in a local `./models` directory (bind mount), so it survives `docker compose down -v` and doesn't need to be re-downloaded. Open-webui data (chat history, settings) uses a Docker volume (`open-webui-data`) which will be removed by `docker compose down -v`. Prometheus and Grafana have no persistent storage and start fresh each time.

### Prerequisites 
In the default configuration, you will need a GPU with at least 20GB of VRAM (e.g. RTX 3090 or 4090).  Technically you can fit Qwen/Qwen3-14B-FP8 on a 16GB GPU, but will have little to no space left over for context.

You will also need NVIDIA's container toolkit installed. See [NVIDIA container toolkit docs](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) for how to install the toolkit on assorted platforms.

### Startup
To start, from the `vllm/` directory run
```
docker compose up -d
```

It will take 2-3 minutes or so to launch, longer on the first go since it needs to download the model (~15G for Qwen/Qwen3-14B-FP8 ), you can check the logs to monitor progress with:
```
docker compose logs -f
```

if you just want to see logs from an individual service (i.e vllm, open-webui, etc) just append the service name to the command.
```
docker compose logs -f vllm
```

You can monitor the vllm logs to see progress, when you see the following, the model is ready to handle queries.
```
vllm-1  | (APIServer pid=1) INFO:     Started server process [1]
vllm-1  | (APIServer pid=1) INFO:     Waiting for application startup.
vllm-1  | (APIServer pid=1) INFO:     Application startup complete.
```

### Use
You can access the web-ui interface in your browser via [http://localhost:3000/](http://localhost:3000/)

Prometheus is available at [http://localhost:9090/](http://localhost:9090/)

Grafana is available at [http://localhost:3001/](http://localhost:3001/). 

Grafana is also preconfigured with prometheus as a datasource, and pre-loaded with a modified version (mostly korean->english translations of panel descriptions) of minesoap's [vllm-monitoring-v2 dashboard](https://grafana.com/grafana/dashboards/24756-vllm-monitoring-v2/). 

>Security NOTE! No login for grafana, prom, nor the vLLM API is required.

The OpenAI API compatible endpoint provided by vLLM is also available via port 8000, for a simple test use curl

```
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-14B-FP8",
    "messages": [{"role": "user", "content": "how many r'\''s are in strawberry"}],
    "temperature": 0.7,
    "max_tokens": 200
  }'
```

On first start, a directory named `models` will be created, this is made available to the vllm container via a bind mount, it is used to store the model when downloaded.  A docker volume is used for the web-ui config.  There is no persistent storage configured for prom or grafana.

### Teardown
To tear down and leave volumes in place
```
docker compose down
```

To tear down and remove volumes too (this will not delete the models directory)
```
docker compose down -v
```

---

## Ollama

Uses Ollama to run gemma4:31b (Google's Gemma 4 31B multimodal model), with open-webui for a web based chat interface. This is a simpler setup compared to vLLM -- just Ollama and open-webui, no monitoring stack.

The model is automatically pulled on first startup. The model cache is stored in a local `./models` directory (bind mount), so it survives `docker compose down -v` and doesn't need to be re-downloaded. Open-webui data (chat history, settings) uses a Docker volume (`open-webui-data`) which will be removed by `docker compose down -v`.

The Ollama API is exposed via port 11434 on localhost.

### Prerequisites
You will need a GPU with NVIDIA's container toolkit installed. See [NVIDIA container toolkit docs](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) for how to install the toolkit on assorted platforms.

### Startup
To start, from the `ollama/` directory run
```
docker compose up -d
```

On first start, Ollama will pull the gemma4:31b model which may take several minutes depending on your connection. You can monitor progress with:
```
docker compose logs -f ollama
```

### Use
You can access the web-ui interface in your browser via [http://localhost:3000/](http://localhost:3000/)

The Ollama API is available at [http://localhost:11434/](http://localhost:11434/)

To test the API directly with curl:
```
curl http://localhost:11434/api/chat \
  -d '{
    "model": "gemma4:31b",
    "messages": [{"role": "user", "content": "how many r'\''s are in strawberry"}],
    "stream": false
  }'
```

### Teardown
To tear down and leave volumes in place
```
docker compose down
```

To tear down and remove volumes too (this will not delete the models directory)
```
docker compose down -v
```

---

## Resources
- [vLLM repo](https://github.com/vllm-project/vllm)
- [vLLM Docs](https://docs.vllm.ai/en/latest/)
- [Ollama](https://ollama.com/)
- [open-webui repo](https://github.com/open-webui/open-webui)
- [open-webui docs](https://docs.openwebui.com/)
- [Qwen3-14B-FP8](https://huggingface.co/Qwen/Qwen3-14B-FP8)
- [Qwen](https://huggingface.co/Qwen)