# FlexGen

FlexGen is a high-throughput generation engine for running large language models with limited GPU memory (e.g., a 16GB T4 GPU or a 24GB RTX3090 gaming card!).
This is a research project developed by
[HazyResearch@Stanford](https://hazyresearch.stanford.edu/),
[SkyComputing@UC Berkeley](https://sky.cs.berkeley.edu/),
[DS3Lab@ETH Zurich](https://ds3lab.inf.ethz.ch/),
[CRFM@Stanford](https://crfm.stanford.edu/),
and [TogetherCompute](https://www.together.xyz/).

<a href="https://hazyresearch.stanford.edu/"><img src="https://identity.stanford.edu/wp-content/uploads/sites/3/2020/06/wordmark-nospace-red.png" height="25"></a> &nbsp;&nbsp;&nbsp; <a href="https://sky.cs.berkeley.edu/"><img src="https://upload.wikimedia.org/wikipedia/commons/thumb/8/82/University_of_California%2C_Berkeley_logo.svg/1280px-University_of_California%2C_Berkeley_logo.svg.png" height="30"></a> &nbsp;&nbsp;&nbsp; <a href="https://ds3lab.inf.ethz.ch/"><img src="https://user-images.githubusercontent.com/1608867/220273382-c09669b3-42fd-47c2-b88c-7ed55cb43820.png" height="25"></a> &nbsp;&nbsp;&nbsp; <a href="https://www.together.xyz/"><img src="https://cdn.discordapp.com/attachments/1032853929098236016/1077448896680296539/B3E025DC-1567-423E-B006-168F94D173CA.png" height="40"></a>

Large language models (LLMs) are at the heart of applications like ChatGPT and Copilot, but the high computational and memory requirements of LLM inference traditionally make it feasible only with multiple high-end accelerators.
FlexGen aims to lower the resource requirements of LLM inference down to a single commodity GPU (e.g., T4, 3090) and allow flexible deployment for various hardware setups.

The key features of FlexGen include:  

⚡ **Lightining Fast Offloading**.  
Up to 100x faster than other offloading-based systems for running 175B models on a single GPU.  

📦 **Extreme Compression**.  
Compress both the parameters and attention cache of models, such as OPT-175B, down to 4 bits with negligible accuracy loss.

🚀 **Scalability**.  
Come with a distributed pipeline parallelism runtime to allow scaling if more GPUs are given.

| [**Read Paper**](docs/paper.pdf) | [**Join Discord**](https://discord.gg/JfphDTkBAh) |

## Content
- [Benchmark Results](#benchmark-results)
- [Install](#install)
- [Get Started with a Single GPU](#get-started-with-a-single-gpu)
- [Run Chatbot with OPT models on a Single GPU](#run-chatbot-with-opt-models-on-a-single-gpu )
- [Scaling to Distributed GPUs](#scaling-to-distributed-gpus)
- [Roadmap](#roadmap)

## Benchmark Results
### Generation Throughput (token/s)
| System | OPT-6.7B | OPT-30B | OPT-175B |
| ------ | -------- | ------- | -------- |
| Hugging Face Accelerate   | 25.12 | 0.62 | 0.01 |
| DeepSpeed ZeRO-Inference | 9.28  | 0.60 | 0.01 |
| Petals\*                 | -     | -    | 0.05 |
| FlexGen                  | 25.26 | 7.32 | 0.69 |
| FlexGen with Compression | **29.12** | **8.38** | **1.12** |

- Hardware: an NVIDIA T4 (16GB) instance on GCP with 208GB of DRAM and 1.5TB of SSD.  
- Workload: input sequence length = 512, output sequence length = 32. The batch size is tuned to a value that maximizes the generation throughput for each system.  
- Metric: generation throughput (token/s) = number of the generated tokens / (time for processing prompts + time for generation).  

How to [reproduce](benchmark/flexgen).

### Latency-throughput Trade-off
The figure below shows the latency and throughput trade-off of three offloading-based systems on OPT-175B (left) and OPT-30B (right).
FlexGen achieves a new Pareto-optimal frontier with a 100x higher maximum throughput for OPT-175B.
Other systems cannot further increase throughput due to out-of-memory. "FlexGen(c)" is FlexGen with compression.

<img src="https://github.com/FMInference/FlexGen/blob/main/docs/throughput_vs_latency.jpg" alt="logo" width="500"></img>


## How It Works
FlexGen can be flexibly configured under various hardware resource constraints by aggregating memory and computation from the GPU, CPU, and disk. Through a linear programming optimizer, it searches for the best pattern to store and access the tensors, including weights, activations, and attention key/value (KV) cache. FlexGen further compresses both weights and KV cache to 4 bits with negligible accuracy loss. 

One key idea of FlexGen is to play the latency-throughput trade-off. Achieving low latency is inherently challenging for offloading methods, 
but the efficiency of offloading can be greatly boosted for throughput-oriented scenarios (see the figure above).
FlexGen utilizes a block schedule to reuse weight and overlap I/O with computation, as shown in figure (b) below, while other baseline systems use an ineffiicent row-by-row schedule, as shown in figure (a) below.

<img src="https://github.com/FMInference/FlexGen/raw/main/docs/block_schedule.jpg" alt="logo" width="500"></img>

More details can be found in [our paper](docs/paper.pdf).


## Install
Requirements:  
 - PyTorch >= 1.12 [(Help)](https://pytorch.org/get-started/locally/)

Instructions:
```
git clone https://github.com/FMInference/FlexGen.git
cd FlexGen
pip3 install -e .

# (Optional) Install openmpi for multi-gpu execution
# sudo apt install openmpi-bin
```

## Get Started with a Single GPU

### OPT-1.3B
To get started, you can try a small model like OPT-1.3B first. It fits into a single GPU so no offloading is required.
FlexGen will automatically download weights from Hugging Face.
```
python3 -m flexgen.flex_opt --model facebook/opt-1.3b
```

You should see some text generated by OPT-1.3B and the benchmark results.

### OPT-30B
To run large models like OPT-30B, you will need to use CPU offloading. You can try commands below.
The `--percent` argument specifies the offloading strategy for parameters, attention cache and hidden states separately.
The exact meaning of this argument can be found [here](https://github.com/FMInference/FlexGen/blob/9d092d848f106cd9eaf305c12ef3590f7bcb0277/flexgen/flex_opt.py#L1271-L1279).
```
python3 -m flexgen.flex_opt --model facebook/opt-30b --percent 0 100 100 0 100 0
```

### OPT-175B
To run OPT-175B, you need to download the weights from [metaseq](https://github.com/facebookresearch/metaseq/tree/main/projects/OPT) and convert the weights into Alpa [format](https://alpa.ai/tutorials/opt_serving.html#convert-opt-175b-weights-into-alpa-formats).
You can then try to offloaind all wieghts to disk by
```
python3 -m flexgen.flex_opt --model facebook/opt-175b --percent 0 0 100 0 100 0 --offload-dir YOUR_SSD_FOLDER
```

### How to set the offloading strategy and `--percent`?
We will release an automatic policy optimizer later, but now you have to manually try a few strategies.
The idea of high-throughput generation is to offload parameters and attention cache as much as possible to the CPU and disk if necessary.
You can see the reference startegies in our benchmark [here](https://github.com/FMInference/FlexGen/blob/9d092d848f106cd9eaf305c12ef3590f7bcb0277/benchmark/flexgen/bench_suite.py#L39-L79).
To avoid out-of-memory, you can tune the `--percent` of offload more tensors to the CPU and disk.

## Scaling to Distributed GPUs
If you have more GPUs, FlexGen can combine offloading with pipeline parallelism to allow scaling.
For example, if you have 2 GPUs but the aggregated GPU memory is less than the model size, you still need offloading. FlexGen allow you to do pipeline parallelism with these 2 GPUs to accelerate the generation.
See examples [here](https://github.com/FMInference/FlexGen/tree/main/benchmark/flexgen#distributed-gpus).

## Run Chatbot with OPT Models on a Single GPU
[apps/chatbot.py](apps/chatbot.py) shows how to build a chatbot with FlexGen and OPT models.
While FlexGen is mainly optimized for large-batch throughput-oriented scenarios like dataset evaluations and information extraction,
FlexGen can also be used for interactive applications like chatbot with better performance than other offloading-based systems.
Note that FlexGen cannot achieve its best throughput in this single-batch case.

### Default Commands
You can use the default commands below.
If you do not have enough GPU/CPU memory, see the [Handle Out-of-memory](#handle-out-of-memory) section.

```
# Chat with OPT-6.7B. You need at least 15GB of GPU memory.
python3 chatbot.py --model facebook/opt-6.7b
```

```
# Chat with OPT-30B. You need at least 64GB of CPU memory.
python3 chatbot.py --model facebook/opt-30b --percent 0 100 100 0 100 0
```

```
# Chat with instruction-tuned OPT-IML-MAX-30B. You need at least 64GB of CPU memory.
python3 chatbot.py --model facebook/opt-iml-max-30b --percent 0 100 100 0 100 0
```

### Example Output
```
A chat between a curious human and a knowledgeable artificial intelligence assistant.
Human: Hello! What can you do?
Assistant: As an AI assistant, I can answer questions and chat with you.
Human: What is the name of the tallest mountain in the world?
Assistant: Everest.
Human: I am planning a trip for our anniversary. What things can we do?
Assistant: Well, there are a number of things you can do for your anniversary. First, you can play cards. Second, you can go for a hike. Third, you can go to a museum.
```

### Handle Out-of-memory
If you do not have enough GPU/CPU memory, here are a few things you can try.
They save more memory but run slower.

- Enable weight compression by adding `--compress-weight`.
- Offload weights to disk by using `--percent 0 0 100 0 100 0`.

## Roadmap
We plan to work on the following features. Community contributions are welcome.

- [ ] Support Apple silicon M1/M2 deployment
- [ ] Support Colab deployement
- [ ] Optimize the latency of the chatbot application
- [ ] Add a text summarization application
- [ ] Support more models (BLOOM, CodeGen, GLM)
- [ ] Release the cost model and policy optimizer
- [ ] Release a pip installable package
