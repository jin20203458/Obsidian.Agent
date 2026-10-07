<project_philosophy>
Focus: Human-agent shared SSOT knowledge architecture, minimal context overhead, and progressive disclosure.
</project_philosophy>

<engineering_rules>
- **SSOT**: Every policy, workflow, and standard lives in exactly one canonical document. Reference via cross-links; never duplicate.
- **Style & Frontmatter**: Technical markdown; zero decorative emojis. Always maintain YAML frontmatter (`description`, `related`).
- **Links & Navigation**: Use explicit relative Markdown links (`.md`) exclusively; avoid wikilinks (`[[...]]`). Traverse projects via `README.md` and frontmatter links.
</engineering_rules>

<critical_rules>
- **Paths & Integrity**: Use relative paths from repo root (`ai_agent_refs/`, `troubleshooting/`) and sibling workspaces (`../../<Repo>`). Verify all links resolve to physical files.
</critical_rules>

<context_triggers>
- **Runtime Operations & QA**: `ai_agent_refs/Agent_Runtime_Operations_Protocol.md`
- **Collaboration & Agent Roles**: `ai_agent_refs/Agent_Collaboration_Workflow_Guidelines.md`
- **Repo Architecture & Integration**: `ai_agent_refs/AI_Project_Integration_Guidelines.md`
- **Specs & Architecture**: Reference target workspace central spec directory without duplication.
</context_triggers>

<post_action>
- **Sync**: Register newly created documents in `README.md` navigation map.
- **Audit**: Verify all new cross-references resolve to existing relative Markdown files.
</post_action>
