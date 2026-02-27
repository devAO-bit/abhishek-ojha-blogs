---
title: "LLM Internals: Tokens, Context Windows, Temperature & Sampling Explained"
excerpt: "I spent time studying how large language models actually work under the hood — from how they tokenize text, to what a context window really is, to why temperature isn't just a creativity dial. Here's everything distilled into one practical guide."
tags:
  - AI
  - LLM
  - Machine Learning
  - NLP
date: "2026-02-24"
featured: false
coverEmoji: "🧠"
---

I recently went deep on a set of articles covering LLM internals — transformers, tokenization, context windows, temperature, Top-K, Top-P, frequency penalty, presence penalty. A lot of it is scattered across docs and papers, so I'm combining it all here into one guide I wish existed when I started.

If you've ever called the OpenAI API and blindly set `temperature: 0.7` without really knowing why — this is for you.

## Part 1: What is an LLM, Really?

A **Large Language Model (LLM)** is a neural network trained on a massive corpus of text to learn the statistical relationships between words. The key insight is that it doesn't "understand" language the way we do — it learns to predict: *given these previous tokens, what token is most likely to come next?*

The architecture almost every modern LLM is built on is the **Transformer**, introduced in the 2017 paper "Attention is All You Need." The transformer replaced recurrent networks (RNNs/LSTMs) with a mechanism called **self-attention**, which lets the model relate any word in a sequence to any other word — regardless of how far apart they are.

The classic transformer has two parts:

- **Encoder** — reads the input and builds a rich representation of it
- **Decoder** — generates output tokens one at a time, attending to both the encoder output and its own previous outputs

Models like BERT are encoder-only (good for classification). Models like GPT are decoder-only (good for generation). Models like T5 use both.

## Part 2: Tokens — The Currency of LLMs

Before any text reaches the model, it gets converted into **tokens**. A token isn't necessarily a word — it's a subword unit. The word `darkness` becomes two tokens: `dark` and `ness`. Each gets assigned a number (say, 217 and 655).

Shared token IDs help the model spot relationships. Both `darkness` and `brightness` end in `ness` (token 655), which signals to the model that these words may share a structural or semantic pattern — even before it learns their meanings.

### Types of Tokenization

- **Word tokenization** — splits on spaces. Simple, but struggles with unknown words.
- **Character tokenization** — breaks into individual letters. Very granular, very long sequences.
- **Subword tokenization** — the industry standard. Breaks rare words into known pieces (`unhappiness` → `un`, `happiness`). Used by GPT (BPE), BERT (WordPiece), and T5 (SentencePiece).

### Why Token Count Matters for Developers

Tokens are how APIs charge you and how models measure input/output length. A rough rule of thumb: **1 token ≈ 0.75 words** in English. So a 1,000-word document is roughly 1,300–1,500 tokens. Always budget for this when building LLM-powered features.

```js
// Rough token estimate for budgeting
const estimateTokens = (text) => Math.ceil(text.split(/\s+/).length * 1.35);
```

## Part 3: Context Window — the Model's Working Memory

The **context window** is the maximum number of tokens an LLM can process in a single call — both your input (prompt) and its output combined. Think of it as the model's working memory. Everything outside the context window simply doesn't exist to the model.

| Model | Context Window |
| --- | --- |
| GPT-3.5 | 4K–16K tokens |
| GPT-4o | 128K tokens |
| Claude 3.5 | 200K tokens |
| Gemini 1.5 Pro | 1M tokens |

### What This Means in Practice

If you're building a chatbot and the conversation grows beyond the context window, the model will simply forget the earlier messages. This is why long-running chat applications need a **context management strategy** — either summarising older turns, storing them in a vector database for retrieval, or sliding the window.

For document Q&A, a larger context window means you can feed an entire PDF at once rather than chunking and embedding it. But bigger context ≠ always better — models tend to have lower accuracy on information buried in the **middle** of a very long context (the so-called "lost in the middle" problem).

## Part 4: Temperature — Controlling Randomness

This is the parameter everyone knows but few fully understand.

When an LLM generates the next token, it doesn't just pick the highest-probability one — it samples from a probability distribution. **Temperature** is a scalar that reshapes that distribution before sampling.

