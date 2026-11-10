---
layout: writing
title: "One cache flag, two attention backends: benchmarking Qwen3.8 on GB200"
description: "A prefix-cache benchmark reached 2.29× throughput, then a configuration check revealed different attention backends. Here is what the measurements actually establish."
date: 2026-11-09
article: true
permalink: /blog/qwen-gb200-prefix-cache/
---

I started with a small question: how much does prefix caching help when serving Qwen3.8-27B on a GB200?

The first results looked straightforward. At concurrency 32, the cache-enabled configuration delivered about 2,215 output tokens/s on a shared-prefix workload, compared with 967 tokens/s with caching disabled. The p95 time to first token fell from about 2.65 seconds to 197 milliseconds.

Then I compared the saved server configurations. The cache flag had changed, but so had the attention backend: Triton with caching enabled, TRT-LLM with caching disabled. Page size and recurrent-state cache settings had changed too.

That changed what I could claim. The 2.29× throughput difference was real for these runs, but it was a comparison of two resolved serving configurations. It did not isolate the benefit of caching alone. Understanding that distinction turned out to be the most useful part of the experiment.

## The model and serving setup

I served `Qwen/Qwen3.8-27B` on one GB200 GPU inside a Kubernetes pod, with the benchmark client running in the same pod against `localhost:30000`. Kubernetes supplied the execution environment; this experiment did not measure cluster scaling or service-to-service networking.

