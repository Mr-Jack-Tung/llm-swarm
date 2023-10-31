# TGI-swarm

This repo is intended for generating massive texts leverage [huggingface/text-generation-inference](https://github.com/huggingface/text-generation-inference)

Prerequisites:
* A slurm cluster


## Hello world

```bash
mkdir -p slurm/logs
sbatch tgi.slurm
```

This will generate log filesa in `slurm/logs` and also `hosts.txt` with the list of nodes used for the job.

```bash
python python generate_hh_simple.py
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