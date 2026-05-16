---
name: telegram-call
description: Synthesize a spoken message and place an outgoing 1-on-1 Telegram voice call that plays it. Self-installs and self-authenticates on first use. Triggers: call me, phone call, voice call, urgent notification, call X about Y, tell X.
version: 0.4.2
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

The skill ships three tools so the LLM can keep concerns clean:

| Tool                     | Purpose                                          |
| ------------------------ | ------------------------------------------------ |
| `synthesize_speech`      | Turn text into an mp3 (Kokoro-FastAPI)           |
| `verify_telegram_code`   | Submit a Telegram verification code or 2FA pwd   |
| `make_call`              | Initiate auth (with `phone`) or place the call   |

### synthesize_speech

Turns a text message into an mp3 file using
[Kokoro-FastAPI](https://github.com/remsky/Kokoro-FastAPI) (expected to be
running locally at `http://localhost:8880`). Returns the absolute path of
the generated mp3 — feed that path back into `make_call` as `audio_path`.

**Parameters:**

- `text` (required): the message to speak. Write it as you'd say it out
  loud, not as a chat message.
- `voice` (optional): Kokoro voice name (e.g. `af_heart`, `af_bella`,
  `af_nova`, `am_michael`, `bf_emma`). Defaults to `af_heart` or
  `$KOKORO_VOICE`.
- `speed` (optional): speech rate multiplier. `1.0` = normal.

Output files live under `$OCTOS_WORK_DIR/telegram-call-tts/` and are cached
by `sha256(voice|speed|text)` — re-synthesizing the same line is free.

### verify_telegram_code

Submits a Telegram verification code (or 2FA cloud password) during the
one-time login flow. **Only call this after `make_call` returned a message
asking for a code or password.** The tool inspects internal state to decide
whether to verify a `code` or a `password`.

**Parameters:**

- `code` (provide when verification was requested): typically 5 digits.
- `password` (provide when 2FA was requested): the user's cloud password.
- `resend` (boolean, optional): set to `true` to ask Telegram to redeliver
  the current code via the next channel — see *Code didn't arrive?* below.

If Telegram rejects the value as invalid (typo), the auth state is preserved
so you can retry just the code/password — do **not** restart the auth flow
or call `make_call` with `phone` again, that would invalidate the code that
was already sent.

#### Code didn't arrive?

By default Telegram sends the login code as an in-app message from the
official "Telegram" service-account chat to your other logged-in Telegram
sessions (phone app, Telegram Desktop, etc.). It does **not** SMS you
unless you don't have any other session reachable, and even then there can
be carrier delays. The skill's success message tells you which channel
Telegram chose (`an in-app Telegram message` / `SMS` / `an automated voice
call` / …) and which channel a resend would escalate to.

If the user truly received nothing:

```
verify_telegram_code({resend: true})
```

This calls `auth.resendCode` under the hood, cycling delivery typically
through `APP → SMS → CALL`. The previous code is invalidated; only accept
the freshly-arriving one.

If you instead see a flood-wait message (Telegram is rate-limiting code
requests for that number), wait the indicated duration before retrying —
sending another code immediately won't deliver and will only push the
flood-wait window further out.

### make_call

Initiates the auth flow (when `phone` is supplied) or places the actual
Telegram call (when authenticated). Same tool, two phases.

> **Two phone numbers, two people.** The skill deals with two distinct
> Telegram identities:
>
> - `phone` — your **own** Telegram account, the one **placing** the call.
>   Used only during the one-time login.
> - `target` — the **recipient**, the one being called.
>
> They are not interchangeable. `phone` is your number; `target` is theirs.
> The skill refuses to send a verification code if `phone` and `target`
> resolve to the same digits, on the assumption the agent has confused
> the two; you'll get a `success: false` envelope explaining the mistake.

**Parameters:**

