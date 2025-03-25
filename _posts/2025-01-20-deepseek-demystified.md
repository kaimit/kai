---
layout: post
title: "DeepSeek V3 & R1 Demystified: Low cost, R1-Zero, and more"
date: 2025-01-20
categories: [AI, LLMs, Research]
---

As DeepSeek shocked the world with its cost and performance, I decided to dig deeper into the technical details of DeepSeek v3 and R1. 

# Is DeepSeek 10 times cheaper than models from the West?
10 times cheaper than Llama. Probably not 10 times cheaper than other closed-source models. 

From the technical report of DeepSeek v3 ([link](https://arxiv.org/html/2412.19437v1)), we can find out that the train cost was broken down to: 

| Stage | H800 GPU Hours | Cost (USD) |
|:---|:---|:---|
| Pretraining | 2664K | $5.328M |
| Context Extension | 119K | $0.238M |
| Post-Training | 5K | $0.01M |
| **Total** | **2788K** | **$5.576M** |

As a comparison, let's take a look at the training cost of Llama 3 405B. According to Llama 3 report ([link](https://arxiv.org/abs/2407.21783)), 405B was pre-trained using 16K H100 GPUs for 54 days, which is 16K * 54 * 24 = 20,736K H100 GPU hours. 

Let's summarize the comparison between DeepSeek v3 and Llama 3 405B: 

| Model | Architecture | Parameters | H800 GPU Hours | Pre-Training Cost (USD) |
| :---|:---|:---|:---|:---|
| DeepSeek v3 | MoE | 671B, with 37B active | 2664K | $5.328M |
| Llama 3 405B | Dense | 405B | 20,736K | $41.5M |

From the table, we can see that DeepSeek v3 is 10 times cheaper than Llama 3 405B in terms of pre-training cost. 
(Assume the cost of H100 GPU hour is the same as H800 GPU hour, $2/GPU hour, which was used in DeepSeek v3 technical report)

Why is the training cost of DeepSeek v3 so cheap? The most important reason is the MoE architecture. We can estimate the pre-training cost of a GPT model roughlyusing this formula: 

Cost = Total FLOPs needed / (GPU Peak FLOPs * mfu) * GPU hour cost
 ~ k * Active Parameters * Tokens Amount / (GPU Peak FLOPs * mfu) * GPU hour cost

where k is a constant, which is around 6 for dense model, and more than 6 for MoE model. 

For simplicity, let's assume k = 6, and  mfu and GPU hour cost are the same for both models. The we can see that Active parameters is the only factor that affects the cost. As long as you use MoE architecture, the cost will be cheaper. 

I am sure other companies are also working on MoE so the cost gap between DeepSeek v3 and other models should not be as big as 10 times. However, I believe other companies are not as aggressive as DeepSeek in using MoE and pushing the limit of cost efficiency. 

Are other factors also important?
## MLA
Very important to reduce inference cost. 93.3% reduction of KV cache during inference according technical report of v2. 
But not as important as MoE for training. 

## FP8 Training
Theoretically FP8 training can reduce the training cost by half compared to BF16. However, in practice, the cost is not reduced as much as expected. 


# How efficient is DeepSeek's training run? 
One ultimate metric to measure the efficiency of training system is the MFU of training cluster. Let's do the math here. 

|Model | k | Number of Activated Parameters | Total Tokens/T | FLOPS | GPU count | Peak TFLOPS per GPU | Training Time/GPU Hours | MFU |
|:---|:---|:---|:---|:---|:---|:---|:---|:---|
| DeepSeek v3 | 6 | 37000000000 | 14.8 | 3.2856E+24 | 2048 | 1700 | 180,000.00 | 20.2% |
| Llama 3 405B | 6 | 4.05E+11 | 15 | 3.645E+25 | 16384 | 1000 | 1,572,864.00 | 42.9% (calculated) / 38% (reported) |

Note: the peak TFLOPS per GPU is roughly 1000 (in BF16) and 2000 (in FP8) for both H100 and H800. However, DeepSeek v3 used 20 out of 132 SMs for communication and thus the peak TFLOPS per GPU is only 2000 * 112/132 = 1700.  

Based on this estimate, DeepSeek v3 is only 20.2% efficient in terms of MFU, which seems quite low. However, FP8 training is not really mature yet and there must quite a lot of overhead in training. If we convert it to MFU in BF16, it is 40.4%, which is roughly the same as Llama 3 405B training.

There are two interpretations of this result:
1) The gain in efficiency is not significant despite the low cost and all the innovations.
2) DeepSeek's training system is so efficient that it is comparable or even better than Meta's. Let's consider DeepSeek is a very small company and Meta is a giant company with years of investment in all kinds of optimization. That result is pretty impressive. 


# Why is DeepSeek v3 so good at counting 'r's in strawberry?
I asked various versions of counting 'r's questions (count 'r's in strawberry, Raspberry; count 'e's in DeepSeek) across all models, including DeepSeek v3, Claude-3.5-sonnet, GPT-4o, Gemini 1.5 Pro and Doubao. DeepSeek v3 got all of them right while other models failed to do so. A close second is Claude-3.5-sonnet but not as good as v3. 

I think there are two main reasons for this: 
1) The tokenizer of DeepSeek v3 is a byte level BPE, which makes it naturally better at counting characters than models that use sub-word level BPE;
2) The post-training of V3 has a strong focus on reasoning, making it very good at reasoning problems. 



# R1-Zero: Pure RL without supervision signal?
In the technical report of R1, they claimed that they used pure RL to train R1-Zero, i.e. without using SFT to cold start. 

However, SFT is not the only way to provide supervision signal. 

The base model of V3 actually achieved a score of 15.6 (pass@1) at AIME, which is not zero and actually pretty high compared to other models. As a comparison, GPT-4o only achieved 13 (pass@1) at AIME. As a result, there must have been very good supervision signal in pre-training of the base model of V3, which acts as good "cold start" for RL.