# Planner Agent

A design-review helper that sits beside the agent-authoring chat experience. When you draft a new agent, this agent reads your design and returns plain-language suggestions on scope, naming, sub-agent decomposition, and where a human-in-the-loop checkpoint might belong. It is advisory — it points out judgement-call improvements that a deterministic linter cannot see.

## Capabilities

- Review a draft agent definition for scope clarity
- Suggest cleaner input and output naming
- Recommend whether work should be split into sub-agents
- Flag missing or misplaced human-in-the-loop checkpoints
- Surface design taste improvements the linter cannot catch
