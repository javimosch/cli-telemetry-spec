---
name: cli-telemetry
description: Add honest, non-identifying, opt-out usage telemetry to a CLI — disclosed on stderr, allow-listed fields, inspectable payload. Use when a tool's author has no idea whether anyone runs it.
---

# Add honest telemetry to a CLI

Implements [cli-telemetry-spec](https://github.com/javimosch/cli-telemetry-spec).

## When to use this

The author cannot tell "nobody uses this" from "it is broken before it runs".
Downloads, clones and stars all report zero in both cases.

## When NOT to use this

- The tool handles credentials, health, financial or otherwise regulated data —
  ship nothing, or make it opt-in (`--on`).
- You cannot name the decision a field would change. Then you do not need the
  field, and probably not the telemetry.

## Steps

1. **Switches first.** `<TOOL>_TELEMETRY=0`, `DO_NOT_TRACK=1`, a config flag, and
   CI detection — evaluated before any network code is reachable.
2. **Disclose on stderr, once, before the first send.** Four lines: what, where,
   how to stop it. Never stdout.
3. **Payload from the allow-list only**: tool, version, event
   (`install`|`run`|`error`), verb, os, arch, exit_class, optional random
   install_id, ts. Write the fields literally; do not serialise a struct.
4. **Send bounded and silent**: one POST, ≤2s timeout, no retry, after the
   user's output is written, failure ignored.
5. **`<tool> telemetry`** prints the REAL next payload from the same builder the
   sender uses, plus `enabled`, `reason`, `endpoint`, `disable`.
6. **Document it** in the guide and, in the open, in the README.

## Checks before you call it done

```sh
<TOOL>_TELEMETRY=0 <tool> <verb>     # no connect() syscall
DO_NOT_TRACK=1 <tool> telemetry      # enabled:false, reason names the switch
CI=1 <tool> telemetry                # enabled:false, reason "CI detected"
<tool> <verb> | jq .                 # stdout still pure JSON on a FIRST run
```

## Never

Hostname, username, paths, arguments, flag values, data, error text, env, or any
hash of those. No timers, no daemons, no per-request events, no retries, no
fourth event type.
