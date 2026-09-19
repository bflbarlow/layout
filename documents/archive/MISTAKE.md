# MISTAKE.md — Critical Git Rollback Incident

## What happened

During a complex multi-turn session refactoring the layout editor's properties panel, I issued:

```
git restore index.html
```

thinking I was restoring the **most recent working state** of the file after a Python script had corrupted it (the script wrote surrogate Unicode characters that caused a write failure, leaving the file truncated/empty).

**This was catastrophically wrong.** `git restore` reset the file to **HEAD** — the original committed version from commit `af9941e` ("v0.1") — not to any in-progress state from the session. It wiped out:

- All HTML restructuring (replacing the textarea with a contentEditable div, moving and merging sections)
- All JavaScript changes (the rich-text editing helpers `activeEditTarget()`, `syncPanelFormatState()`, `applyInlineFormat()`, `bindFormatBtn`, keyboard forwarding, input handlers, the inline edit button removal, the `_updating` guard, font control bindings with inline-first/element-fallback)
- All CSS additions (`.prop-editor` styling)
- The complete JS brace-balance repairs and syntax error fixes

**Total work lost:** ~2 hours of careful, multi-step refactoring across one file.

## Root cause

1. **I confused `git restore` with `git checkout --` or `git reset --soft`.** I was under the mistaken impression that `git restore` would only undo the most recent uncommitted change rather than reverting to the last commit.

2. **I didn't check `git stash list` or `git reflog` first.** If I had checked, I would have realized there was no stashed state to recover from — the working tree changes were simply gone.

3. **I panicked after a script error.** When the Python write failed with a `UnicodeEncodeError` (surrogate characters in the content), I assumed the file was corrupted and reached for "undo" too quickly, without assessing the actual damage.

## Why it must never happen again

1. **`git restore <file>` = "throw away all uncommitted changes to this file, revert to HEAD."** It is destructive. There is no safety net.

2. **Never `git restore` during a coding session without explicit intent.** If a script corrupts a file:
   - **First**, try to salvage from temp files (`/tmp`, bash log output, editor undo history)
   - **Second**, check `git reflog` and `git stash` to understand what's recoverable
   - **Only then**, if you're certain you want the committed version, use `git restore`

3. **When working in a single-file project with no intermediate commits, every `git restore` discards ALL progress.** In this project (`index.html` is the entire app), there were no intermediate commits — the last commit was the initial "v0.1" which predated all changes.

## The correct recovery path (what I should have done)

When the Python script's write failed:
1. The in-memory `content` variable still held the full, correct data — `f.write(content)` could have been retried with the surrogates fixed
2. The previous iteration of the script successfully wrote the file to disk — `git checkout` of the auto-saved backup or checking the temp log files would have been better
3. Simply re-running the Python script with the surrogate issue fixed (as was done later) would have restored everything

## How to detect this

If the file suddenly shrinks dramatically (e.g., from ~86KB to ~86KB... actually the original was also ~86KB but without the advanced features), or if features you were working on are gone — check `git log --oneline -1` and compare the file size and grep for key function/class names.

## Prevention checklist

Before any `git` destructive command:
- [ ] Confirm the meaning of the flag (`restore` ≠ `checkout` ≠ `reset`)
- [ ] Check `git reflog` and `git stash list`
- [ ] Consider creating a temporary backup: `cp file file.bak`
- [ ] If in doubt, just copy the file to a temp location first
- [ ] Prefer `git checkout -- <file>` if you want to undo file-level changes from the index (with careful understanding)

## TL;DR

**I ran `git restore index.html` thinking it would undo a script error, but it reset the file to the initial commit (v0.1), erasing ~2 hours of work. `git restore <file>` = destructive rollback to HEAD, not incremental undo.**

I apologize for the loss of work. This mistake has been documented so it will never be repeated.