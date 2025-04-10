---
layout: post
title: "Making Sense of the H100, NVLink, and InfiniBand Jargon: Simplifying Distributed AI Training"
date: 2025-04-10
categories: [AI, Distributed Training, Fine-Tuning]
tags: [PyTorch, Kubernetes, InfiniBand, RDMA, Distributed Training]
author: "Saurabh Salunkhe"
---

## Introduction
Let's face it: understanding terms like H100, tensors, NVLink, InfiniBand, RDMA, and Kubernetes can feel like decoding an alien language. Imagine a world where everyone speaks in fancy acronyms and technical jargon—it’s like walking into a restaurant where the menu is written in Sanskrit! In this blog, we’ll break down how large language models are trained and fine-tuned using these technologies. We’ll use simple analogies and practical examples to make the complex world of distributed AI training easy to grasp—even for non-technical folks.

## Why Distributed Training? A Simple Analogy
Picture this: You're tasked with copying an entire library by hand. Doing it alone (using a single GPU) would take forever. Instead, you assemble a team of scribes (multiple GPUs) to copy different sections simultaneously. But for the final library to look perfect, every scribe must share their progress, ensuring consistency. In AI, distributed training works in the same way:
- **Each GPU processes a chunk of data.**
- **Periodically, they synchronize (share their "copying" progress) to ensure the final model is accurate.**
This teamwork across GPUs is what makes training enormous models feasible.

## The Infrastructure: Multi-GPU and Multi-Node Systems
Imagine your team of scribes working in various kitchens:
- **Multi-GPU (Single Node):** One kitchen with many burners (GPUs) sharing ingredients (data) over a fast internal link (NVLink).
- **Multi-Node (Cluster):** Multiple kitchens (servers) spread out over a city (data center), connected by super-fast bullet train roads (InfiniBand or RoCE) that speed up the delivery of ingredients.

This is the backbone of modern AI supercomputers—a setup where numerous GPUs across various nodes work together like a finely tuned orchestra.

## The Fast Lane: Communication Technologies Explained
For all these GPUs to stay in sync, they need to talk to each other quickly. Think of it like using instant messaging versus sending letters:
- **InfiniBand:** The bullet train of networking—ultra-fast and low latency.
- **RDMA:** Allows one GPU to directly “peek” into another's memory, bypassing lengthy protocols.
- **RoCE:** Brings RDMA’s speed to standard Ethernet by creating dedicated express lanes.

These technologies ensure that our distributed training system functions smoothly, with each GPU quickly sharing its “ideas” (gradients) with its peers.

## Orchestrating the Madness: Kubernetes and Containerization
Now, imagine you’re not just managing a team of scribes—you’re coordinating several kitchens across town. Rather than yelling orders to each chef, you use a smart management system that:
- **Deploys standardized meal kits (containers).**
- **Assigns tasks to each kitchen (scheduling GPUs on nodes).**
- **Restarts operations if something goes wrong.**

This is where **Kubernetes** comes into play. It’s the maestro that launches and manages containerized training jobs, ensuring that every part of the process works in unison without manually logging into hundreds of machines.

## A Practical Glimpse: Running a Distributed Training Job

### Example 1: Launching a PyTorch Distributed Job
Assume you have two servers (nodes), each with 4 GPUs:
```bash
# On Node 0 (master)
torchrun --nnodes=2 --node_rank=0 --nproc_per_node=4 \
         --master_addr="10.0.0.1" --master_port=29500 train.py

# On Node 1 (worker)
torchrun --nnodes=2 --node_rank=1 --nproc_per_node=4 \
         --master_addr="10.0.0.1" --master_port=29500 train.py


These commands launch your training script across 8 GPUs (4 per node), ensuring that every process connects via the designated master address and port. Think of it as calling a conference where every chef joins in to share their progress in real time.

### Example 2: Orchestrating a Job with Kubernetes
Here’s a simplified Kubernetes YAML snippet to launch a training job:

apiVersion: batch/v1
kind: Job
metadata:
  name: llm-training-job
spec:
  template:
    spec:
      containers:
      - name: trainer
        image: myregistry/llm-trainer:latest
        command: ["torchrun"]
        args: ["--nnodes=4", "--node_rank=$(NODE_RANK)", "--nproc_per_node=4",
               "--master_addr=trainer-0", "--master_port=29500", "train.py"]
        env:
        - name: NODE_RANK
          value: "0"  # Adjust for each pod instance
        resources:
          limits:
            nvidia.com/gpu: 4
      restartPolicy: OnFailure

This configuration tells Kubernetes to allocate 4 GPUs for each pod, sets up the necessary environment variables, and manages the training process automatically—so even if one "kitchen" has a hiccup, the job continues with minimal fuss.

Conclusion: From Complex Jargon to Simple Understanding
Training and fine-tuning large language models involves much more than just powerful GPUs—it's about orchestrating a whole ecosystem where hardware, distributed training frameworks, and orchestration tools like Kubernetes work together seamlessly. Think of it as a massive, coordinated kitchen where every chef (GPU) contributes to creating a masterpiece.

Even if terms like H100, NVLink, and InfiniBand seem intimidating at first, they are simply the tools that enable this organized collaboration. By breaking down these concepts into everyday analogies and practical examples, we hope you now have a clearer picture of how distributed AI training works—and why it’s crucial for building the next generation of intelligent systems.

Happy training, and may your GPUs never idle!