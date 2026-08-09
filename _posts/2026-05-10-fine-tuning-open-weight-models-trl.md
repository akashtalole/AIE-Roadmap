---
title: "Fine-Tuning Open-Weight Models with Hugging Face TRL"
date: 2026-05-10 08:00:00 +0530
categories: [AI, Fine-Tuning]
tags: [fine-tuning, fine-tuning-series, huggingface, python, open-source]
---

Managed APIs cover most needs but hand you neither the weights nor control over training details. TRL (Transformer Reinforcement Learning, despite the name it covers supervised fine-tuning too) is Hugging Face's library for the fully self-hosted alternative — full control, at the cost of managing your own training infrastructure.

## Loading Data and Model

```python
from datasets import load_dataset
from transformers import AutoModelForCausalLM, AutoTokenizer
from trl import SFTTrainer, SFTConfig
from peft import LoraConfig

dataset = load_dataset("json", data_files={"train": "train.jsonl", "validation": "val.jsonl"})
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-3-8b-Instruct")
model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3-8b-Instruct")
```

## Configuring the Training Run

```python
peft_config = LoraConfig(r=16, lora_alpha=32, target_modules=["q_proj", "v_proj"], task_type="CAUSAL_LM")

training_config = SFTConfig(
    output_dir="./ft-output",
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,
    num_train_epochs=3,
    learning_rate=2e-4,
    logging_steps=10,
    eval_strategy="steps",
    eval_steps=50,
    save_strategy="steps",
    save_steps=50,
    load_best_model_at_end=True,
)

trainer = SFTTrainer(
    model=model,
    args=training_config,
    train_dataset=dataset["train"],
    eval_dataset=dataset["validation"],
    peft_config=peft_config,
)
trainer.train()
```

`load_best_model_at_end=True` combined with periodic eval is what prevents shipping an overfit late checkpoint — the trainer keeps track of validation performance across checkpoints and restores whichever one scored best, not simply whichever came last.

## Why Batch Size and Gradient Accumulation Both Show Up

`per_device_train_batch_size=4` with `gradient_accumulation_steps=4` gives an effective batch size of 16 without needing memory for 16 examples at once — gradients accumulate across the smaller batches before a single optimizer step. This is the standard way to fit a target effective batch size into limited GPU memory, directly relevant after yesterday's QLoRA memory-budget discussion.

## Saving and Reloading the Adapter

```python
model.save_pretrained("./ft-output/final-adapter")

# Later, or on a different machine:
from peft import PeftModel
base_model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3-8b-Instruct")
model = PeftModel.from_pretrained(base_model, "./ft-output/final-adapter")
```

## What Self-Hosting Actually Buys You

Full ownership of the weights, no per-token inference markup from a provider, complete control over data mixing and every hyperparameter, and the ability to fine-tune on genuinely sensitive data that shouldn't leave your infrastructure at all. It costs you the GPU provisioning, training infrastructure, and monitoring work a managed API would otherwise absorb — a real tradeoff, not a strictly better option.

---

*Part of the [Fine-Tuning series]({{ site.baseurl }}/tags/fine-tuning-series/) — next: [supervised fine-tuning step by step]({{ site.baseurl }}/posts/supervised-fine-tuning-step-by-step/), covering what actually happens during the `.train()` call above.*
