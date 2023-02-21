# tensor_parallel
[![PyPI version](https://img.shields.io/pypi/v/tensor-parallel.svg?color=blue)](https://pypi.org/project/tensor-parallel/)
[![Black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)
[![CI status](https://github.com/BlackSamorez/tensor_parallel/actions/workflows/run-tests.yaml/badge.svg?branch=main)](https://github.com/BlackSamorez/tensor_parallel/actions)

Run large PyTorch models on multiple GPUs in one line of code.

```python
import transformers
import tensor_parallel as tp
tokenizer = transformers.AutoTokenizer.from_pretrained("facebook/opt-13b")
model = transformers.AutoModelForCausalLM.from_pretrained("facebook/opt-13b")  # use opt-125m for testing

model = tp.tensor_parallel(model, ["cuda:0", "cuda:1"])  # <- each GPU has half the weights

inputs = tokenizer("A cat sat", return_tensors="pt")["input_ids"].to("cuda:0")
outputs = model.generate(inputs, num_beams=5)
print(tokenizer.decode(outputs[0])) # A cat sat on my lap for a few minutes ...

model(input_ids=inputs, labels=inputs).loss.backward()  # training works as usual
```

## Installation
Latest stable version (recommended):
```
pip install tensor_parallel
```
Bleeding edge version:
```
pip install https://github.com/BlackSamorez/tensor_parallel/archive/main.zip
```


## Usage


Simply wrap your PyTorch model with `tp.tensor_parallel` and use it normally.
For best memory efficiency, call `tp.tensor_parallel` while the model is still on CPU.  

Here's a few use cases:
- [`examples/training_flan-t5-xl.ipynb`](./examples/training_flan-t5-xl.ipynb) - fine-tune full FLAN-T5 model on text summarization
- __TBA__ - inferencing a large language model with LLM.8bit + tensor_parallel
- __TBA__ - defining custom parallelism strategy


Advanced parameters to `tensor_parallel`:
- `device_ids: List[device]` - which devices to use; defaults to all available GPUs
- `output_device: device` - model outputs will have this device
- `config: tp.Config` - use custom parallelism strategy, see [`slicing_configs.py`](./tensor_parallel/slicing_configs.py)
- `distributed: bool` - if True, use torch.distributed backend instead of threading (requires `torchrun`)
- `sharded: bool` - if True, find all trainable parameters that weren't split by Tensor Parallelism and split them using [ZeRO-3 algorithm](https://deepspeed.readthedocs.io/en/latest/zero3.html).
   - weights will be split between GPUs and re-assembled before each forward pass
   - TL;DR use this when training to avoid duplicate parameters (enabled by default!) 
   - `sharded_param_names: List[str]` - parameter names that should be sharded this way, default = found automatically

  
## FAQ

- __Q:__ I don't have a multi-GPU server. Can I use tensor_parallel in Google Colab?
- __A:__ Colab has a single GPU, so there's no point in tensor parallelism. However, [Kaggle offers two T4 for free](https://www.kaggle.com/code/muellerzr/multi-gpu-and-accelerate) to all phone-verified accounts.


- __Q:__ What is tensor parallelism?
- __A:__ You split each layer's weights into parts, multiply each part on a separate GPU, then gather results. Read more [here](https://colossalai.org/docs/concepts/paradigms_of_parallelism/)
 

- __Q:__ Should I use `TensorParallel` or `DataParallel`?
- __A:__ TensorParallel for large models, DataParallel for smaller ones


- __Q:__ How does it compare against FullyShardedDataParallel and ZeRO?
- __A:__ ZeRO is better if you can fit a large batch, TensorParallel is better for small batches


Why use `tensor_parallel` ...
- v.s. [DeepSpeed](https://github.com/microsoft/DeepSpeed) and [FairScale](https://github.com/facebookresearch/fairscale/)
  - DeepSpeed has many parallelization strategies, but requires careful configuration
  - tensor_parallel has one strategy that works with 1 line of code
  - tensor_parallel works in a jupyter notebook
- v.s. [MegatronLM](https://github.com/NVIDIA/Megatron-LM)?
  - MegatronLM has _great_ tensor parallelism for one model architecture
  - tensor_parallel has _good_ parallelism for any architecture
  - tensor_parallel is way easier to install
- v.s. [parallelformers](https://github.com/tunib-ai/parallelformers)?
  - parallelformers implements a fixed [list of architectures](https://github.com/tunib-ai/parallelformers/tree/main/parallelformers/transformers)
  - tensor_parallel works for any architecture automatically 
  - parallelformers is inference-only, tensor_parallel supports training
- v.s. [`alpa`](https://github.com/alpa-projects/alpa)
  - alpa is a powerful tool for automatic distributed training / inference in JAX
  - tensor_parallel works with PyTorch
- v.s. [`Model.parallelize()`](https://huggingface.co/docs/transformers/model_doc/gpt2#transformers.GPT2Model.parallelize)?
  - both are easy to use, both fit large models
  - in parallelize, one GPU works at a time
  - in tensor_parallel, GPUs work in parallel

In short, use `tensor_parallel` for quick prototyping on a single machine.
Use DeepSpeed+Megatron or alpa for million-dollar training runs.


## Troubleshooting

If you experience NCCL errors, or random hanging, you may have some code errors that are not displayed properly. 
To debug these errors, we recommend restarting with `export TENSOR_PARALLEL_USE_NATIVE=1` or a on single device. 

If you found a bug or encountered a problem, please report it to [our issue tracker](https://github.com/BlackSamorez/tensor_parallel/issues).
We will do our best to help, but it may take some time before we get to it.
Please create issues only if your problem is specifically with `tensor_parallel`.
For example, if you need help installing `transformers` or optimizing your code, please seek it elsewhere.

### Code style

We use [black](https://black.readthedocs.io/en/stable/the_black_code_style/current_style.html) and [isort](https://pycqa.github.io/isort/) for all pull requests.
Before committing your code, simply run `black . && isort .` and you will be fine.

--------------------------------------------------------------------------------