The checkpoint was the standard [Qwen3.8-27B repository on Hugging Face](https://huggingface.co/Qwen/Qwen3.8-27B). Its [configuration](https://huggingface.co/Qwen/Qwen3.8-27B/blob/main/config.json) declares BF16 for the text model and uses the `Qwen3_5ForConditionalGeneration` architecture identifier. The layers mix linear attention with full attention, with a full-attention layer every fourth layer. The architecture identifier also explains why Qwen3.5-named code appears in this investigation.

The weights are stored in Safetensors shards. Safetensors describes the file format; BF16 describes numerical precision. Multiple checkpoint files do not imply quantization or multiple GPUs. With tensor parallelism set to one, the required model weights are loaded onto one GPU.

My server arguments recorded `dtype=auto` and `quantization=null`; the command did not request FP8, INT4, or another quantization method. The checkpoint configuration supports the expectation of BF16, although `dtype=auto` in the server snapshot is not itself a measurement of every tensor's runtime dtype.

```bash
CUDA_VISIBLE_DEVICES=0 sglang serve \
  --model-path Qwen/Qwen3.8-27B \
  --host 0.0.0.0 \
  --port 30000 \
  --tp-size 1 \
  --context-length 32768 \
  --mem-fraction-static 0.80 \
  --reasoning-parser qwen3 \
  --tool-call-parser qwen3_coder
```

For the second configuration, I restarted the server with the same command plus `--disable-radix-cache`. Neither command explicitly selected an attention backend or page size.

The 32K context limit was a serving configuration, not the length of each benchmark request. Each measured request had 2,048 input tokens and generated 128 output tokens.

## What I measured

Prefix caching reuses state computed for a previously processed prompt prefix. A common application is a long system instruction shared across many requests, followed by a different user question.

I built a small streaming client around SGLang's native `/generate` endpoint. It sent token IDs directly, so tokenization and chat-template application were outside the timed request path. The reasoning and tool-call parser flags were retained in the launch command, but this was not a tool-use or chat-quality evaluation.

| Parameter | Value |
|---|---|
| Prompt patterns | Shared prefix and unique prefixes |
| Input length | 2,048 tokens |
| Shared workload | 1,984-token common prefix + 64-token request-specific suffix |
| Unique workload | Independently seeded synthetic prompts, without a deliberately shared long prefix |
| Output length | 128 tokens, with EOS stopping disabled |
| Sampling | Temperature 0 |
| Concurrency | 1, 8, 32 |
| Requests per trial | 96 |
| Repetitions | 3 per workload/concurrency combination |
| Server configurations | Cache enabled and cache disabled |
| Total measured requests | 3,456 |

The prompts were deterministic synthetic numeric text, tokenized and truncated to exact lengths. They controlled workload shape; they were not representative conversations. Independently generated prompts can still share incidental short token prefixes.

Before measurement, the harness issued warmup requests at each concurrency level. Before each trial, it requested a server cache flush. For shared-prefix trials, it processed the common prefix once outside the timed region, then submitted the measured requests. With caching disabled, the same priming request ran but could not create a reusable prefix cache entry.

This measures a **warm shared-prefix scenario**, not the cold request that first establishes the prefix. The 1,984-token prefix is the intended shared length; I did not instrument the exact number of tokens or recurrent-state checkpoints actually reused.

The client used a fixed number of worker threads. Each worker sent another request when its previous request completed. This is a closed-loop concurrency test, not a fixed arrival-rate test. Workload/concurrency trial order was shuffled within each repetition using a fixed seed. The two server configurations were run in separate blocks.

For example, the cache-enabled run used:

```bash
python qwen_bench.py \
  --label cache-on \
  --concurrency 1 8 32 \
  --requests 96 \
  --repeats 3 \
  --input-tokens 2048 \
  --output-tokens 128 \
  --out ./qwen-bench-results \
  --allow-cache-flush
```

The label only names the output. It does not enable or disable server caching.

## Reading the metrics

I recorded client-observed time to first token (TTFT), end-to-end request latency, output throughput, and average time per output token after the first token (TPOT).

TTFT starts when a worker begins its HTTP request and ends when it receives the first stream event reporting generated tokens. It includes HTTP and serving overhead, not just GPU prefill time. TPOT divides the remaining response time by the remaining token count; it is not a distribution of individual inter-token gaps.

Output throughput is successful output tokens divided by trial wall time, including client execution overhead. At concurrency 32 it is aggregate throughput across requests, not one request's generation speed.

All measured requests completed with the expected output-token count. That validates completion and token accounting, not semantic correctness or equivalence between configurations.

## Results: shared prefixes benefit under concurrency

Each table entry below is the median across three repetitions. For p95 metrics, I took the median of the three per-trial p95s, rather than pooling all requests into one percentile.

### Shared-prefix workload

| Concurrency | Cache off: output tok/s | Cache on: output tok/s | Throughput change | Cache off: p95 TTFT | Cache on: p95 TTFT |
|---:|---:|---:|---:|---:|---:|
| 1 | 98.6 | 95.1 | −3.5% | 91.1 ms | 65.9 ms |
| 8 | 521.6 | 695.4 | +33.3% | 664.4 ms | 131.7 ms |
| 32 | 966.6 | 2,214.5 | +129.1% | 2,645.5 ms | 197.3 ms |

At concurrency 32, the cache-enabled configuration delivered **2.29× output throughput**, **92.5% lower p95 TTFT**, and **56.3% lower p95 end-to-end latency**: approximately 4.26 seconds became 1.86 seconds.

The single-request case was more subtle. Response startup improved, but median TPOT increased from about 9.52 to 10.12 milliseconds. Generating the remaining tokens outweighed the smaller startup saving, and total throughput declined slightly. Faster TTFT did not imply faster completion.

### Unique-prefix workload

| Concurrency | Cache off: output tok/s | Cache on: output tok/s | Cache off: p95 TTFT | Cache on: p95 TTFT |
|---:|---:|---:|---:|---:|
| 1 | 98.7 | 92.5 | 90.9 ms | 98.2 ms |
| 8 | 521.1 | 499.0 | 666.8 ms | 686.9 ms |
| 32 | 962.7 | 925.0 | 2,652.7 ms | 2,732.3 ms |

With caching disabled, shared and unique workloads performed similarly. With caching enabled, the shared workload pulled ahead as concurrency increased. That pattern is consistent with prefix reuse reducing repeated prefill work and the contention associated with it.

The unique workload was about 4–6% slower in throughput in the cache-enabled configuration. Initially, it was tempting to describe this as cache-management overhead. The configuration comparison showed why that explanation would be premature.

## The configuration comparison changed the conclusion

The harness captured `/get_server_info` before each run. Comparing those snapshots exposed several differences:

| Resolved setting | Cache enabled | Cache disabled |
|---|---|---|
| `disable_radix_cache` | `False` | `True` |
| `attention_backend` | `triton` | `trtllm_mha` |
| `page_size` | 1 | 64 |
| `mamba_radix_cache_strategy` | `extra_buffer` | `auto` |
| `uses_mamba_radix_cache` | `True` | `False` |
| Effective maximum running requests per DP | 62 | 314 |
| Largest captured decode graph batch size | 62 | 314 |
| Server random seed | 323299915 | 588433170 |

Model path, context length, tensor parallelism, and configured memory fraction matched in the comparison. Both runs reported about 51.05 GiB for weights and 50.41 GiB for KV cache. Token capacity differed slightly, and graph allocations differed too.

The effective request limits were both above the highest tested concurrency, 32, so this experiment does not demonstrate that those caps caused the performance difference. The backend and page-size differences are nevertheless material confounders.

The precise result is therefore:

> For this synthetic warm shared-prefix workload at concurrency 32, SGLang's cache-enabled configuration achieved 2.29× output throughput relative to its cache-disabled configuration, with different automatically selected attention backends and page sizes.

That is narrower than claiming a 2.29× improvement caused solely by radix caching.

## Why a hybrid model needs an extra buffer

Full-attention layers preserve KV entries for prefix tokens. Linear-attention layers instead maintain recurrent state summarizing the processed sequence, together with associated convolution state. Reusing a prefix in a hybrid model requires preserving the relevant state across both layer types.

The recurrent state evolves as a request advances. If multiple requests branch from one cached prefix, the implementation must preserve the reusable prefix checkpoint while managing each request's subsequent state.

SGLang calls one strategy for this `extra_buffer`. It provides additional recurrent-state storage for checkpointing and updates, trading GPU memory for caching and scheduling flexibility. It is not a second copy of the model weights or a CPU-offload buffer. This describes its role; I did not trace every allocation or copy in the state-cache implementation.

In the installed configuration code, automatic strategy resolution considers overlap scheduling, paging, and backend support. Its `no_buffer` validation requires page size 1, disables overlap scheduling, and rejects TRT-LLM MHA. The extra-buffer path permits combinations that `no_buffer` cannot support.

## Why backend choice should be orthogonal—and where compatibility intervenes

At the API level, these settings express different choices:

- **Prefix caching:** whether previously computed prefix state may be reused.
- **Attention backend:** which kernel implementation executes an attention operation.

Ideally, I would choose each independently within the set of supported configurations. That separation makes behavior easier to reason about and experiments easier to control.

But implementations have physical requirements. Kernels consume particular page layouts, block tables, and state representations. A cache strategy may constrain how those structures are allocated and updated. In the inspected code, `no_buffer` requires page size 1, while the relevant TRT-LLM path requires a larger page size. Those requirements make some combinations incompatible.

So orthogonality is a useful design goal, not a guarantee that every combination is executable. A resolver still has to select a compatible combination. For an operator, the important questions are whether it honors explicit choices, exposes its resolved decisions, and explains why a default or fallback was selected.

Here, neither launch command specified a backend. The server was selecting defaults during startup, not switching attention implementations midway through a measured request.

## TRT-LLM kernels inside SGLang

`trtllm_mha` refers to TensorRT-LLM attention kernels exposed through FlashInfer. SGLang's [backend implementation](https://github.com/sgl-project/sglang/blob/main/python/sglang/srt/layers/attention/trtllm_mha_backend.py) integrates those operations; FlashInfer documents the [decode API](https://docs.flashinfer.ai/generated/flashinfer.decode.trtllm_batch_decode_with_kv_cache.html) and [prefill API](https://docs.flashinfer.ai/generated/flashinfer.prefill.trtllm_batch_context_with_kv_cache.html).

Selecting that backend does not hand the whole model to a serialized TensorRT engine. SGLang still orchestrates serving and model execution. Full-attention kernels can come from TRT-LLM while the separate `linear_attn_backend` remains Triton. Likewise, `attention_backend=triton` refers to GPU kernels, not NVIDIA Triton Inference Server.

## The unresolved part of backend selection

The installed Qwen hybrid override contained an explicit cache-dependent condition. Its policy can be expressed as the following pseudocode:

```python
backend = "triton"
blocked = (
    not extra_buffer_active
    and radix_cache_enabled
    and no_speculative_decoding
)
if general_default == "trtllm_mha" and not blocked:
    backend = "trtllm_mha"
```

The helper defining `extra_buffer_active` checks that radix caching is enabled and that the strategy is `extra_buffer` or `extra_buffer_lazy`.

This explains why backend selection can depend on the cache strategy. It does **not** fully explain my final cache-on configuration: that snapshot reports `extra_buffer`, which would allow TRT-LLM under the inspected rule, yet the backend is Triton.

One hypothesis is that backend selection saw an earlier strategy value, chose Triton, and was not revisited after extra buffering resolved. Another possibility is an additional override or a distinction between configuration views. I did not trace the resolver far enough to distinguish them, and I did not test an explicit TRT-LLM-plus-cache launch. This remains an unresolved configuration-selection observation, not a confirmed correctness or performance bug.

There is relevant upstream history. [SGLang issue #27418](https://github.com/sgl-project/sglang/issues/27418), now closed, discusses overlapping cache-strategy resolution paths, an earlier ordering problem, and Qwen's coupling between cache state and backend selection. It also describes the page-size incompatibility of `no_buffer` with TRT-LLM. That discussion supports investigating resolution order, but does not establish that my run reproduces the same historical issue.

## What these measurements establish

The experiment completed all 3,456 measured requests without failure, and the three repetitions were consistent. Shared-prefix requests benefited substantially under the cache-enabled serving configuration at higher concurrency. The saved configuration snapshots also showed that a one-flag command difference can expand into several runtime differences.

There are limits to the result. This was one GPU, one synthetic input/output length, three concurrency levels, and a prewarmed shared prefix. It did not measure model quality, a production traffic distribution, maximum sustainable arrival rate, or kernel-level timing. No Nsight trace was collected for this experiment, so the explanation about reduced prefill pressure is an interpretation of request-level measurements rather than a kernel-level attribution.

The model revision was not pinned in the launch command. Exact software versions, a checkpoint commit, and the resolved runtime configuration should accompany any strict reproduction. The source extracts describe the installed code inspected during this investigation; linked `main` branches can change.

I am leaving the backend-selection investigation open here. The useful lesson from this first pass is already concrete: **a launch command records the requested configuration; a benchmark needs the resolved configuration too.** Without that snapshot, I would have published an apparently clean caching speedup while overlooking an attention-backend change.
