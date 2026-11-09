# ADR-001: Start with RAG, add narrow tools, no multi-agent

Context: Most questions are answered by help articles. A few need live data
(usage, balance) or exact arithmetic (roaming cost).

Decision: Always retrieve context (RAG, level 3). Give the model three narrow
tools (level 4) for account lookup, roaming cost and ticket creation.
No multi-agent design.

Consequences: Knowledge questions need one model call. Tool questions need two
or more, which costs latency on a CPU laptop. Small models may pick tools
badly, so we measure it.

Revisit when: tool-choice accuracy stays below target after prompt fixes, or a
new feature needs long-running multi-step work.
