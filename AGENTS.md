# Maintaining cli-telemetry-spec

This spec exists to be *defensible to the person running the tool*. Every change
is judged against that, not against what would be convenient to measure.

## Rules for changes

- **The allow-list only shrinks easily.** Adding a field requires stating, in
  PROTOCOL.md, what question it answers that the existing fields cannot, and why
  it cannot identify anyone. If that paragraph is hard to write, the field is
  wrong.
- **`event.schema.json` keeps `additionalProperties: false`.** It is the
  enforcement mechanism, not documentation.
- **Never add a field that is a hash of something identifying.** A hashed
  hostname is a hostname.
- **Never add events.** Three exist. A fourth is a request log.
- **Never make telemetry improve the tool for the user.** The moment it does,
  disabling it costs them something, and consent stops being free.

## Style

Match the other four specs: normative MUST/SHOULD language in PROTOCOL.md, a
copy-pasteable RECIPE.md, a compressed llms.txt for agents, and a README that
opens with the problem rather than the solution.
