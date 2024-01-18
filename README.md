# inference-swarm

This repo is intended for generating massive texts leverage [huggingface/text-generation-inference](https://github.com/huggingface/text-generation-inference)

Prerequisites:
* A slurm cluster
* docker


## Install and prepare

```bash
pip install -e .
# or pip install -r ./examples/hh/requirements.txt

mkdir -p slurm/logs
# you can customize the following docker image cache locations and change them in `templates/tgi_h100.template.slurm` and `templates/vllm_h100.template.slurm`
mkdir -p .cache/vllm
mkdir -p .cache/tgi
```

## Hello world

```bash
export HF_TOKEN=<YOUR_HF_TOKEN>
python examples/hello_world.py
python examples/hello_world_vllm.py
```

```python
import asyncio
import pandas as pd
from inference_swarm import InferenceSwarm, InferenceSwarmConfig
from huggingface_hub import AsyncInferenceClient
from transformers import AutoTokenizer
from tqdm.asyncio import tqdm_asyncio


tasks = [
    "What is the capital of France?",
    "Who wrote Romeo and Juliet?",
    "What is the formula for water?"
]
with InferenceSwarm(
    InferenceSwarmConfig(
        instances=2,
        inference_engine="tgi",
        slurm_template_path="templates/tgi_h100.template.slurm",
        load_balancer_template_path="templates/nginx.template.conf",
    )
) as inference_swarm:
    client = AsyncInferenceClient(model=inference_swarm.endpoint)
    tokenizer = AutoTokenizer.from_pretrained("mistralai/Mistral-7B-Instruct-v0.1")
    tokenizer.add_special_tokens({"sep_token": "", "cls_token": "", "mask_token": "", "pad_token": "[PAD]"})

    async def process_text(task):
        prompt = tokenizer.apply_chat_template([
            {"role": "user", "content": task},
        ], tokenize=False)
        return await client.text_generation(
            prompt=prompt,
            max_new_tokens=200,
        )

    async def main():
        results = await tqdm_asyncio.gather(*(process_text(task) for task in tasks))
        df = pd.DataFrame({'Task': tasks, 'Completion': results})
        print(df)
    asyncio.run(main())
```
```
(.venv) costa@login-node-1:/fsx/costa/tgi-swarm$ python examples/hello_world.py
None of PyTorch, TensorFlow >= 2.0, or Flax have been found. Models won't be available and only tokenizers, configuration and file/data utilities can be used.
running sbatch --parsable slurm/tgi_1705591874_tgi.slurm
running sbatch --parsable slurm/tgi_1705591874_tgi.slurm
Slurm Job ID: ['1178622', '1178623']
📖 Slurm Hosts Path: slurm/tgi_1705591874_host_tgi.txt
✅ Done! Waiting for 1178622 to be created                                                                 
✅ Done! Waiting for 1178623 to be created                                                                 
✅ Done! Waiting for slurm/tgi_1705591874_host_tgi.txt to be created                                       
obtained endpoints ['http://26.0.161.138:46777', 'http://26.0.167.175:44806']
⣽ Waiting for http://26.0.161.138:46777 to be reachable
Connected to http://26.0.161.138:46777
✅ Done! Waiting for http://26.0.161.138:46777 to be reachable                                             
⣯ Waiting for http://26.0.167.175:44806 to be reachable
Connected to http://26.0.167.175:44806
✅ Done! Waiting for http://26.0.167.175:44806 to be reachable                                             
Endpoints running properly: ['http://26.0.161.138:46777', 'http://26.0.167.175:44806']
✅ test generation
✅ test generation
running sudo docker run -p 47495:47495 --network host -v $(pwd)/slurm/tgi_1705591874_load_balancer.conf:/etc/nginx/nginx.conf nginx
b'WARNING: Published ports are discarded when using host network mode'
b'/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration'
🔥 endpoint ready http://localhost:47495
haha
100%|████████████████████████████████████████████████████████████████████████| 3/3 [00:01<00:00,  2.44it/s]
                             Task                                         Completion
0  What is the capital of France?                    The capital of France is Paris.
1     Who wrote Romeo and Juliet?   Romeo and Juliet was written by William Shake...
2  What is the formula for water?   The chemical formula for water is H2O. It con...
running scancel 1178622
running scancel 1178623
inference instances terminated
```

It does a couple of things:


- 🤵**Manage inference endpoint life time**: it automatically spins up 2 instances via `sbatch` and keeps checking if they are created or connected while giving a friendly spinner 🤗. once the instances are reachable, `inference_swarm` connects to them and perform the generation job. Once the jobs are finished, `inference_swarm` auto-terminates the inference endpoints, so there is no idling inference endpoints wasting up GPU researches.
- 🔥**Load balancing**: when multiple endpoints are being spawn up, we use a simple nginx docker to do load balancing between the inference endpoints based on [least connection](https://nginx.org/en/docs/http/load_balancing.html#nginx_load_balancing_with_least_connected), so things are highly scalable.

`inference_swarm` will create a slurm file in `./slurm` based on the default configuration (` --slurm_template_path=tgi_template.slurm`) and logs in `./slurm/logs` if you are interested to inspect.


## Development mode

It is possible to run the `inference_swarm` to spin up instances until the user manually stops them. This is useful for development and debugging.

```bash
# run tgi
python -m inference_swarm --instances=1
# run vllm
python -m inference_swarm --instances=1 --slurm_template_path templates/vllm_h100.template.slurm --inference_engine=vllm
```

Running commands above will give you outputs like below. 

```
(.venv) costa@login-node-1:/fsx/costa/inference-swarm$ python -m inference_swarm --slurm_template_path templates
/vllm_h100.template.slurm --inference_engine=vllm
None of PyTorch, TensorFlow >= 2.0, or Flax have been found. Models won't be available and only tokenizers, configuration and file/data utilities can be used.
running sbatch --parsable slurm/vllm_1705590449_vllm.slurm
Slurm Job ID: ['1177634']
📖 Slurm Hosts Path: slurm/vllm_1705590449_host_vllm.txt
✅ Done! Waiting for 1177634 to be created                                                          
✅ Done! Waiting for slurm/vllm_1705590449_host_vllm.txt to be created                              
obtained endpoints ['http://26.0.161.138:11977']
⣷ Waiting for http://26.0.161.138:11977 to be reachable
Connected to http://26.0.161.138:11977
✅ Done! Waiting for http://26.0.161.138:11977 to be reachable                                      
Endpoints running properly: ['http://26.0.161.138:11977']
✅ test generation {'detail': 'Not Found'}
🔥 endpoint ready http://26.0.161.138:11977
Press Enter to EXIT...
```

You can use the endpoints to test the inference engine. For example, you can pass in `--debug_endpoint=http://26.0.161.138:11977` to tell `inference_swarm` not to spin up instances and use the endpoint directly.

```bash
python examples/benchmark.py --debug_endpoint=http://26.0.161.138:11977 --inference_engine=vllm
```

![](static/debug_endpoint.png)


When you are done, you can press `Enter` to stop the instances.




```bash
python ./examples/hh/generate_hh_simple.py
```

If your `slurm` cluster uses Pyxis and Enroot for deploying Docker containers (e.g our H100 cluster), run this instead:
* TGI:
```bash
# deploy TGI
sbatch tgi_h100.slurm
# get hostname, this uses the latest created log path
# you may need to run this multiple times until the TGI instance is up
bash get_hostname.sh
# upon success, you should see something like this
(inference-swarm-py3.10) costa@login-node-1:/fsx/costa/inference-swarm$ bash get_hostname.sh
Using tgi
PWD: /fsx/costa/inference-swarm
Latest created log file is slurm/logs/inference-swarm_513810.out
Port not found in log file.
Hostname: ip-26-0-164-236
Saving address http://ip-26-0-164-236:59085 in /fsx/costa/inference-swarm/hosts.txt
{"generated_text":"\n\nLife is a characteristic that distinguishes physical"}
The TGI endpoint works 🎉!
```

* vLLM:
```bash
# deploy vLLM
sbatch vllm_h100.slurm
# get hostname, this uses the latest created log path
# you may need to run this multiple times until the vLLM instance is up
bash get_hostname.sh vllm
# upon success, you should see something like this
(inference-swarm-py3.10) costa@login-node-1:/fsx/costa/inference-swarm$ 
Using vllm
PWD: /fsx/costa/inference-swarm
Latest created log file is slurm/logs_vllm/vllm_549609.out
Job 549609 running on ip-26-0-161-221
Hostname: ip-26-0-161-221
Saving address http://ip-26-0-161-221:8000 in /fsx/costa/inference-swarm/host_vllm.txt
{"text":["What is Life?\n\nLife is a characteristic that distinguishes physical entities that have biological processes and"]}
The vLLM endpoint works 🎉!
```

This will generate log files in `./slurm/logs` and also `./hosts.txt` (`./slurm/vllm_logs` and also `./hosts_vllm.txt` for vLLM) with the list of nodes used for the job.

```bash
# tgi
python ./examples/hh/generate_hh_simple.py --max_samples 50 --manage_tgi_instances  --instances 1
# vllm
python ./examples/hh/generate_hh_simple.py --max_samples 50 --use_vllm --output_folder output/hh_simple_vllm --manage_tgi_instances  --instances 1
```
```
Loaded 1 endpoints: http://26.0.149.1:45920
Prompt formatting is ON
Preparing data
Starting workers
Loading dataset
Generating...
24it [00:30,  5.02s/it]

## after a minute or so
64it [02:24,  2.26s/it]
Saving chunk 1/None
Processing complete.
```

Then you should be able to see some sample outputs in `output/hh_simple`


## Generating data for the entire harmless dataset

```
python examples/hh/generate_hh.py --instances 8 --m anage_tgi_instances --max_samples=-1
python merge_data.py --output_folder=output/hh
```

# Installing TGI from scratch

```
conda install pytorch pytorch-cuda=11.8 -c pytorch -c nvidia
cd server
pip install packaging ninja
make build-flash-attention
make build-flash-attention-v2
make build-vllm
```
