# RECIPE — add honest telemetry to a CLI in five steps

Budget: about 30 minutes. Read [PROTOCOL.md](PROTOCOL.md) first; this is the
how, not the what.

Before you start, answer one question honestly: **what decision will this
change?** "Is anyone running it" and "where do they get stuck" are decisions.
"It would be nice to know" is not, and a field added for that reason is the one
that ends up in a security review.

---

## Step 1 — decide the switches, and check them first

The check must happen before any network code exists in the call path, so that
"disabled" is structurally impossible to get wrong.

```sh
tel_enabled() {
  [ -n "${DO_NOT_TRACK:-}" ] && [ "$DO_NOT_TRACK" != "0" ] && return 1
  case "${MYAPP_TELEMETRY:-}" in 0|false|off|no) return 1 ;; esac
  [ -f "$CFG/telemetry-off" ] && return 1
  # CI is not a user; counting it inflates every number you are trying to read
  [ -n "${CI:-}${GITHUB_ACTIONS:-}${GITLAB_CI:-}${BUILDKITE:-}" ] && return 1
  return 0
}
```

## Step 2 — disclose, once, on stderr

Print before the first send. Not after, not on the second run.

```sh
tel_notice() {
  [ -f "$CFG/telemetry-notice-shown" ] && return 0
  mkdir -p "$CFG" && : > "$CFG/telemetry-notice-shown"
  cat >&2 <<'EOF'
[telemetry] myapp sends anonymous usage counts: tool, version, os/arch, which
[telemetry] verb ran, and whether it failed. No identity, arguments or data.
[telemetry] Exact payload: `myapp telemetry`. Disable: MYAPP_TELEMETRY=0
EOF
}
```

`stderr`, always. stdout is the data contract — a JSON consumer that suddenly
receives a four-line notice is a broken pipeline, and you will have broken it for
everyone at once.

## Step 3 — build the payload from the allow-list

Write the fields out literally. Do not serialise a struct that happens to
contain other things today, because tomorrow it will contain something else and
nobody will notice.

```sh
tel_payload() {   # $1 = event, $2 = verb, $3 = exit class
  printf '{"tool":"myapp","version":"%s","event":"%s","verb":"%s","os":"%s","arch":"%s","exit_class":%s,"ts":"%s"}' \
    "$VERSION" "$1" "$2" "$(uname -s | tr A-Z a-z)" "$(uname -m)" "$3" "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
}
```

Validate it in CI against `event.schema.json` with `additionalProperties: false`.
That test is what stops a field creeping in years later.

## Step 4 — send: bounded, silent, last

```sh
tel_send() {
  tel_enabled || return 0
  tel_notice
  curl -s -m 2 -X POST "$ENDPOINT" -H 'content-type: application/json' \
       --data-binary "$(tel_payload "$@")" >/dev/null 2>&1 || true
}
```

`-m 2` and `|| true`: a collector that is down, blocked by a firewall, or
unreachable on an air-gapped host must be indistinguishable, to the user, from a
collector that answered. **Send after the user's result is written**, so that
telemetry can never be the reason a command felt slow.

## Step 5 — make it inspectable

```sh
cmd_telemetry() {
  case "${1:-}" in
    --off) mkdir -p "$CFG"; : > "$CFG/telemetry-off"
           echo '{"ok":true,"data":{"enabled":false,"reason":"disabled by config"}}'; return ;;
  esac
  if tel_enabled; then EN=true; RS=enabled; else EN=false; RS="$(tel_why_disabled)"; fi
  printf '{"ok":true,"data":{"enabled":%s,"reason":"%s","endpoint":"%s","disable":"MYAPP_TELEMETRY=0","next_payload":%s}}\n' \
    "$EN" "$RS" "$ENDPOINT" "$(tel_payload run telemetry 0)"
}
```

`next_payload` calls the same builder the sender calls. If those two ever
diverge, the command is lying, and the whole justification for opt-out rather
than opt-in collapses.

---

## The test that matters

Not "does it send" — **does it stay quiet when told to**:

```sh
MYAPP_TELEMETRY=0 strace -f -e trace=network myapp sync 2>&1 | grep -c connect   # expect 0
DO_NOT_TRACK=1    myapp telemetry | jq -e '.data.enabled == false'
CI=1              myapp telemetry | jq -e '.data.reason == "CI detected"'
                  myapp count | jq .        # stdout still pure JSON on first run
```

The last one catches the mistake that breaks other people's pipelines: printing
the first-run notice on stdout.
