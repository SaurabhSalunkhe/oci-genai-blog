---
layout: post
title: "Fine-Tuning DeepSeek-R1 for Healthcare: Generating Patient Discharge Summaries"
date: 2025-03-09
categories: [Healthcare, AI, Fine-Tuning]
tags: [DeepSeek, Quantization, LoRA, Healthcare AI]
author: "Saurabh Salunkhe"
---
## Introduction
In this blog post, we will explore high-level steps to fine-tune a large language model, specifically selecting the DeepSeek-R1-Distill-Qwen-1.5B for generating patient discharge summaries tailored to healthcare needs. We wll discuss why fine-tuning is crucial for improving model accuracy and relevance compared to alternatives like Retrieval-Augmented Generation (RAG). Key steps, including installation, quantization, and LoRA, will be highlighted with code snippets

## Sample Dataset overview

Sample Dataset Overview
The sample dataset consists of patient records (artificially generated) like the below, each with a concise clinical note and a corresponding detailed discharge summary. Here is an example from the dataset:

Clinical Note: "36 y/o female, adm 2025-03-07, c/o headache x 1 week. Dx: appendicitis. PMH: hyperlipidemia. Tx: azithromycin 500 mg x 1 then 250 mg QD x 4 days, albuterol PRN. CT scan: 90% RCA occlusion. Labs: CRP 15 mg/L. D/c 2025-03-07."


Discharge Summary: "Ms. Joseph Morton, 36, was admitted on 2025-03-07 with headache for 1 week, diagnosed with appendicitis. History of hyperlipidemia. Treated with azithromycin 500 mg x 1 then 250 mg QD x 4 days, albuterol PRN. CT scan revealed 90% RCA occlusion. Labs indicated CRP 15 mg/L. Discharged on 2025-03-07. Follow up with your cardiologist in 1 week."

This dataset represents a sequence-to-sequence task where the model must transform short, jargon-heavy notes into structured, patient-friendly summaries.


## Step-by-Step Guide to Fine-Tuning

I. Installing and Loading the Model from Hugging Face
First, set up the environment and load the model and tokenizer from Hugging Face.

{% highlight ruby %}

!pip install transformers torch peft bitsandbytes

from transformers import AutoModelForCausalLM, AutoTokenizer

model_name = "deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B"
auth_token = "hf_YourAuthTokenHere"  # Replace with your Hugging Face token

model = AutoModelForCausalLM.from_pretrained(
    model_name,
    use_auth_token=auth_token,  # omit if the model is public
    trust_remote_code=True      # required for models with custom code
)

tokenizer = AutoTokenizer.from_pretrained(
    model_name,
    trust_remote_code=True
)

input_text = "Clinical Note: 36 y/o female, adm 2025-03-07, c/o headache x 1 week."
inputs = tokenizer(input_text, return_tensors="pt")
outputs = model.generate(**inputs, max_new_tokens=50)
print(tokenizer.decode(outputs[0], skip_special_tokens=True))

{% endhighlight %}

Explanation:

The transformers library provides tools to load pre-trained models and tokenizers.
trust_remote_code=True is necessary if the model includes custom code from Hugging Face.
The test inference checks if the model is working, though the output may not yet be healthcare-specific without fine-tuning.

II. Using Quantization

Quantization reduces the models memory footprint and speeds up inference by lowering the precision of weights (e.g., from 32-bit floats to 8-bit integers).

Why Use Quantization?
Reduced Memory Usage: Allows deployment on devices with limited resources, such as hospital servers or edge devices.

Faster Inference: Speeds up prediction, which is vital for real-time healthcare applications.

Energy Efficiency: Lowers computational costs, making it more sustainable.


{% highlight ruby %}

from transformers import BitsAndBytesConfig
import torch

quant_config = BitsAndBytesConfig(
    load_in_8bit=True,                # Use 8-bit precision
    bnb_8bit_use_double_quant=True,   # Improves accuracy with double quantization
    bnb_8bit_quant_type="nf8",        # Normal float 8-bit type
    bnb_8bit_compute_dtype=torch.bfloat16  # Computation in bfloat16 for efficiency
)

print("Loading base model with quantization...")
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    token=auth_token,
    trust_remote_code=True,
    quantization_config=quant_config,
    torch_dtype="auto",
    device_map="auto"  # Automatically maps model to available devices
)

{% endhighlight %}

Explanation:

BitsAndBytesConfig configures the quantization settings.
device_map="auto" ensures the model is distributed across available GPUs or CPUs, optimizing resource use

III. Implementing LoRA for Efficient Fine-Tuning

LoRA (Low-Rank Adaptation) is a parameter-efficient technique that fine-tunes a model by adding low-rank matrices to specific layers while keeping the original weights frozen.

