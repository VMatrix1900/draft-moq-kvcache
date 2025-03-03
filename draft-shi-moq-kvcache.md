---

title: "KVCache over MoQT"
abbrev: "KVCache"
category: info

docname: draft-shi-moq-kvcache-00
submissiontype: IETF
number:
date:
consensus:
v: 3
area: "WIT"
workgroup: "Media over QUIC"
keyword: AI inference, KVCache

author:
 -
    ins: H. Shi
    fullname: Hang Shi
    organization: Huawei Technologies
    email: shihang9@huawei.com
    country: China

normative:

informative:
  I-D.ietf-moq-transport:

  CacheGen:
    title: >
      CacheGen: Fast Context Loading for Language Model Applications via KV Cache Streaming (SIGCOMM24)
    author:
    org:
    date: 2024
    target: https://github.com/UChi-JCL/CacheGen

  CacheBlend:
    title: >
      CacheBlend: Fast Large Language Model Serving for RAG with Cached Knowledge Fusion
    author:
    org:
    date: 2024
    target: https://arxiv.org/abs/2405.16444

--- abstract

Large language model (LLM) inference involves two stages: prefill and decode. The prefill phase processes the prompt in parallel, generating the KVCache, which is then used by the decode phase to produce tokens sequentially. KVCache can be reused if the model and prompt is the same, reducing computing cost of the prefill. However, its large size makes efficient transfer challenging. Delivering these over architectures enabled by publish/subscribe transport like MoQT, allows local nodes to cache the KVCache to be later retrieved via new subscriptions, saving the bandwidth. This document specifies the transmission of KVCache over MoQT.

--- middle

# Introduction: KVCache in LLM inference

The inference process of large language models is typically divided into two distinct stages: prefill and decode. The prefill phase processes the input prompt in parallel, generating a KVCache, which serves as an essential input for the decode phase. The decode phase then utilizes the KVCache to generate output tokens sequentially, one at a time. Prefill is a computationally intensive process, whereas decoding is constrained by memory bandwidth. Due to their differing resource requirements, prefill and decode processes are often deployed on separate computing clusters using different hardware chips optimized for computational performance in prefill nodes and memory bandwidth efficiency in decode nodes, with KVCache transferred between them.

~~~
               +--------------------+
               |    Prompt Input    |
               |  (System + User)   |
               +--------------------+
            Tokenization |
                ---------------------
                |                   |
                v                   |
    +--------------------+          |
    |   Prefill Nodes    |          |
    | (Generate KVCache) |          |
    +--------------------+          |
                |                   |
                v                   |
    +--------------------+          |
    |      KVCache       |<---------+
    | (Stored & Reused)  |
    +--------------------+
                |
      -----------------------------
      |              |            |
      v              v            v
+----------------+       +----------------+
|  Decode Node 1 |  ...  |  Decode Node N |
| (Use KVCache)  |       | (Use KVCache)  |
+----------------+       +----------------+

~~~
{: #fig-kvcache title='LLM inference process'}


KVCache is significantly large, with a single token requiring 160KB for a 70B model(8bit quantization). For a prompt of 1000 tokens, the KVCache size reaches 160MB. To reduce the size of KVCache, various quantization and compression algorithm are proposed such as {{CacheGen}}. Furthermore, KVCache can be reused across sessions if derived from the same prompt and model, as shown in {{fig-kvcache}}. The most basic reuse strategy is prefix caching, where KVCache is shared among prompts with a common prefix. More advanced methods, such as {{CacheBlend}}, improve reuse efficiency by selectively reusing KVCache beyond prefix matching. To minimize transmission costs, a publish/subscribe architecture is required to distribute KVCache. This document defines how to send KVCache over MoQT{{I-D.ietf-moq-transport}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the following terms:

- LLM: A large language model (LLM) that utilizes the attention mechanism to process and generate text efficiently by capturing long-range dependencies within input sequences.

- KVCache: A key-value cache storing intermediate representations used in LLM inference.

- Prompt: A prompt consists of two parts: the system prompt and the user prompt. The system prompt is predefined by the LLM model developer to guide the model's behavior, while the user prompt is provided dynamically by the user to specify the task or request.

- Token: The smallest unit of processing in LLM inference, typically representing a word or subword.

# KVCache Data Model

The KVCache data model is structured as follows.

**Naming**: The Track Namespace consisting of following tuples (moq://kvcache.moq.arpa/v1/),(modelName), (prompt) is defined in this specification. The track name identifies the compression level for the KVCache. Thus, a track name can be identified with the tuple (`<compression>`) and the full track name having the following format (when represented as a string):

```
moq://kvcache.moq.arpa/v1/<modelName>/<compression>
```

Following compressions are defined in this specification, along with their size:

| Compression  | Description                                   | Size per Weight |
| ------------ | --------------------------------------------- | --------------- |
| FP16        | Quantized using FP16                         | 2 bytes         |
| BF16        | Quantized using BF16                         | 2 bytes         |
| FP8         | Quantized using FP8                          | 1 byte          |
| Int8        | Quantized using Int8                         | 1 byte          |
| FP4         | Quantized using FP4                          | 0.5 byte        |
| Int4        | Quantized using Int4                         | 0.5 byte        |
| AC (5x)     | Compressed using Arithmetic Coding (5x ratio)| Variable        |
{: #tab-kvcache-compression title='Compression of KVCache'}

**Group ID**: Normally the tokens are split into chunks of uniform length(typical value is 128). The KVCache are organized into groups corresponding into token chunks. The ID of the group represents the index of a token group within the KVCache.

**Object ID**: An identifier for a specific token within a group.

**Object Payload**: The content of the KVCache, which varies based on the compression algorithm used for storage and transmission.

# Security Considerations

TBD

# IANA Considerations

TBD

--- back
