# Super Voice — OpenAI backend

[![coverage](https://img.shields.io/endpoint?url=https://super-libre.github.io/super-voice-openai/coverage.json)](https://super-libre.github.io/super-voice-openai/)

[OpenAI](https://platform.openai.com/docs/guides/speech-to-text)'s hosted
speech-to-text, `whisper-1`, `gpt-4o-transcribe`, `gpt-4o-mini-transcribe` and
the streaming `gpt-live-transcribe`, as a backend for
[Super Voice](https://github.com/jorge-menjivar/super-voice). The audio goes to
OpenAI, or to any OpenAI-compatible server you point it at, so it needs an API
key and a connection rather than a GPU. It hears your dictation context: your
vocabulary and prompt go to the model with every recording.

## What it is

The Super Voice daemon does not compile model inference in-tree. It discovers
backends and drives each one over a `/v1` HTTP contract. This is one such
backend: a sandboxed WebAssembly component that answers `POST /v1/transcribe`
and streams realtime sessions over a WebSocket, with no network access but the
endpoint it is configured for.

## License

GPL-3.0-only. See `LICENSE`.
