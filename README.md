# Bimo-7B-CyberAgent on one M5 Ultra (TensorFold native, the bf16 release)

Serves [Bimo-7B-CyberAgent](https://huggingface.co/iBimo-o/Bimo-7B-CyberAgent), iBimo-o's defensive-security
fine-tune of Qwen2.5-Coder-7B-Instruct (Qwen2 architecture, 7.6B parameters, a 32,768-token YaRN window), from its
bf16 release as published, with TensorFold's native Metal engine, as an OpenAI-compatible server with tool calls,
structured output, the prompt cache and two streams. Nothing is converted: the weights are 15.2 GB, which any M5 Mac
holds as released, so there was no reason for a lossy quantization. Use the model only on systems and code you are
authorized to assess (the model card says the same).

The engine's `llama` Metal family gains Qwen2 checkpoints with this recipe (q, k and v biases and static YaRN at
Hugging Face's rounding points): branch `llama-qwen2` on
[BobClawblaw/TensorFold](https://github.com/BobClawblaw/TensorFold/tree/llama-qwen2), pinned in `recipe.toml`, on
its way to [ashhart/TensorFold](https://github.com/ashhart/TensorFold) as a pull request.

## Quick start

Needs an M5 Mac with the weights' 15 GiB plus a stream's caches free (the M5's tensor units; M1-M4 are refused by
name), Python 3.11+, git, `hf` and [Zig 0.17.0](https://ziglang.org/download/).

```bash
git clone https://github.com/BobClawblaw/TensorFold ~/github/TensorFold
hf download iBimo-o/Bimo-7B-CyberAgent --revision 9c1954d83d1768720912a8ebdff5db1fdcfb4f87 --local-dir ~/models/Bimo-7B-CyberAgent
cp local.example.toml local.toml      # then set zig (and paths, if yours differ)
python3 -m mac build                  # the pinned engine and libtfgrammar, under ~/.cache/bimo-7b-cyberagent
python3 -m mac up                     # http://127.0.0.1:8896/v1, model "bimo"
python3 -m mac chat "Write a Python function that merges overlapping intervals."
```

`python3 -m mac down|status|logs|command|validate` stop, inspect and check it. To serve the LAN on login:
`python3 -m mac launchd --port 8888` writes `~/Library/LaunchAgents/com.bimo.bimo-7b-cyberagent.plist` (not loaded),
then `launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.bimo.bimo-7b-cyberagent.plist`.

## Settings

`recipe.toml` holds the defaults; `local.toml` overrides them for one Mac, and an environment variable of the same
name in upper case overrides both (`PORT=8888 HOST=0.0.0.0 python3 -m mac up`).

| setting | default | |
|---|---|---|
| `context` | 32768 | a stream's window, the release's `max_position_embeddings` (YaRN 4x over its 8,192 trained window); more is refused |
| `parallel` | 2 | streams decoded together, each reply equal to its run alone |
| `max_tokens` | 8192 | a reply's default ceiling |
| `temperature`, `top_p`, `top_k` | 0.7, 0.8, 20 | the release's `generation_config.json` for requests that set none (its `repetition_penalty` 1.1 is not applied: the family has no penalties) |
| `prompt_cache_gib` | 4 | kept prompt states between turns; they give way to streams |
| `load_limit_gb` | 40 | weights, scratch and every stream's caches together (`LLAMA_LOAD_LIMIT_GB`; 0: 75% of RAM) |
| `drafts` | false | the lane core's copy drafts, off: they land 1.05 tokens a round on this model and cost 6% (replies are identical either way; no drafter model is offered) |

**Memory.** The weights take 14.7 GiB wired. Every one of the 28 layers keeps bf16 keys and values, 56 KiB a token:
1.75 GiB a stream at 32,768. 40 GB holds two full streams and the kept prompts with room to spare.

**Thinking.** None: the published template has no think block, so every reply is `content` (the server is told so).

**Tool calls.** The published template renders no `tools` field. Either describe the tools in the system message in
the Hermes form the model was trained on (`<tool_call>{"name": ..., "arguments": {...}}</tool_call>`, each tool's
name and parameters) and pass the same tools in the request's `tools`, or set `chat_template =
"templates/qwen25-coder-tools.jinja"`: the base model's own template (Qwen/Qwen2.5-Coder-7B-Instruct, revision
c03e6d35) with Bimo's published identity text in place of Qwen's, which renders `tools` in that form and gives the
same prompt as the published template when a request has none. Either way the model's `<tool_call>` JSON is parsed
into `tool_calls` and `tool_choice: required` is honoured; the harness's tool cases pass 4/5 with each (the `auto`
case with two tools gets a fabricated `<tool_response>` instead of a call: the model's habit, the same with both
templates). A request with `tools` alone on the published template gets prose.

## What works, and how it was checked

All on one M5 Ultra (256 GB, macOS 27), checkpoint revision 9c1954d8, engine commit 7662e079 (10 Nov 2026); the
pinned 4cc353f0 adds that commit's engine doc and changes no code.

- **Text, greedy and sampled.** Against a float64 run of the release's own arithmetic
  (`tools/zig/llama_f64_reference.py`, the biases and YaRN at HF's rounding points), 192 teacher-forced positions
  over three chats: 190 equal, 2 near-ties (the repository's rule); HF's own bf16 run (transformers 5.22.2) gets
  189 and 3 on the same positions. Greedy, 64 tokens: two chats equal HF's token for token, the third parts at a
  one-bf16-step tie at step 36. One-row steps and 16-row windows give the same logits bit for bit. Seeded sampling
  repeats (9/9, `evidence/bf16-quick`). The suite's thinking cases (0/10) and one of its three max-tokens cases do
  not apply: the model has no think block, and it refuses the suite's essay prompt as off-topic in 14 tokens
  (limits of 1 and 7 tokens are honoured exactly).
- **Long context.** A passphrase comes back from 4k, 8k, 16k and 20k-token prompts and from 90% of a 27,633-token
  one; at 10% and 50% of 27.6k the model answers with another number, and so does HF's own run of the release on the
  same prompt (its behaviour, not the engine's). A 27,633-token cold prompt serves in 5.7 s.
- **Tools and structured output.** The harness's structured check (json_object, json_schema, regex, choice,
  grammar) 18/18 and drafted == plain 12/12; its tool cases (auto, required, named, thinking, streamed) 4/5 with the
  published template and 4/5 with the opt-in one ([above](#settings); `evidence/bf16-quick`,
  `evidence/tools-template-quick`).
- **Prompt cache.** The harness's third turn resumed from 5,493 of its 5,583 tokens equals the same request after a
  restart and starts in 0.13 s against 1.06 s cold (`evidence/bf16-speed`); a 13,182-token second turn resumed from
  13,147 kept tokens in the recipe's own check, same reply. Turns shorter than a 2,048-token prompt chunk keep no
  state (the cache's grid).
- **Concurrency.** The suite's soak (3 chats of 20 turns beside a decoding stream) 60/60 and 12 concurrent replies
  each equal the request alone, streaming 4/4, multi-turn recall 8/8, stop 3/3, copy 2/2 (`evidence/bf16-quick`);
  two concurrent greedy replies equal their solo runs in the recipe's own check.
- **Drafts.** The checkpoint has no draft head and no drafter is checked with it (`--drafter` is refused by name).
  The lane core's copy drafts land 1.05 tokens a round on the ruler and cost 6% (49 against 52 tokens/s on the
  suite's check), so the recipe serves without them; with them, replies equal the undrafted ones 12/12
  (`evidence/bf16-quick`).
- **Vision.** Not applicable: Bimo-7B-CyberAgent is text-only.

## Measured

Through [recipe-harness](https://github.com/BobClawblaw/recipe-harness) (target `bimo-7b-m5`; the qwen Spark
recipe's ruler), greedy, 256-token replies, engine 7662e079, copy drafts off, no other model loaded; the raw results
are in `evidence/`.

| streams | prose | code | structured | list | first token |
|---:|---:|---:|---:|---:|---:|
| 1 | 55.7 | 44.4 | 55.6 | 55.9 | 0.07-0.10 s |
| 2 | 88.6 | 103.8 | 113.3 | 113.4 | 0.11-0.14 s |

Tokens a second, all streams together (`evidence/bf16-speed`, the fresh ruler; the same ruler after the chat checks
ran 42-49 on one stream, and the quick plan's rulers 44-64). One stream reads the 14.7 GiB of weights each step
(19.0 ms a round); two streams read them once for both rows (19.2 ms a round). With the lane core's copy drafts on
the rounds land 1.05 tokens and run 20 ms (`evidence/bf16-quick`).

| cold prompt | 2,288 | 8,838 | 17,567 | 27,633 tokens |
|---|---:|---:|---:|---:|
| tokens a second | 5,974 | 6,163 | 5,654 | 4,850 |
| time to first token | 0.38 s | 1.43 s | 3.11 s | 5.7 s |

A fresh server, each prompt starting with words no earlier one did (`evidence/bf16-prefill-32k`, measured with copy
drafts on, which do not touch the prompt pass); the 27,633-token prompt is the recipe's own needle check. The
plan's largest prompt renders to 32,860 tokens with this tokenizer, over the window, and is refused by name. A
multi-turn chat's third turn, resumed from 5,493 of its 5,583 tokens, starts in 0.13 s against 1.06 s cold
(`evidence/bf16-speed`).

## Performance options

Each keeps the replies' bits; the numbers are this Mac's (`evidence/`).

- **Streams (`parallel`).** One stream decodes at 44-56 tokens/s, bound by reading the weights each step. Two
  streams give 89-113 tokens/s in all (19.2 ms a round against 19.0), each reply equal to its run alone; the first
  token moves from 0.07-0.10 s to 0.11-0.14 s.
- **Weights.** The release's bf16 is exact and is the only form served; no quantized conversion is offered.
- **Context and memory (`context`, `load_limit_gb`).** 56 KiB a token; the 32,768-token window is the release's
  and the recipe's maximum. Cold prompts run 5.7-6.2k tokens/s to 17.5k tokens and 4.9k at 27.6k.
- **Prompt cache (`prompt_cache_gib`).** A resumed turn starts in 0.13 s against 1.06 s cold, with the same reply;
  kept states give way to streams that need the room.
- **Not offered.** No int8 or fp8 cache, no fast-math, no reduced-precision sums, no drafter model.

## Notes

- No quality suite was run on this recipe; the checks above are exactness against the release's own arithmetic,
  the harness's serving checks and its speed rulers.
- The engine's doc for the family's Qwen2 path, with the kernel contract's summary, is
  `docs/recipes/bimo-7b-cyberagent-metal.md` in the pinned TensorFold branch.

## License

The recipe is under the [MIT License](LICENSE). Bimo-7B-CyberAgent is published under Apache 2.0 on
[Hugging Face](https://huggingface.co/iBimo-o/Bimo-7B-CyberAgent).
