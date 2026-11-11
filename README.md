# Super Voice — OpenAI backend

[![coverage](https://img.shields.io/endpoint?url=https://super-libre.github.io/super-voice-openai/coverage.json)](https://super-libre.github.io/super-voice-openai/)

A speech-to-text backend for **[Super Voice](https://github.com/jorge-menjivar/super-voice)**.
It proxies audio to [OpenAI](https://platform.openai.com/docs/guides/speech-to-text)'s
hosted transcription API, so transcription runs in the cloud rather than on your machine.

Super Voice is an on-device speech engine. It doesn't ship any models of its own —
it loads **backends** like this one at runtime. This repo packages OpenAI as one of those
backends, shipped as a sandboxed **WASM component** that serves both transports: batch
transcription over `wasi:http`, and streaming transcription over a WebSocket.

## Using it

You don't run this directly. Super Voice discovers it through its backend registry,
downloads the prebuilt `.wasm` from this repo's GitHub release, and runs it in-process in
a WASM sandbox whose only network egress is the allowlisted OpenAI API. To use it,
install Super Voice, enable OpenAI from the app, and add your **OpenAI API key** —
see the [Super Voice docs](https://github.com/jorge-menjivar/super-voice).

## Models

Chosen by `name` when Super Voice loads the backend. These are **online** models: they send
audio to OpenAI and need an OpenAI API key (set in the app); no local GPU or weights are
involved.

| Model (`name`)           | Provider | Type     | Languages | Requires                       |
| ------------------------ | -------- | -------- | --------- | ------------------------------ |
| `whisper-1`              | openai   | online   | en        | OpenAI API key                 |
| `gpt-4o-transcribe`      | openai   | online   | en        | OpenAI API key                 |
| `gpt-4o-mini-transcribe` | openai   | online   | en        | OpenAI API key                 |
| `other`                  | openai   | online   | en        | **Custom model name**          |
| `gpt-live-transcribe`    | openai   | realtime | en        | OpenAI API key                 |
| `other-realtime`         | openai   | realtime | en        | **Custom model name**          |

`other` is a placeholder for a model this backend does not list — one served by
an OpenAI-compatible endpoint. Set **Custom model name** to the name that server
expects (e.g. `Systran/faster-whisper-large-v3`) and it is sent instead of
`other`; point **API base URL** at the server, including the API version
(`http://localhost:8000/v1`). Selecting `other` without a custom model name is an
error rather than a request for a model called `other`. Both settings are ignored
by the listed OpenAI models. `other-realtime` is the same placeholder on the
realtime transport, and reads the same **Custom model name**.

The **OpenAI API key** is optional. It is required to reach `api.openai.com` —
requests there are refused without one, since they can only come back 401 — but a
self-hosted or gateway endpoint set through **API base URL** is called with no
`Authorization` header at all when no key is set. Set a key and it is sent as
`Bearer` to whatever endpoint is configured.

## Dictation contexts

This backend accepts a [dictation
context](https://github.com/jorge-menjivar/super-voice/blob/main/docs/protocol/endpoints/v1/context.md)
— the words you are dictating, so the model hears them instead of guessing at
them. Pick one in the app; there is nothing to configure here.

Super Voice sends the context's two halves as `x-voice-prompt` and
`x-voice-vocabulary` (Super STT as `x-stt-prompt` and `x-stt-vocabulary`, which
this backend reads too), and this backend folds them into the single `prompt` field
OpenAI's transcription API takes. The **vocabulary goes first**, because it is
the half `whisper-1` can use at all: OpenAI's guide describes the parameter as a
way "to pass a dictionary of the correct spellings" and says in the same breath
that "Whisper doesn't follow instructions like a general-purpose text model".
The prompt sentence follows it, for `gpt-4o-transcribe` and
`gpt-4o-mini-transcribe`, which do follow instructions.

So a context named "Coding" with the terms `main branch`, `rebase`, `kubectl`
and the prompt "I dictate code." is sent as:

```
main branch, rebase, kubectl. I dictate code.
```

`whisper-1` documents a **224-token** ceiling on the prompt, and OpenAI does not
say what happens past it — so keep a vocabulary you care about short rather than
relying on a truncation rule that is not written down.

### On the realtime models

The realtime API has a field for each half, so nothing is folded together there:
the prompt goes to `prompt` ("describe the recording or its setting") and the
terms go to `keywords` ("product names, acronyms, and other literal terms"), as
a real list. Keywords are hints, not required output.

`keywords` is sent only to `gpt-live-transcribe`, which is the model OpenAI
documents it against. `other-realtime` points at a third-party server that has
most likely never heard of the field, and an unrecognized field there is not
ignored — it is a **rejected session update**, which ends the recording before a
word of it is transcribed. So that model gets the terms folded into `prompt`
instead, the same way the batch path does.

For the same reason, both fields are sanitized before they are sent. The
realtime API refuses `<`, `>`, carriage returns and line feeds, and a context
prompt is typed into a multi-line box — so line breaks become spaces and the
angle brackets are dropped. Without that, a perfectly ordinary two-line prompt
would break the take.

## Realtime

The `realtime` models stream instead of uploading a finished recording. Super Voice hands
the component a live consumer WebSocket, and the component bridges it to OpenAI's realtime
transcription API (`wss://api.openai.com/v1/realtime?intent=transcription`) — or to the
same endpoint on whatever server **API base URL** names. Partial transcripts come back as
`preview` frames and the final one as `done`.

Two details are worth knowing:

- **Sample rate.** OpenAI's realtime API accepts 24 kHz PCM only, and Super Voice streams
  at whatever rate the session declares (16 kHz, typically), so the component resamples on
  the way upstream. Nothing to configure.
- **Previews arrive late.** The host does not yet implement `wasi:io/poll` for WebSocket
  resources, so the component cannot wait on the consumer and OpenAI at the same time. It
  runs half-duplex: all audio goes up first, then the transcript events come back. The
  partials still arrive, but in a burst near the end rather than as you speak.

## What's in here

A small, self-contained Rust component (`src/lib.rs` plus `src/component/`) that speaks
the Super Voice backend protocol — the `/v1` contract over `wasi:http`, and the
`super-stt:realtime` WebSocket session — and forwards audio to OpenAI. It shares no code
with the Super Voice project; `wit/` is a vendored copy of the protocol's WIT. The pure
audio, request-shaping, and realtime-payload helpers are unit-tested natively; the
component as a whole is exercised by a wasmtime harness under `tests/` that loads the
built `.wasm` and drives both transports against mock upstreams.

## Building from source

Most people never need to — Super Voice downloads prebuilt releases. For development
(requires [`just`](https://github.com/casey/just) and the `wasm32-wasip2` target):

```bash
rustup target add wasm32-wasip2
just build-component   # builds target/wasm32-wasip2/release/super_voice_backend_openai.wasm
just ci                # format, lint, build, and test
```

## License

GPL-3.0-only.
