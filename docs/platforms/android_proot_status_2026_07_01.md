# Android/PRoot SGLang status - 2026-07-01

This note records the current state of the phone/proot SGLang experiment on Allan's Android device.

## Environment

- Repo: `watsoncsulahack/sglang-android`
- Local checkout: `/root/.openclaw/workspace/sglang-android`
- Current commit tested: `3e83e03184` (`Build proot source dependency wheels`)
- Python environment: `/root/.openclaw/workspace/.venvs/sglang-android-proot`
- Built SGLang wheel: `/root/.openclaw/workspace/artifacts/sglang-actions-28354656294/sglang-0.5.15.dev238+g3e83e0318-cp313-cp313-linux_aarch64.whl`
- Extra local ARM64 wheel: `/root/.openclaw/workspace/artifacts/sglang-actions-28354656294/outlines_core-0.1.26-cp313-cp313-linux_aarch64.whl`

Installed package highlights:

- `sglang @ .../sglang-0.5.15.dev238+g3e83e0318-cp313-cp313-linux_aarch64.whl`
- `torch==2.11.0`
- `transformers==5.8.1`
- `tokenizers==0.22.2`
- `xgrammar==0.2.1`
- `sglang-kernel==0.4.4`
- `pytest==9.1.1`

## What works

### Local wheel install and CLI import surface

The GitHub Actions ARM64 wheel installs in the proot Linux environment without local Rust/source builds. Previous smoke checks confirmed:

- `import sglang` works.
- `sglang.srt.grpc._core.cpython-313-aarch64-linux-gnu.so` is present.
- `python -m sglang.launch_server --help` runs.
- `sglang --help` runs.

Important caveat: the environment warns that Triton is unsupported on this platform and falls back to CPU-oriented paths. `torch.cuda.is_available()` is false.

### MiniCPM5 tool-call parser

SGLang's native MiniCPM5 XML tool-call parser is the strongest successful result.

Command:

```sh
source /root/.openclaw/workspace/.venvs/sglang-android-proot/bin/activate
python -m pytest -q test/registered/unit/function_call/test_minicpm5_detector.py
```

Result:

```text
14 passed, 3 warnings in 48.46s
```

The parser correctly handles:

- MiniCPM-style `<function name="..."><param name="...">...</param></function>` XML.
- Multiple calls interleaved with normal text.
- Non-string arguments such as arrays and booleans.
- CDATA/multiline text.
- Unknown tools, missing required arguments, duplicate params, and malformed XML fallback.
- Streaming chunks where a call should only emit after the closing `</function>`.

Raw log:

`/root/.openclaw/workspace/artifacts/sglang-minicpm5-toolcall-20260629/sglang_minicpm5_detector_pytest_20260701.log`

## What does not work yet

### SGLang live inference on the phone/proot stack

SGLang did not reach an inference-ready state for the local MiniCPM GGUF experiments within the guardrails used here.

Attempts and outcomes:

| Test | Input/model | Result |
|---|---|---|
| SGLang server, text GGUF | `MiniCPM5-1B-Q4_K_M.gguf` | Timed out before readiness. |
| SGLang offline `Engine`, text GGUF | `MiniCPM5-1B-Q4_K_M.gguf` | Timed out before generation. |
| SGLang server, MiniCPM-V GGUF multimodal | `MiniCPM-V-4_6-Thinking-Q4_K_M.gguf` | Timed out before readiness. |
| SGLang server, HF dummy MiniCPM-V 4.6 multimodal | `openbmb/MiniCPM-V-4.6 --load-format dummy` | Timed out before readiness. |

The logs consistently show platform fallback warnings before timing out:

- `Triton is not supported on current platform, roll back to CPU.`
- `Only CUDA, HIP and XPU support AWQ currently.`

Raw logs:

