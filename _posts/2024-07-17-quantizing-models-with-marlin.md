---
title: "Quantizing Models with GPTQ utilizing Marlin kernels"
date: "2024-07-17"
categories: AI
layout: post
---

In this post we cover the steps to quantize (4-bit) a big model using GPTQ and then using Marlin kernels to achieve better inference performance. Let´s first cover some of the basic before getting into the steps for quantization.

## What is GPTQ?

GPTQ is an accurate post-training quantization technique that allows for efficient deployment of large language models (LLMs) like GPT on consumer hardware. The key points about GPTQ are:

- **Layerwise Quantization**: GPTQ quantizes the weights of the LLMs one layer at a time, minimizing the error at the output.
- **Arbitrary Order Insight**: Quantizing weights in any fixed order can perform just as well, as the order doesn't matter as much as previously thought.
- **Lazy Batch Updates**: GPTQ introduces *lazy batch* updates, where it applies the quantization algorithm to a batch of columns at a time, updating only those columns and a corresponding block of the weight matrix. This helps utilize GPU compute better.
- **4-bit Quantization**: GPTQ can quantize models down to 4-bit precision with minimal performance degradation, allowing LLMs to run on consumer-grade hardware like RTX 3090 GPUs.

## What is Marlin?

Marlin stands for **M**ixed **A**uto-**R**egressive **Lin**ear kernel. It is a specialized CUDA kernel optimized for efficient inference of large language models (LLMs) quantized using GPTQ (Quantization for Large Language Models). It unlocks unprecedented performance for FP16xINT4 matrix multiplications (matmul), delivering close to ideal (4x) speedups for batch sizes up to 32 tokens.

In summary, GPTQ enables efficient quantization of LLMs to low precision, while Marlin provides a highly optimized CUDA kernel to run inference on these quantized models, delivering significant speedups on modern GPUs. The two techniques are complementary and work together to enable deployment of large language models on consumer hardware.

## Quantization Steps

Once we covered the basics, let's do the model quantization!

### Install dependencies

The first step is to install the needed dependencies to download the model from huggingface and perform the quantization of it. We use a python venv for it:

```bash
$ python -m venv quant-venv
$ source quant-venv/bin/activate
(quant-venv)$ cd quant-venv
(quant-venv)$ pip install datasets auto-gptq torch sentencepiece huggingface-hub
```

### Download model

Then we download the model. In our example we are using a big model based on Mixtral, named prometheus: https://huggingface.co/prometheus-eval/prometheus-8x7b-v2.0. This model is an alternative of GPT-4 evaluation when doing fine-grained evaluation of an underlying LLM.

To download the model we simply use huggingface-cli:

```bash
(quant-venv)$ huggingface-cli download prometheus-eval/prometheus-8x7b-v2.0 --local-dir prometheus-eval-8x7b --local-dir-use-symlinks False
```

### Quantization script

Once we have the model locally, we can use the next script to load it, quantize it and save it using Marli format.

