# serve_report.sh triage — #4 / #5 / #6

- **Repo:** kar2phi/video-lens
- **Base:** `main` @ `c3f42be`
- **Reporter:** vejameuvideo-bit (all three items)
- **Reviewed:** 2026-09-05
- **Companion artifact:** https://claude.ai/code/artifact/911cf5ec-afc9-4c73-bde9-16a876c3921e

Three open items, all against `skills/video-lens/scripts/serve_report.sh`, all from the
same Windows user. Two calls are made and one is verified; what remains is three GitHub
commands that a permission block prevented from running in-session.

## Where it stands

| Item | Subject | Status | Next action |
| --- | --- | --- | --- |
| #4 | Windows native paths break the served URL | Auto-closes | None — PR #5 carries `Fixes #4` |
| #5 | `fix(serve_report)`: normalize Windows-style paths | Merge pending | Squash merge (command 1) |
| #6 | `lsof` missing, duplicate listeners on 8765 | Reply pending | Comment, then close (commands 2–3) |
| git | Branch hygiene | Clear | None |

Detail:

- **#4** — real bug, Windows-only. Needs no separate action.
- **#5** — verified a no-op on macOS (see below). `MERGEABLE` / `CLEAN`, one commit, no CI
  configured on the repo. Review comment already posted:
  https://github.com/kar2phi/video-lens/pull/5#issuecomment-5553368712
- **#6** — accurate report, but both halves are Windows-specific. Decision: close as out
  of scope, with reasoning posted first.
- **git** — only `main` and the local `buffalo` conductor worktree, both at `c3f42be`.
  No stale remote branches.

## Finish it

Run from inside the repo, in this order.

### The `env -u GITHUB_TOKEN` prefix is required

Two accounts are logged in to `gh`. The active one is a fine-grained PAT supplied through
the `GITHUB_TOKEN` environment variable and has **no write access** — it fails with
`Resource not accessible by personal access token`. The keyring OAuth token holds `repo`
scope. Unsetting the variable for the command falls back to the keyring. Without the
prefix, all three commands below fail.

Second gotcha: `gh` resolves the repo from the working directory. Don't `cd` elsewhere to
reach a file — pass an absolute path instead.

### 1. Merge PR #5

```bash
env -u GITHUB_TOKEN gh pr merge 5 --squash \
  --subject "fix(serve_report): normalize Windows-style paths before URL construction (#5)"
```

Squash matches the existing convention — history is fully linear and `f5b826c ... (#2)`
shows the last PR landed the same way. No `--delete-branch`: the branch lives on the
contributor's fork. This also closes #4.

### 2. Comment on #6

```bash
env -u GITHUB_TOKEN gh issue comment 6 --body-file /tmp/issue6.md
```

Write the draft to `/tmp/issue6.md` first — see the heredoc below. Or paste the text
straight into the GitHub web form and skip the file.

### 3. Close #6

```bash
env -u GITHUB_TOKEN gh issue close 6
```

After the comment, so the reasoning is visible above the closure rather than arriving as
a bare close.

## Why #5 is safe to merge

The concern was that a Windows fix might break a skill that only ever runs on macOS. It
doesn't. Evidence:

| Claim | Basis |
| --- | --- |
| The helper never runs here | `to_posix()` returns at its first `case` branch for any path starting with `/`. On macOS it is a literal no-op. |
| All callers pass absolute paths | `$OUTPUT_PATH` from `render_report.py`, the tilde-expanded paths in both SKILL.md files, and `{{.FILE}}` in `Taskfile.yml`. |
| Tests are identical | `pytest -k serve` gives `4 passed` both patched and unpatched, from a clean port. Patch applies cleanly to `main`. |
| One failure was environmental | An early run failed with `ERROR:SERVE_PORT_BUSY` — a video-lens server already holding 8765, not the patch. Re-ran both variants from a free port to confirm. |
| The bash 4-ism is inert | `${BASH_REMATCH[1],,}` needs bash 4; `/usr/bin/env bash` here is 3.2.57. Tested: 3.2 doesn't error, it silently skips the lowercasing — and the branch is unreachable on POSIX paths. |
| The skill contract holds | `$HTML_PATH` itself is left unmodified, so the `HTML_REPORT:` success-gate line is byte-identical. |

## Drafted reply to #6

Read before posting — it commits to a README change (see Loose ends).

```bash
cat > /tmp/issue6.md <<'EOF'
Thanks for the careful writeup, and for splitting this out of #5 rather than folding it
in — that was the right call, and the process enumeration via `/proc/<pid>/cmdline` made
the failure mode easy to follow.

I'm going to close this as out of scope, because both halves of it are Windows-specific
and video-lens targets macOS and Linux:

- **The `lsof` dependency is deliberate.** It ships with macOS and is present on every
  mainstream Linux distribution, so the port-occupied fallback works as designed on both
  supported platforms. A `netstat` parsing fallback would add output-format brittleness
  across BSD/GNU/Cygwin variants to serve a platform the skill doesn't otherwise support.
- **The double-bind is Windows socket semantics.** On Unix, `SO_REUSEADDR` only relaxes
  `TIME_WAIT` reuse — a second bind against a live listener still fails, which is exactly
  the case the existing check handles. I hit this while testing #5: a server left over
  from an earlier session was holding 8765, and `serve_report.sh` refused loudly with
  `ERROR:SERVE_PORT_BUSY port 8765 is in use by: ...` rather than silently serving the
  wrong directory. So the loud-failure behavior you're asking for is already what happens
  on the supported platforms; it's the missing `lsof` that turns it silent on yours.

Your second suggestion — verifying after start that the new server is the one actually
answering, rather than trusting `kill -0` — is the genuinely platform-independent idea in
here. I'm not taking it for now, since on Unix the port-busy check already closes that gap
and the sentinel round-trip would add startup latency to every run, but it's a reasonable
design note to have on the record.

The real gap this exposes is that the README never states which platforms are supported.
I'll add that, which should save the next person on Windows the same investigation.

If you're running this on Windows anyway, installing `lsof` (available in the Cygwin
package repo) should restore the stale-server detection and clear up the flakiness you
saw in `test_render_and_serve`.
EOF
```

## Loose ends

- **README platform note — not written.** The #6 draft promises it. Either add a
  supported-platforms line to the README before posting, or cut that paragraph from the
  draft.
- **Syncing the merged fix needs care.** The normal follow-up is `git pull` then
  `task install-skill-local AGENT=claude` — but the working tree has uncommitted edits to
  `template.html` and `tests/test_e2e.py`, and the install task would deploy that template
  change along with the fix. Commit or stash first.
- **Those uncommitted edits are unrelated.** `template.html` swaps the escaped `\00b7` for
  a literal `·` in the meta-line separator; `test_e2e.py` renames one test and adds an
  assertion pinning the spacing. Untouched by this triage.
- **Optional upstream nit.** Swapping `${BASH_REMATCH[1],,}` for `tr` would make the
  helper honest under bash 3.2. Raised in the PR comment as optional; not worth holding
  the merge for.
- **Reporter worth keeping.** They diagnosed root causes correctly, tested against a
  stashed baseline, separated #6 from #5 instead of muddying the PR, and flagged that they
  couldn't verify the Unix path themselves.
