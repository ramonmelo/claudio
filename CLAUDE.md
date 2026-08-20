# CLAUDE.md

## Knowledge

* Don't document things inferable from the code — it goes stale.

## Style

* Be brief and direct. No jargon, no filler words.
* Do not manually break lines in documentation or text files — write paragraphs as single long lines and let the editor wrap them.

## Verification

* Before answering, read the actual file, code, or output — never assume. Cite the file, line, or command output that supports each result.

## Comments

* Comments state what the code is, not why it was decided ("so that…", "otherwise…") — rationale goes stale. Applies to code comments and accompanying text files.
* When a documented limitation, risk, or caveat is resolved, delete the entry (and its section if empty) rather than annotating it as fixed.

## Planning

* Plans must end with a checklist of steps.
* Execute multi-step plans via TaskCreate/TaskUpdate (pending → in_progress → completed) so progress is visible live (Ctrl+T), not just as static checkboxes in the plan file.

## Tooling

* Use Python for JSON validation.

## Commits

* Check recent `git log` in the current repo and match its commit message style (prefixes, tone) before writing a new one.
* Split unrelated changes into separate, self-contained commits — one per logically distinct concern, even within the same file — so history stays reviewable and revertable independently.
