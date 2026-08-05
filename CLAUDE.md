# CLAUDE.md

## Knowledge
* If something can be inferred from the code, don't write it on READMEs or CLAUDE.md, otherwise the information can get old.

## Style
* Be brief and direct. No jargon, no filler words.
* Do not manually break lines in documentation or text files. Write paragraphs as single long lines and rely on editor line wrapping.

## Verification
* Before answering, read the actual file, code, or output — never assume. Cite the file, line, or command output that supports each result.

## Comments
* Comments state what the code is, not why it was decided — omit rationale and decision history ("so that…", "otherwise…"); it goes stale. Applies to code comments and accompanying text files.
* When a documented limitation, risk, or caveat is resolved, delete the entry (and its section if empty) rather than annotating it as fixed.

## Planning
* Plans must end with a checklist of steps.
* Execute multi-step plans via TaskCreate/TaskUpdate (pending → in_progress → completed) so progress is visible live (Ctrl+T), not just as static checkboxes in the plan file.

## Tooling
* Use Python for JSON validation.
