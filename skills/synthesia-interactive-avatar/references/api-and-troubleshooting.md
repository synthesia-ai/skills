# API Reference & Troubleshooting — Synthesia Interactive Avatar

All symbols live in `livekit.plugins.synthesia`.

## `synthesia.AvatarSession`

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `avatar_config` | `AvatarConfig` | required | Avatar identity and rendering options. |
| `api_key` | `str \| None` | `None` | Synthesia workspace API key. Falls back to `SYNTHESIA_API_KEY`. |
| `api_url` | `str \| None` | `None` | API base URL. Falls back to `SYNTHESIA_API_URL`, then `https://developers.synthesia.io`. |
| `join_timeout` | `float` | `30.0` | Seconds to wait for the avatar to join and publish before raising `SynthesiaTimeoutError`. |

## `synthesia.AvatarConfig`

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `avatar_ids` | `Sequence[str]` | required | One to five gallery ids of avatars available to the workspace. The first is the active avatar; the rest are precomputed so `swap_avatar()` can switch to them. Default avatar: **Kenji**, `7572faa9-15da-400d-8227-ef1ab8932523`. Inaccessible ids raise `UnknownAvatarError` at `start()`; a bare string instead of a list raises `SynthesiaError`. |

## Methods

| Method | Notes |
| --- | --- |
| `await avatar.start(session, room, *, livekit_url=None, livekit_api_key=None, livekit_api_secret=None)` | Mounts the avatar into `room`, launches the worker, wires `session.output.audio`. Returns once the avatar has joined and published video. **Call before `AgentSession.start()`.** LiveKit credentials fall back to `LIVEKIT_URL` / `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET`. |
| `await avatar.swap_avatar(avatar_id, *, timeout=15.0)` | Switches the rendered avatar mid-session to another id from `avatar_ids`, or back to the first id with `"default"`. Returns the now-active id. Requires a started session (`SynthesiaError` otherwise); an id not in `avatar_ids` raises `UnknownAvatarError`; RPC failure raises `SynthesiaConnectionError`. |
| `await avatar.aclose()` | Cooperative shutdown; restores audio routing. Called automatically on room disconnect. |

## Events

| Event | Fires when |
| --- | --- |
| `session_ended` | The room ended cleanly. |
| `error` | The avatar track dropped unexpectedly mid-session (carries a `SynthesiaConnectionError`). |

```python
avatar.on("session_ended", lambda: ...)
avatar.on("error", lambda exc: ...)
```

## Exceptions

| Exception | Meaning | Retryable |
| --- | --- | --- |
| `SynthesiaError` | Base class; also raised for missing credentials. | no |
| `SynthesiaAuthError` | Invalid/expired key or workspace without Interactive Avatar access (HTTP 401/403). | no |
| `InvalidRoomTokenError` | The minted room token can't produce a joined session — malformed, or missing the attribute naming the agent the avatar publishes for (`invalid_token`). Distinct from `SynthesiaAuthError`: the Synthesia key is fine. | no |
| `LiveKitCredentialsRejectedError` | Token is well formed but the LiveKit project rejected it (`invalid_livekit_credentials`) — check the LiveKit key/secret, not the Synthesia key. | no |
| `InvalidSessionRequestError` | The backend rejected the session request payload (`validation_error` / `bad_request`). | no |
| `UnknownAvatarError` | An id in `avatar_ids` not accessible to the workspace (404 / `unknown_avatar` / `avatar_not_accessible`), or a `swap_avatar()` target that wasn't in `avatar_ids`. | no |
| `QuotaExceededError` | Minute or concurrent-session cap hit (HTTP 402). | no |
| `RateLimitedError` | Throttled (HTTP 429); carries `retry_after` (seconds) when provided. | yes |
| `SynthesiaTimeoutError` | Avatar didn't join within `join_timeout`. | yes |
| `SynthesiaConnectionError` | Transport failure after retries, or avatar dropped mid-session. | yes |

Retry guidance: back off and retry only the retryable three. Honour `retry_after` on 429s. For timeouts, raise `join_timeout` before retrying — cold starts are the usual cause. Never wrap auth/quota/unknown-avatar/token/validation errors in retry loops; surface them to the user with the fix.