- `target` (effectively required for actual calls): the **recipient** —
  `@username`, numeric `user_id`, or `+phone_number` (phone numbers only
  resolve if that user is already in your Telegram contacts).
- `audio_path` (required): absolute path to an audio file. Any format
  `ffmpeg` can decode is accepted; usually the path returned by
  `synthesize_speech`.
- `message` (optional): human-readable description surfaced in the result.
- `phone` (one-time, first-login only): the **caller's** own Telegram
  account number, with international country code, e.g. `+447700900000`.
  Provide this on the very first authentication step to receive a
  verification code. **Do not include `phone` again after a code has been
  sent** — use `verify_telegram_code` instead. Do not include it on normal
  calls once authenticated.

A missing parameter always produces a `success: false` envelope instructing
the agent what to ask for next, so nothing silently does nothing.

## Typical agent flow

User: "call @alice about a possible gas leak in her house"

1. **Synthesize the spoken message:**
   Agent → `synthesize_speech({text: "Hi Alice, this is an automated
   alert from Keith. There may be a gas leak in your house. Please check
   immediately and call emergency services if you smell gas."})`
   Skill returns the path to a cached mp3.

2. **(First time only) Authenticate Telegram:**
   Agent → `make_call({target: "@alice", audio_path: "/.../abc.mp3"})`
   Skill returns: *"Telegram is not authenticated yet. Ask the user for
   the phone number of the Telegram account that should PLACE the call
   (this is the user's OWN account, NOT the recipient's number)…"*
   Agent → user → user replies `+447700900000` (their own number).
   Agent → `make_call({target, audio_path, phone: "+447700900000"})`
   Skill calls `auth.sendCode` and returns: *"Telegram has issued a
   verification code for +447700900000 … Telegram delivers this as an
   in-app message from the official 'Telegram' service account to the
   user's existing logged-in sessions FIRST; SMS is a fallback. Tell the
   user to check the 'Telegram' chat in their app for a 5-digit code,
   then pass it to `verify_telegram_code` as `code`."*
   Agent → user → user reads the code from their Telegram app → replies.
   Agent → `verify_telegram_code({code: "12345"})`
   On success: *"Telegram sign-in complete. The agent can now call
   `make_call` to place the actual call."*
   (If Telegram has 2FA enabled, the same tool then asks for `password`.)

3. **Place the call:**
   Agent → `make_call({target: "@alice", audio_path: "/.../abc.mp3",
   message: "gas leak warning"})`
   Skill places the call, audio plays, hangs up, returns *"Call completed
   (stream_finished): gas leak warning"*.

On all subsequent runs, only step 1 + step 3 are needed.

## Setup

There is no setup script and no `pip install` to run.

1. Get a Telegram API ID + hash from <https://my.telegram.org/apps> and set
   them in your shell / Octos profile:

   - `TELEGRAM_API_ID`: numeric API ID
   - `TELEGRAM_API_HASH`: API hash string

2. Run a Kokoro-FastAPI server locally — see
   <https://github.com/remsky/Kokoro-FastAPI>. The skill expects it on
   `http://localhost:8880` by default; set `KOKORO_URL` to override.

3. Install the skill:

   ```sh
   octos skills install <user>/telegram-call
   ```

4. Just use it. The first time `make_call` runs the skill self-installs
   its Python dependencies into a private venv at `<install-dir>/.venv`
   (~30-90s, network required) and then proceeds with the auth flow
   above.

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

## Resetting auth

If something gets stuck and you want to start the login over from scratch,
delete the auth artifacts under `$OCTOS_DATA_DIR/telegram-call/`:

```sh
rm -f "$OCTOS_DATA_DIR/telegram-call/auth_state.json" \
      "$OCTOS_DATA_DIR/telegram-call/auth_complete" \
      "$OCTOS_DATA_DIR/telegram-call/telegram-call.session"
```

Next call to `make_call` will start the phone → code → optional 2FA flow
again from the beginning.