- **Temperature → 0**: The distribution becomes a spike — the highest probability token wins almost every time. Deterministic, predictable, repetitive.
- **Temperature = 1**: The model samples directly from its learned probabilities. Balanced.
- **Temperature > 1**: The distribution flattens — lower-probability tokens get boosted. More creative, more surprising, more likely to hallucinate.

### Practical Temperature Guide

``` js
Temperature 0.0 – 0.3  →  Code generation, SQL queries, factual Q&A
Temperature 0.4 – 0.7  →  Summarisation, structured writing, chatbots
Temperature 0.8 – 1.2  →  Creative writing, brainstorming, ideation
Temperature > 1.2       →  Experimental only — output gets chaotic fast
```

A concrete example: ask an LLM to write a joke at `temperature=0` and you'll get the same joke every run. Set it to `1.0` and each run produces something different.

## Part 5: Top-K and Top-P — Smarter Sampling

Temperature alone can still let the model pick genuinely nonsensical tokens. **Top-K** and **Top-P** add a filtering step before sampling.

### Top-K Sampling

Limit the model to only the **K most probable next tokens**, then sample from those.

- `top_k=1` is equivalent to greedy decoding (always pick the top token)
- `top_k=50` gives the model 50 options to sample from

The downside: K is a fixed number, so in some situations you're cutting off genuinely plausible tokens, and in others you're including very unlikely ones.

### Top-P (Nucleus) Sampling

Instead of a fixed K, Top-P picks the **smallest set of tokens whose cumulative probability exceeds P**.

- `top_p=0.9` means: take the top tokens until their combined probability hits 90%, then sample from that set.

If the model is very confident (one token has 95% probability), the nucleus might contain just 1 token. If the model is uncertain, the nucleus expands to include more options. This adaptive behaviour is why Top-P tends to produce more coherent output than Top-K in practice.

### Combining Temperature + Top-P

Most APIs let you set both. The recommended pattern is:

```python
# Balanced creative generation
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": prompt}],
    temperature=0.8,
    top_p=0.95,
    max_tokens=500
)
```

Note: OpenAI recommends only altering one of `temperature` or `top_p` at a time — changing both simultaneously makes behaviour harder to reason about.

## Part 6: Frequency Penalty & Presence Penalty

These two parameters fight repetition, but in different ways.

### Frequency Penalty

Penalises tokens **proportional to how many times they've already appeared** in the output. The more a word has been used, the less likely the model is to use it again.

- Range: `-2.0` to `2.0`. Positive = discourage repetition.
- Low value (`0.3`): *"I really, really, really like this product. It's really amazing."*
- High value (`0.9`): *"I'm genuinely impressed with this product. It's exceptional and intuitive."*

Use frequency penalty when you want varied vocabulary across a long piece of text.

### Presence Penalty

Penalises tokens **simply for having appeared at all** — regardless of frequency. It's a flat penalty that encourages the model to introduce new topics and ideas.

- Low value: model stays on-topic, may circle back to the same ideas.
- High value: model actively seeks new angles, can wander off-topic.

### When to Use Which

``` js
Frequency Penalty  →  Long-form writing, technical docs, avoiding word repetition
Presence Penalty   →  Brainstorming, creative exploration, idea diversity
Both at once       →  Use sparingly — combined effect is hard to predict
```

## Part 7: Max Tokens & Stop Sequences

Two housekeeping parameters worth knowing:

**`max_tokens`** caps the length of the model's response. It doesn't make the model summarise — it just cuts off. Set it based on your expected output length, and always leave some headroom.

**Stop sequences** tell the model to stop generating when it hits a specific string. Useful for structured output:

```python
# Stop when the model writes "END" or a newline after code
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "List 3 items:"}],
    stop=["4.", "END"]  # stop before it writes a 4th item
)
```

## Putting It All Together

Here's a mental model for setting LLM parameters:

1. **Start with temperature** — decide how creative vs. deterministic you need the output.
2. **Add top_p** — keep it at 0.9–0.95 for most use cases. Only lower it if you're seeing incoherent outputs.
3. **Set max_tokens** — based on your expected output. Never leave it at the API default for production.
4. **Use frequency_penalty** if you're generating long text and seeing word repetition.
5. **Use presence_penalty** only if you need the model to explore different ideas rather than stay focused.

Understanding these levers doesn't just make you better at prompt engineering — it makes you a better AI product builder. The difference between a frustrating chatbot and a delightful one often comes down to three numbers: temperature, top_p, and max_tokens.
