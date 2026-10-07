<project_philosophy>
Focus: Human-agent shared SSOT knowledge architecture, minimal context overhead, and progressive disclosure.
</project_philosophy>

<engineering_rules>
- **SSOT**: Every policy, workflow, and standard lives in exactly one canonical document. Reference via cross-links; never duplicate.
- **Paths & Formatting**: ASCII snake_case paths only (no spaces/non-ASCII). Technical markdown; zero decorative emojis.
- **Links & Navigation**: Use explicit relative Markdown links (`.md`) exclusively; avoid wikilinks (`[[...]]`). Traverse projects sequentially via `README.md` and frontmatter `related` links.
</engineering_rules>

<critical_rules>
- **Paths & Integrity**: Use relative paths from repo root (`ai_agent_refs/`, `troubleshooting/`) and sibling workspaces (`../../<Repo>`). Verify all links resolve to physical files.
</critical_rules>

<context_triggers>
- **Document Authoring & Structure**: `ai_agent_refs/Knowledge_Base_Authoring_Guidelines.md`
- **New Project Onboarding**: `ai_agent_refs/AI_Project_Integration_Guidelines.md`
</context_triggers>

<post_action>
- **Sync**: Register newly created documents in `README.md` navigation map.
- **Audit**: Verify all new cross-references resolve to existing relative Markdown files.
</post_action>
