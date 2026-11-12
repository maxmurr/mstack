---
name: council
description: Trigger this if the user says 'council'.
---

# Council

Before an audit, address outstanding PR comments, fix known issues, and run
relevant checks. Give council the updated implementation; do not defer known
fixes until reviews return. If the user wants to discard the implementation
and explore new ideas, skip fixes to code being replaced.

Bring in three delegates to help with the user's request, usually for review
or feedback. Use one strong model from each of three different providers, at
high reasoning effort. If the user's instructions name council members, use
those. Otherwise pick the latest flagship model from each provider, for
example Claude Opus, GPT Sol, and Gemini Argon.

Start all three at once. Write the brief to a file first. Each delegate must
be read-only:

| Provider | How |
| --- | --- |
| Anthropic | In Claude Code, a native subagent with the Opus model. Elsewhere, `"$(command -v claude)" -p --model <model> --effort high --permission-mode plan "$(cat "$brief")"` |
| OpenAI | `"$(command -v codex)" exec -m <model> -c model_reasoning_effort=high -s read-only -o "$out" "$(cat "$brief")"` |
| Google | `"$(command -v agy)" --model <slug> --print-timeout 30m --settings '{"permissions":{"deny":["command(*)","write_file(*)"]}}' -p "$(cat "$brief")"` |

Run each CLI delegate in its own background Rex tab (`REX_SESSION` is set):

```sh
rex run -s "$REX_SESSION" --label council-<name>-r<N> --keep-open --focus=false \
  -c "$repo" --json -- <command>
```

Save each `.block_ids[0]`. Wait on each delegate with
`rex wait -s "$REX_SESSION" <block> --json` as its own background command (in
Claude Code, `run_in_background`), so each exit wakes you. Read the reply with
`rex capture -s "$REX_SESSION" -b <block> --trim --unwrap`, or from `$out` for
Codex. A nonzero exit, an empty reply, or a `no output produced` message is a
failed review, not a pass. Leave the tabs open for the user.

- **Model IDs:** resolve them, do not hardcode. `agy models` lists slugs with
  the effort built in; pick the latest flagship Gemini at `-high`.
- **Absolute CLI paths:** Rex tabs load the login shell's PATH, not the
  interactive one, so a bare name can start an older install.
- **agy runs no shell commands** under those settings (headless `agy` would
  otherwise write files). Put the full diff in its brief, for example by
  appending `git diff <base>` output. It reads the rest of the code with its
  own file tools.

Outside Rex, use the harness's subagent tools. If you cannot reach three
providers, or a named member is unavailable, report it instead of
substituting.

Give each delegate the request and relevant context. For review or feedback,
keep their work read-only. Collect their responses and summarize the useful
findings, disagreements, and recommended next steps.

Do not wait idle. When a result arrives, check its findings against the code
at once and decide what to fix while the other delegates run. Do not edit
files while a delegate still reviews the same checkout. If the user's
instructions mark a delegate as non-blocking, do not wait for it; treat a
late result as feedback and do not record its score.

For a code review, ask each delegate for a score out of 10.
Count each full review pass as one round.
A fresh review after fixes starts a new round.

If the work ends in a pull request, put the results in the PR description.
Put this section before the model and harness line at the end:

```markdown
## Council review

Rounds: 2

| Delegate | Round 1 | Round 2 |
| --- | --- | --- |
| Claude Opus | 7/10 | 9/10 |
| GPT Sol | 8/10 | 10/10 |
| Gemini Argon | 7/10 | 9/10 |
```

If a delegate gives no score, write `no score`. Do not invent one.
If the PR is open, edit the description. Do not post a comment for this.
If there is no PR, put the same results in the final reply only.

If you agree with their recommendations, make the changes and request a fresh
review after. Continue this process until all delegates are satisfied with the
outcome.

Do not let repeated reviews result in scope creep. It is your job to fight to
keep scope limited.