What is LoRA?
LoRA approximates weight updates with a low-rank decomposition, training only a small subset of parameters. This reduces memory and computational requirements, making it ideal for fine-tuning large models like DeepSeek-R1-Distill-Qwen-1.5B on specialized tasks.

How to Choose LoRA Parameters
Rank (r): Controls the capacity of adaptation. Start with r=8 or r=16; higher values increase expressiveness but require more memory.
Alpha: Scaling factor for LoRA updates. A value of 32 is a good starting point; adjust based on task complexity.
Target Modules: Focus on attention layers (e.g., q_proj, v_proj) where adaptation is most impactful. Optionally include feed-forward layers if needed.

{% highlight ruby %}

from peft import PeftModel, LoraConfig, get_peft_model

# Define LoRA configuration
lora_config = LoraConfig(
    r=16,                     # Rank of the low-rank matrices
    lora_alpha=32,            # Scaling factor
    target_modules=["q_proj", "v_proj"],  # Target attention layers
    lora_dropout=0.1,         # Dropout for regularization
    bias="none"               # No bias adaptation
)

# Apply LoRA to the model
model = get_peft_model(model, lora_config)

# Print trainable parameters to verify
model.print_trainable_parameters()

{% endhighlight %}

Explanation:

PeftModel and LoraConfig are from the peft library, designed for parameter-efficient fine-tuning.
target_modules specifies which layers to adapt; adjust based on the models architecture (check documentation if needed).
print_trainable_parameters() confirms only LoRA parameters are trainable, keeping the base model frozen.

IV. Fine-Tuning Hyperparameters
Beyond LoRA, tuning additional hyperparameters ensures optimal performance.

Key Hyperparameters
Learning Rate: Start with 2e-5. Reduce to 1e-5 if the model overfits (e.g., training loss decreases but validation loss increases).
Batch Size: Use 4 or 8, depending on GPU memory. Use gradient accumulation if memory is limited.
Epochs: Train for 3-5 epochs; monitor validation loss to prevent overfitting.


{% highlight ruby %}
from transformers import TrainingArguments, Trainer

# Sample dataset (convert to Dataset object in practice)
dataset = [
    {"input": "Clinical Note: 36 y/o female, adm 2025-03-07, c/o headache x 1 week. Dx: appendicitis. PMH: hyperlipidemia. Tx: azithromycin 500 mg x 1 then 250 mg QD x 4 days, albuterol PRN. CT scan: 90% RCA occlusion. Labs: CRP 15 mg/L. D/c 2025-03-07.",
     "output": "Ms. Joseph Morton, 36, was admitted on 2025-03-07 with headache for 1 week, diagnosed with appendicitis. History of hyperlipidemia. Treated with azithromycin 500 mg x 1 then 250 mg QD x 4 days, albuterol PRN. CT scan revealed 90% RCA occlusion. Labs indicated CRP 15 mg/L. Discharged on 2025-03-07. Follow up with your cardiologist in 1 week."}
    # Add other records here
]

# Define training arguments
training_args = TrainingArguments(
    output_dir="./fine_tuned_deepseek",
    num_train_epochs=3,
    per_device_train_batch_size=4,
    learning_rate=2e-5,
    save_steps=500,          # Save checkpoint every 500 steps
    eval_steps=500,          # Evaluate every 500 steps
    logging_dir="./logs",
    logging_steps=10         # Log every 10 steps
)

# Initialize Trainer
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=dataset,   # Replace with proper Dataset object
    eval_dataset=dataset     # Optional validation split
)

# Start training
trainer.train()

{% endhighlight %}

TrainingArguments configures the training process.
The dataset should be converted to a Dataset object using datasets library for real implementation.
Monitor logs and evaluation metrics to adjust hyperparameters if needed.

V. Generating Discharge Summaries
After fine-tuning, the model can generate summaries from clinical notes.


{% highlight ruby %}
prompt = "Clinical Note: Patient presents with chest pain. Discharge Summary:"

inputs = tokenizer(prompt, return_tensors="pt").to(model.device)

model.eval()

outputs = model.generate(**inputs, max_new_tokens=50)
print(tokenizer.decode(outputs[0], skip_special_tokens=True))

{% endhighlight %}

Explanation:

model.eval() disables training-specific operations like dropout.
max_new_tokens limits the output length; adjust based on desired summary detail.

Conclusion
Fine-tuning the DeepSeek-R1-Distill-Qwen-1.5B model with quantization and LoRA transforms it into a powerful tool for generating patient discharge summaries in healthcare. This approach outperforms RAG by internalizing medical knowledge, understanding jargon, ensuring security, and producing compliant documentation. The provided code snippets covering model loading, quantization, LoRA, hyperparameter tuning, and inference offer a practical starting point. Experiment with these techniques and the sample dataset to optimize performance for your specific healthcare use-case. By leveraging fine-tuning, you can create a tailored, efficient, and secure solution that meets the demands of modern healthcare documentation.