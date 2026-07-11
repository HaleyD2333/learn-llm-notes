# Fundational Architecture
## Transformer (2017)
Encoder-decoder for machine translation; self-attention.

Attention is All you Need

## BERT (2018)
Encoder-only, bidirectional. Masked language modeling (predict masked tokens using both left and right context). Dominated understanding / classification tasks. Key idea is to pretrain then fine-tune.

[Pre-training of Deep Bidirectional Transformers](https://arxiv.org/abs/1810.04805)

## GPT1 / GPT2
Decoder-only, autoregressive. GPT-2 showed that scale + next-token prediction yields surprising zero-shot ability.

GPT-2: Language Models are Unsupervised Multitask Learners

## T5 (2019)
Encoder-decoder. Reframed every NLP task as text-to-text ("translate to..", "summarize:.."). Clean unifying framework.

[Exploring the Limits of Transfer Learning with T5](https://arxiv.org/abs/1910.10683)

# The Scaling Era
## GPT-3 (2020)

175B params. Demonstrated in-context / few-shot learning: no fine-tuning, just examples in the prompt. The model that made "prompting" a discipline.

Language Models are Few-Shot Learners

## Scaling Laws (2020) & Chinchilla (2022)
Not models, essential theory. Chinchilla corrected GPT-3's recipe: for a fixed compute budget, you should train a smaller model on more tokens (data and params should scale together)

- Kaplan et al - Scaling Laws for Nueral LMs
- Hoffmann et al - Training Compute-Optimal LLMs (Chinchilla)


# Alignment and Instruction Following
## InstructGPT (2022)
Introduced RLHF. The bridge from GPT-3 to ChatGPT.
- Training language models to follow instructions with human feedback

## DPO (2023)
Simpler alternative to RLHF that skips the reward model and does alignment via a direct classification-style loss. Now widely used.
- Direct Preference Optimization

# Open-weigth models
## LLaMA / Llama 2 / Llama 3 (2023-2024, Meta)

- LlaMA: Open and Efficient Foundation Models
- Llama 2

## Mistral 7B / Mixtral (2023, Mistral AI)
Strong small model. MoE (Mixture-of-Experts) architecture, only a subset of expert sub-networks activate per token -> more parameters, similar compute.
- Mistral 7B
- [Mixtral of Experts](https://arxiv.org/abs/2401.04088)

## Qwen, Gemma, Phi
- Qwen (Alibaba)
- Gemma (Google)
- Phi (Microsoft)

# Reasoning models
## DeepSeek-R1 (2025)
Large-scale RL can elicit strong reasoning with open weights. The shift toward inference-time reasoning.
- Deepseek-R1

## Chain-of-Thought prompting (2022)
The technique behind reasoning models: prompting the model to "think step by step" dramatically improves multi-step problem solving.
- Chain-of-Thought Prompting Elicits Reasoning


Materials
- Hugging Face Open LLM Learderboard and [Model docs](https://huggingface.co/docs/transformers/index)
- [Understanding LLM](https://magazine.sebastianraschka.com/p/understanding-large-language-models)
- Stanford CS25: [Transformers United](https://web.stanford.edu/class/cs25/)
- [A Survey of Large Language Models](https://arxiv.org/abs/2303.18223)