## Troubleshooting matrix

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `SynthesiaError: a Synthesia API key is required` | No `SYNTHESIA_API_KEY` and no `api_key=`. | Set the env var or pass `api_key=`. |
| `SynthesiaError: LiveKit url, API key, and API secret are required` | Missing `LIVEKIT_*` at `start()`. | Set the env vars or pass them to `start()`. |
| `SynthesiaAuthError` | Key invalid/expired, or workspace lacks Interactive Avatar access. | Confirm the key; confirm workspace access with Synthesia support. |
| `SynthesiaError: avatar_ids must be a list of ids, not a single string` | `AvatarConfig(avatar_ids="<id>")` with a bare string. | Wrap it in a list: `avatar_ids=["<id>"]`. |
| `SynthesiaError: livekit_url ... is not a ws:// or wss:// URL` | Malformed `LIVEKIT_URL` that can't be normalized to `ws(s)://`. | Use the project's `wss://` URL (`https://` is auto-normalized and fine). |
| `InvalidRoomTokenError` at `start()` | Synthesia rejected the room token the plugin minted — malformed or missing the agent attribute. Not a Synthesia-key problem. | Connect the room (local participant needs an identity) before `avatar.start()`; if that's already the case, upgrade the plugin. |
| `LiveKitCredentialsRejectedError` at `start()` | LiveKit key/secret don't match the project at `LIVEKIT_URL` — the token mints fine locally, then LiveKit refuses the avatar's join. | Fix `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET` to the project `LIVEKIT_URL` points at. The Synthesia key is not the problem. |
| `InvalidSessionRequestError` | Backend rejected the session request payload. | Check `avatar_ids` are the raw gallery ids from Synthesia; the exception message and `body` name the offending field. |
| `UnknownAvatarError` at `start()` | Avatar id not in the workspace gallery / not on the tier. | Use Kenji's id (the default); otherwise check access with Synthesia support. |
| `UnknownAvatarError: ... not in initial list of avatar_ids` on `swap_avatar()` | Target id wasn't passed to `AvatarConfig`. | Include every swappable id (max 5) in `avatar_ids` up front; adding one requires a new session. |
| `SynthesiaError: swap_avatar() requires a started avatar session` | `swap_avatar()` called before `start()` or during/after shutdown. | Only swap while the session is live. |
| `QuotaExceededError` | Minute or concurrency cap hit. | Reduce concurrent sessions or ask Synthesia support about limits. |
| `SynthesiaTimeoutError` | Avatar didn't join in time (cold start or network). | Raise `join_timeout`; retry; confirm the worker is reachable. |
| Avatar never appears, **no error** | Terminal `console` run mode (local mock room), or `start()` called after `session.start()`. | Run with `dev`/`connect` and test in a real room — e.g. the LiveKit Cloud **Agent Console** (dashboard → Agents → Console; the browser tool, not the terminal mode of the same name). Attach the avatar **before** starting the session. |
| Avatar joins but no lip-sync | Agent isn't producing TTS audio, or `session.output.audio` was reassigned after `start()`. | Verify the agent speaks at all (temporarily without the avatar); never reassign `output.audio` after attaching. |
| Avatar video looks low-res or blurry | Subscriber-side adaptive streaming downscaled the track to a small or hidden video element. | Create the room with `adaptiveStream: false`, or render the avatar large and call `publication.setVideoQuality(VideoQuality.HIGH)` on its track. Resolution is set by the hosted worker — there is no plugin-side quality knob. |
| Frontend connects but the agent/avatar never joins | Agent worker not running; frontend points at a different LiveKit project; or `agent_name` is set (explicit dispatch) but nothing dispatches the agent into the room. | Keep `python agent.py dev` running; use the same `LIVEKIT_URL` / API key / secret in both; dispatch via `RoomAgentDispatch` in the token's room config (see the Dispatch section in `greenfield-quickstart.md`). |
| Works locally, fails deployed | Secrets not present in the deploy environment. | Confirm all five env vars (`LIVEKIT_*` ×3, `SYNTHESIA_API_KEY`, provider keys) exist in the runtime. |
