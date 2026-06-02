Triage the files in _inbox/. For each file:

Read frontmatter and content.
Determine target path per repo conventions:
Checkpoints (file matches YYYY-MM-DD--<scope>--<slug>.md and has the checkpoint schema) → projects/<project>/checkpoints/ or general/checkpoints/
Non-checkpoint artefacts → infer correct path from content, cross-reference the checkpoint's "Artefacts produced" section if it lists them
Check _meta/topics.md and_meta/projects.md — flag any topic tags in the checkpoint frontmatter that are not in topics.md as "unconfirmed".
Show me the proposed moves (source → destination) before executing
anything. Do not commit yet.