- `/root/.openclaw/workspace/artifacts/sglang-minicpm5-mm-20260629/minicpm5_gguf_text_server.log`
- `/root/.openclaw/workspace/artifacts/sglang-minicpm5-mm-20260629/minicpm5_gguf_text_server_unbuffered.log`
- `/root/.openclaw/workspace/artifacts/sglang-minicpm5-mm-20260629/minicpm5_gguf_engine_text.log`
- `/root/.openclaw/workspace/artifacts/sglang-minicpm5-mm-20260629/minicpmv46_gguf_multimodal_server.log`
- `/root/.openclaw/workspace/artifacts/sglang-minicpm5-mm-20260629/minicpmv46_hf_dummy_multimodal_server.log`

Interpretation: SGLang is installable on the proot ARM64 environment, but it is not currently a practical phone inference runtime here. The main missing piece is not the Python wheel anymore; it is a usable backend path for this device. On this Pixel/Mali setup, SGLang cannot use the Android Vulkan path that makes llama.cpp viable, and the CUDA/Triton-oriented acceleration stack is absent.

## llama.cpp comparison for MiniCPM

The Android Vulkan llama.cpp binary can run the local MiniCPM models, which makes it the practical local runtime today.

MiniCPM5 text-only:

| Runtime | Result |
|---|---|
| llama.cpp CPU/dev none smoke | Passed; prompt about 34.3 tok/s, generation about 8.7 tok/s. |
| llama.cpp Vulkan smoke | Passed; prompt/generation about 17.5 tok/s in the CLI smoke. |
| llama-bench CPU/dev none | `pp64 27.03 tok/s`, `tg32 2.34 tok/s`. |
| llama-bench Vulkan | `pp64 4.52 tok/s`, `tg32 5.28 tok/s`. |

MiniCPM-V 4.6:

| Test | Result |
|---|---|
| Text-only CPU | Passed, but very slow. |
| Text-only Vulkan | Passed and much faster. |
| Vision/mmproj | Blocked by binary support, not by model presence. The Android Vulkan binary rejected `minicpmv4_6` projector support; the proot `llama-mtmd-cli` accepts `--mmproj` but lacks the required `qwen35` model architecture support for the paired GGUF. |

## Tool-calling comparison

llama.cpp can run MiniCPM5, but it did not reliably produce MiniCPM5 tool-call XML in the tested prompts.

Observed llama.cpp behavior:

- Free-form XML prompt entered thinking mode and produced no parseable `<function ...>` call.
- `--reasoning off` produced a generic XML weather document, not the MiniCPM tool-call XML.
- JSON-schema prompting with the chat template failed because the template prepended `<think>...</think>` before the JSON grammar could match.
- A raw-ish JSON-schema prompt produced shaped JSON, but with bad arguments (`city` empty, `unit` truncated to `uni`).
- Parsing the generated-only portions with SGLang's MiniCPM5 parser found zero calls.

Interpretation: SGLang has the better semantic layer for MiniCPM5 tool calling because it includes a native parser. llama.cpp has the better local inference/runtime layer on this phone. A useful local tool-calling path would combine llama.cpp inference with a MiniCPM5-compatible external parser/prompt wrapper, or wait until SGLang can reach inference on a supported phone backend.

## Current verdict

SGLang on this Android/proot target is now tested far enough to call the current state:

1. The ARM64 wheel/build path works.
2. The MiniCPM5 tool-call parser works and is covered by the upstream unit test file.
3. Live SGLang inference did not become usable on the phone/proot stack under CPU fallback.
4. For actual on-device MiniCPM inference today, use llama.cpp/Vulkan.
5. For MiniCPM5 tool-call semantics, SGLang's parser is the correct reference implementation, but it needs either a working SGLang runtime backend or an adapter around llama.cpp outputs.

## Next useful work

- Do not spend more phone time trying default SGLang server flags against GGUF MiniCPM unless a new Android/Vulkan/CPU backend path appears.
- If tool calling is the goal, port or wrap `MiniCPM5Detector` outside SGLang's server runtime and run it against llama.cpp generation output.
- If multimodal MiniCPM-V is the goal, update or rebuild llama.cpp/mtmd with both `minicpmv4_6` projector support and the `qwen35` architecture support needed by the current GGUF pair.
