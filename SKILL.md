---
name: telegram-call
description: Synthesize a spoken message and place an outgoing 1-on-1 Telegram voice call that plays it. Self-installs and self-authenticates on first use. Triggers: call me, phone call, voice call, urgent notification, call X about Y, tell X.
version: 0.3.0
author: Keith Yeung
always: false
requires_bins: python3,ffmpeg
requires_env: TELEGRAM_API_ID,TELEGRAM_API_HASH
---

# Telegram Call

Synthesizes spoken audio from text via a local Kokoro-FastAPI server and
places an outgoing 1-on-1 Telegram voice call to a user, playing the audio
through ffmpeg/pytgcalls and hanging up — when the audio finishes, when the
recipient declines or is busy, or when a max-duration fallback fires.

Use this for urgent notifications that need immediate attention, e.g. "call
@alice about a possible gas leak".

## Tools

### synthesize_speech

Turns a text message into an mp3 file using
[Kokoro-FastAPI](https://github.com/remsky/Kokoro-FastAPI) (expected to be
running locally at `http://localhost:8880`). Returns the absolute path of
the generated mp3 — feed that path back into `make_call` as `audio_path`.

**Parameters:**

- `text` (required): the message to speak. Write it as you'd say it out
  loud, not as a chat message.
- `voice` (optional): Kokoro voice name (e.g. `af_heart`, `af_bella`,
  `am_michael`, `bf_emma`). Defaults to `af_heart` or `$KOKORO_VOICE`.
- `speed` (optional): speech rate multiplier. `1.0` = normal.

Output files live under `$OCTOS_WORK_DIR/telegram-call-tts/` and are cached
by `sha256(voice|speed|text)` — re-synthesizing the same line is free.

### make_call

Places the actual Telegram call.

**Parameters:**

- `target` (effectively required): `@username`, numeric `user_id`, or
  `+phone_number` (phone numbers only resolve if that user is already in
  your Telegram contacts).
- `audio_path` (required): absolute path to an audio file. Any format
  `ffmpeg` can decode is accepted; usually the path returned by
  `synthesize_speech`.
- `message` (optional): human-readable description, surfaced in the result.
- `phone`, `verification_code`, `password` (one-time): only used during
  the first-call authentication flow, see below.

Missing parameters always produce a `success: false` envelope instructing
the agent what to ask for next, so nothing silently does nothing.

## Typical agent flow

User: "call @alice about a possible gas leak in her house"

1. Agent → `synthesize_speech({text: "Hi Alice, this is an automated
   alert from Keith. There may be a gas leak in your house. Please check
   immediately and call emergency services if you smell gas."})`
2. Skill returns: *"Generated TTS audio at /…/abc123.mp3 (54321 bytes).
   Now call `make_call` with audio_path='/…/abc123.mp3' to play it."*
3. Agent → `make_call({target: "@alice",
   audio_path: "/…/abc123.mp3", message: "gas leak warning"})`
4. Skill places the call, audio plays, recipient hears the message,
   call ends, skill returns *"Call completed (stream_finished): gas
   leak warning"*.

## Setup

There is no setup script and no `pip install` to run.

1. Get a Telegram API ID + hash from <https://my.telegram.org/apps> and set
   them in your shell / Octos profile:

   - `TELEGRAM_API_ID`: numeric API ID
   - `TELEGRAM_API_HASH`: API hash string

2. Run a Kokoro-FastAPI server locally — see
   <https://github.com/remsky/Kokoro-FastAPI>. By default the skill expects
   it on `http://localhost:8880`; set `KOKORO_URL` to override.

3. Install the skill:

   ```sh
   octos skills install <user>/telegram-call
   ```

4. Just use it. The first time `make_call` runs the skill will:

   a. Create a private virtualenv at `<install-dir>/.venv` and install
      `pyrogram` / `py-tgcalls` / `tgcrypto` (~30-90s, network required).
   b. Detect that no Telegram session exists and walk the user through
      a one-time login via chat: `phone` → `verification_code` →
      optionally `password`. Each step returns `success: false` with the
      next thing to ask for; the agent relays the question to the user
      and passes the answer back as the named parameter.
   c. Place the actual call.

   Subsequent calls only need `target` and `audio_path`.

## Optional environment variables

- `KOKORO_URL` (default `http://localhost:8880`): base URL of the
  Kokoro-FastAPI server.
- `KOKORO_VOICE` (default `af_heart`): default voice for `synthesize_speech`.
- `KOKORO_SPEED` (default `1.0`): default speed multiplier.
- `KOKORO_TIMEOUT` (default `120`): seconds to wait for a TTS response.
- `TELEGRAM_CALL_RING_TIMEOUT` (default `30`): seconds to wait for the
  recipient to pick up.
- `TELEGRAM_CALL_MAX_DURATION` (default `90`): max seconds on the call
  after connection before force-hanging up.

## System dependencies

- `python3` >= 3.10 with the `venv` module available (on Debian/Ubuntu:
  `apt install python3-venv`).
- `ffmpeg` on `PATH` (used by pytgcalls to decode audio for the call).
- A running Kokoro-FastAPI server (only required for `synthesize_speech`;
  `make_call` works without it as long as you supply your own audio file).
