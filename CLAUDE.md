# CLAUDE.md

* Don't document things inferable from the code - it goes stale.
* Do not manually break lines in documentation or text files - write paragraphs as single long lines and let the editor wrap them.
* Before answering, read the actual file, code, or output — never assume. Cite the file, line, or command output that supports each result.
* Comments state what the code is, not why it was decided ("so that…", "otherwise…") — rationale goes stale. Applies to code comments and accompanying text files.
* When a documented limitation, risk, or caveat is resolved, delete the entry (and its section if empty) rather than annotating it as fixed.
* Check recent `git log` in the current repo and match its commit message style (prefixes, tone) before writing a new one.
* Split unrelated changes into separate, self-contained commits — one per logically distinct concern, even within the same file — so history stays reviewable and revertable independently.
* Use ASCII hyphens (-) only.
* If you are unsure about any information, API, function signature, file path, or implementation detail, stop and ask me rather than guessing. Never fabricate code, file contents, or terminal output. If you don't know something, say so.
* Lead with the recommended approach. The first sentence should state what to do, before any reasoning or caveats. Give the "why" only after the recommendation.
* Put rejected alternatives and their downsides last, and only when they change the decision. Do not open by explaining what isn't the solution.