```python
def quantize_model(model_path:str, compress_model_path: str, ds: str):
    from transformers import AutoTokenizer
    from datasets import load_dataset

    from auto_gptq import AutoGPTQForCausalLM, BaseQuantizeConfig

    print("Starting the quantization")

    # Apply GPTQ
    quantize_config = BaseQuantizeConfig(
        bits=4,                         # Only support 4 bit
        group_size=128,                 # Set to g=128 or -1 (for channelwise)
        damp_percent=0.1,
        desc_act=False,                 # Marlin does not support act_order=True
        model_file_base_name="model",   # Name of the model.safetensors when we call save_pretrained
    )
    print("Applying GPTQ for quantization")

    model = AutoGPTQForCausalLM.from_pretrained(
        model_path,
        quantize_config,
        device_map="auto")
        # device_map="cuda:5")  # Requires exporting CUDA_VISIBLE_DEVICES=5
        # max_memory={i:"75GB" for i in range(8)})

    print("Loading the dataset and tokenizers")
    MAX_SEQ_LEN = 2048
    NUM_EXAMPLES = 4096
    
    dataset = load_dataset(ds, split="train")
    tokenizer = AutoTokenizer.from_pretrained(model_path)
    
    ds = dataset.shuffle().select(range(NUM_EXAMPLES))
    def preprocess(example):
        chat = [
        	{"role": "user", "content": example["alpaca_prompt"]},
        	{"role": "assistant", "content": example["response"]},
    	]
        return {"text": tokenizer.apply_chat_template(chat,
                                                      tokenize=False)}
    ds = ds.map(preprocess)
    
    examples = [
        tokenizer(
            example["text"], padding=False, max_length=MAX_SEQ_LEN,
            truncation=True,
        ) for example in ds
    ]
    print("Loaded the dataset and tokenizers")
    
    print("Starting quantization")
    model.quantize(examples, cache_examples_on_gpu=False) 

    gptq_save_dir = f"{model_path}-gptq"
    print(f"Saving gptq model to {gptq_save_dir}")
    model.save_pretrained(gptq_save_dir)
    tokenizer.save_pretrained(gptq_save_dir)

    # Convert to Marlin
    print("Reloading in marlin format")
    marlin_model = AutoGPTQForCausalLM.from_quantized(
        gptq_save_dir,
        use_marlin=True,
        device_map="auto")

    print(f"Saving model in marlin format to {compress_model_path}")
    marlin_model.save_pretrained(compress_model_path)
    tokenizer.save_pretrained(compress_model_path)

    print("Quantization process completed")


ds = "VMware/open-instruct"
model_path="prometheus-eval-8x7b"
compressed_model="prometheus-eval-vmware-gptq"

quantize_gpu_model(model_path=model_path, compress_model_path=compressed_model, ds=ds)
```

We save the function in a python file (named `quantize.py`) and execute it with:
```bash
(quant-venv)$ python quantize.py
```

There are a few things to note in that `quantize_model` function:

#### Datasets

The dataset used for quantization is `VMware/open-instruct`, and we limit the amount of samples (after shuffling them) to 4096. The rationale is that after 2048 samples there isn't much gain on adding more samples while the computations are expensive.

Also depending on the dataset model you may need to adapt the `preprocess` function so that it creates an appropiate `chat` list to be parsed by the `apply_chat_template` function.

#### GPU usage

Instead of using `device_map="auto"`, you can specify a given GPU with `device_map="cuda:3"`, and make it visible with: `export CUDA_VISIBLE_DEVICES=3` before executing the function.

Also, you can load the model across several GPUs using `max_memory={i:"75GB" for i in range(8)}` (instead of the `device_map`), however the **quantization process will only use one GPU**, therefore there is little gain of doing this, plus it will use VRAM that could be needed for the quantization process.

#### Saving memory

Besides using `device_map="auto"` to have the model loaded on system memory (instead of VRAM), we can also save some extra VRAM by ensuring the dataset samples are not cached on the GPU itself by using `cache_examples_on_gpu=False` when calling the `quantize` function.

### Upload the model

Once we have the Marlin Quantized version (took me around 2 hours to run the complete process on a A100 NVIDIA GPU), we can proceed to upload it to huggingface (or an S3 bucket or similar):

```bash
(quant-venv)$ huggingface-cli login
     [ENTER TOKEN]
(quant-venv)$ huggingface-cli upload --repo-type model luis5tb/prometheus-eval-8x7b-gptq-marlin-vmwareds-4096 prometheus-eval-vmware-gptq
```

## Conclusions

There are several benefits of quantizing a model (specially big ones) at the expense of a possible impact on its accuracy:

- *Small model size*: 4-bit quantization means 75% reduction compared to base FP32 models.
- *Faster inference*: lower precision computations means faster and more efficient inference. The reduced memory footprint also descreses memory access costs.
- *Power efficiency*: due to the above points, the inference also requires less power consumption.
- *Enabling cheaper hardware*: the small size and the faster inference also enable more hardware to run the models. As an example the model used in this blogpost would require 4 A100 to run the base model, while the quantize model would only need 1 GPU.

This blog post just demonstrated one way of quantizing a model. There are many others, with different formats. Even this method has different configuration options that would lead to different outcomes.