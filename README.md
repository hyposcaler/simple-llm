This is a simple docker compose set up to use vLLM to run Qwen/Qwen2.5-14B-Instruct-AWQ (4bit quant version of Qwen 2.5 14B instruct model), and uses open-webui for a web based chat interface. The OpenAI compatible API is also exposed via port 8000 on localhost.

This setup is useful to experiment with local LLMs, vLLM, and open-webui. 

The model itself fits within 8-9G of VRAM and runs easily on a 5090. The more VRAM above the 9GB you have the larger the context that will be available.  On a 5090 vLLM auto configures for 2 concurrent connections with ~32K of context consuming an additional 18G for a total of ~28G

Smaller GPUs technically can work but you need to reduce the size of the context and/or use a smaller model, i.e. rather than Qwen/Qwen2.5-14B-Instruct-AWQ you could try with Qwen/Qwen2.5-7B-Instruct-AWQ.

The docker-compose setup uses a docker volume to store both the model and data for the web-ui. This saves time on startup the second time around since it doesn't have to download the model.

> Note alot of what is done here can just as easily be done without docker and just using python venvs, I just personally find the docker route easier.

Prerequisites, in the default configuration, you will need docker installed on the machine and a GPU with at least 28GB of VRAM (e.g. RTX 5090 or better).  You will also need NVIDIA's container toolkit installed

See the [docker website](https://docs.docker.com/engine/install/) for install instructions for a variety of platforms 

see [NVIDIA container toolkit docs](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) for how to install the toolkit on assorted platforms.

This repo has primarily been tested using debian, but arguably should work anywhere you can install docker and the NVIDIA container toolkit.

to start, from the repo run
```
docker compose up -d
```

It will take a minute or so to boot, longer on the first time since it needs to download the models, you can check the logs to monitor progress with
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

The OpenAI API compatible endpoint provided by vLLM is also available via port 8000, for a simple test use curl

```
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-14B-Instruct-AWQ",
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