# cli-telemetry-spec — an honest usage signal for agent-first CLIs

A tiny, copy-pasteable convention for giving any command-line tool a **`telemetry`**
command and a minimal, disclosed, opt-out-able usage ping — so the author can tell
whether anyone is using the tool and where it breaks, without collecting anything
that identifies who.

It is deliberately small — one `POST` route, one CLI subcommand, one allow-list of
fields — so an agent can add it to a tool in a few minutes, and so a user can read
the whole contract in under a minute and decide.

```sh
myapp telemetry
# -> {"ok":true,"data":{"enabled":true,"endpoint":"https://t.example.com/e",
#                       "install_id":"7f3a…","next_payload":{"tool":"myapp","version":"1.4.0",
#                       "event":"run","verb":"sync","os":"linux","arch":"x86_64","exit_class":0},
#                       "disable":"MYAPP_TELEMETRY=0","last_sent":"2026-07-30T12:04:11Z"}}

myapp telemetry --off
# -> [telemetry] disabled; nothing further will be sent
```

## Why

**A CLI author is blind in a way a web developer never is.** There is no server
log, no session, no bounce rate. The signals that exist are worse than nothing
because they look like data:

- **Downloads ≠ usage.** A tool with a broken install command and a tool nobody
  wants both report zero.
- **Clones are mostly machines.** One project measured 308 clones from 162
  uniques against 5 unique page views in the same window — mirrors and CI, not
  people.
- **Stars measure interest**, at best, and months late.
- **Agent-operated tools have no human to ask.** The user is a subprocess. It
  will never file an issue, tweet, or answer a survey.

The gap this closes is specific: *did anyone actually run this, and did it work?*
A tool whose documented install path had never once succeeded would report the
same zero as a tool nobody wanted — and the author cannot tell those apart.

**And yet a CLI must not behave like a web page.** It runs inside other people's
infrastructure: in CI, in air-gapped networks, in containers, under agents,
against production data. The ordinary analytics SDK — silent, identifying,
blocking, retrying — is not acceptable here, and shipping one is how a tool loses
the trust that made someone try it. This spec is the smallest thing that answers
the author's question while being defensible to the person running it.

## The documents

| File | For | What it is |
|---|---|---|
| **[PROTOCOL.md](PROTOCOL.md)** | implementers | the normative contract (MUST/SHOULD): disclosure, off switches, the field allow-list, transport, the `telemetry` command |
| **[RECIPE.md](RECIPE.md)** | implementers | the 5-step how-to with reference snippets |
| **[llms.txt](llms.txt)** | agents | the machine-first entry point — fetch this first |
| **[event.schema.json](event.schema.json)** | everyone | JSON Schema for the event payload — the allow-list, enforced |
| **[SKILL.md](SKILL.md)** | agents | drop-in skill: "add honest telemetry to this CLI" |
| **[AGENTS.md](AGENTS.md)** | agents | coding guidelines for maintaining this spec |

## The contract in one breath

- **Disclose before you send.** On the first run, print a notice **on stderr**
  saying what is collected, where it goes, and how to turn it off. The first
  event MUST NOT leave the machine before that notice has been printed.
- **Three off switches, checked before any network code runs**: `<TOOL>_TELEMETRY=0`,
  the cross-vendor `DO_NOT_TRACK=1`, and a config file. **CI defaults to off.**
- **An allow-list, never a deny-list.** Tool, version, OS, arch, event, verb name,
  exit class, and a rotatable random install id. That is the whole list.
- **Never**: hostname, username, home directory, file paths, arguments, flag
  *values*, document contents, connection strings, environment, or a retained IP.
- **Never blocks, never fails, never retries.** Hard timeout, fire-and-forget, no
  effect on the exit code, and **nothing on stdout** — stdout belongs to the data
  contract ([cli-output-spec](https://github.com/javimosch/cli-output-spec)).
- **Inspectable.** `<tool> telemetry` prints the exact payload that would be sent.
  An agent — or a security reviewer — verifies the claim instead of trusting it.
- **Aggregate on receipt.** The collector stores counts, not a row per install,
  and does not log IPs.

Full details in [PROTOCOL.md](PROTOCOL.md).

## What it answers, and what it does not

It answers: *is anyone running this; on what OS; which verbs do they reach; where
do they fail; did they get past the install.* That is enough to know a tool is
alive and where it hurts.

It does not answer: who they are, what they store, whether they like it, or what
they would pay for. **If you need those, ask a human.** A convention that stayed
small enough to be defensible is worth more than one that answers everything and
gets disabled by the first person who reads it.

## The other four specs

This is the fifth of a set. The others:

1. **[cli-output-spec](https://github.com/javimosch/cli-output-spec)** — the output contract
2. **[cli-guide-spec](https://github.com/javimosch/cli-guide-spec)** — the embedded mental model
3. **[cli-feedback-spec](https://github.com/javimosch/cli-feedback-spec)** — the dual-write relay
4. **[cli-update-spec](https://github.com/javimosch/cli-update-spec)** — content-hash self-update

Telemetry is the last one for a reason: it is the only one that sends anything
*out*, so it is the only one that has to earn its place. Adopt the other four
first.

MIT.
