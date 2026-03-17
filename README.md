# Simple LLM

This is a simple docker compose setup for using vLLM to run Qwen/Qwen3-14B-FP8 (FP8 quant version of Qwen 3 14B model with thinking/non-thinking mode), and uses open-webui for a web based chat interface. It includes Prometheus and Grafana for monitoring vLLM metrics. The OpenAI compatible API is also exposed via port 8000 on localhost.

> Note a lot of what is done here can just as easily be done without docker and just using python venvs, I just personally find the docker route easier.

This setup is useful to experiment with local LLMs, vLLM, and open-webui. 

In the default config (running Qwen/Qwen3-14B-FP8) the model itself fits within ~15GB of VRAM and runs easily on a 5090. The more VRAM above 15GB you have the larger the context that will be available. On a 5090 vLLM has ~16GB remaining for KV cache.  vLLM in this config seems to autoconfigure for ~2 concurrent  connections with ~40k tokens worth of context each. 

Smaller GPUs technically can work but you need to reduce the size of the context and/or use a smaller model, i.e. rather than Qwen/Qwen3-14B-FP8 you could try with Qwen/Qwen3-8B-FP8.

The model cache is stored in a local `./models` directory (bind mount), so it survives `docker compose down -v` and doesn't need to be re-downloaded. Open-webui data (chat history, settings) uses a Docker volume (`open-webui-data`) which will be removed by `docker compose down -v`. Prometheus and Grafana have no persistent storage and start fresh each time.

Prerequisites, in the default configuration, you will need docker installed on the machine and a GPU with at least 28GB of VRAM (e.g. RTX 5090 or better).  You will also need NVIDIA's container toolkit installed

See the [docker website](https://docs.docker.com/engine/install/) for install instructions for a variety of platforms 

see [NVIDIA container toolkit docs](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) for how to install the toolkit on assorted platforms.

This repo has primarily been tested using debian, but arguably should work anywhere you can install docker and the NVIDIA container toolkit.

to start, from the repo run
```
docker compose up -d
```

It will take a 2 minutes or so to fire up, longer on the first time since it needs to download the model (~15G for Qwen/Qwen3-14B-FP8 ), you can check the logs to monitor progress with
```
docker compose logs -f
```

if you just want to see logs from an individual service (i.e vLLM or open-webui) just append the service name to the command.
```
docker compose logs -f vllm
```

The model will take a bit to load, you can monitor the vLLM logs to see progress, you want to look for 
```
vllm-1  | (APIServer pid=1) INFO:     Started server process [1]
vllm-1  | (APIServer pid=1) INFO:     Waiting for application startup.
vllm-1  | (APIServer pid=1) INFO:     Application startup complete.
```

Once the model is loaded you should be able to access the web-ui interface in your browser via [http://localhost:3000/](http://localhost:3000/)

Prometheus is available at [http://localhost:9090/](http://localhost:9090/)

Grafana is available at [http://localhost:3001/](http://localhost:3001/) and is a preconfigured with prometheus as a datasource, and pre-loaded with a modified version (mostly korean->english translations of panel descriptions) of minesoap's [vllm-monitoring-v2 dashboard](https://grafana.com/grafana/dashboards/24756-vllm-monitoring-v2/). 

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

to tear down and leave volumes in place
```
docker compose down
```

to tear down and remove volumes too
```
docker compose down -v
```