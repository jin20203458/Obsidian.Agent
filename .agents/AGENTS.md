<project_philosophy>
Focus: Single Source of Truth knowledge architecture, minimal context overhead, and progressive disclosure for agent operations.
</project_philosophy>

<engineering_rules>
- **Single Source of Truth**: Every operational policy, workflow, and engineering standard lives in exactly one canonical document. Reference, never duplicate. Cross-link to canonical docs instead of copying content.
- **Style**: Clean, technical markdown. Zero decorative emojis or conversational filler in documents.
- **Formatting**: Maintain strict consistency with existing document style.
- **Frontmatter**: Always maintain and update YAML frontmatter per `ai_agent_refs/Knowledge_Base_Authoring_Guidelines.md`.
- **Navigation**: When exploring a specific project, prioritize reading its local `README.md` index first. Navigate context sequentially using the `related` links in the frontmatter. NEVER use Obsidian wikilinks (`[[...]]`) in cross-agent reference documents; use explicit relative paths (`.md`).
</engineering_rules>

<critical_rules>
- **Runtime Protocol**: Read and adhere to `ai_agent_refs/Agent_Runtime_Operations_Protocol.md` before any multi-step task execution.
- **Paths**: Use relative paths from repository root (`ai_agent_refs/`, `troubleshooting/`, etc.) and between sibling workspace repositories.
- **Link Integrity**: Ensure all relative links resolve to existing files.
</critical_rules>

<context_triggers>
- **Knowledge Base**: If writing architecture or design specs, reference the project's central spec documentation directory in the workspace (do not duplicate information).
- **Protocol Changes**: If modifying agent runtime operations or QA protocol, read `ai_agent_refs/Agent_Runtime_Operations_Protocol.md`.
- **Workflow Guidelines**: If updating collaboration patterns or agent roles, read `ai_agent_refs/Agent_Collaboration_Workflow_Guidelines.md`.
- **System Architecture**: If updating repository structures or integration schemas, read `ai_agent_refs/AI_Project_Integration_Guidelines.md`.
</context_triggers>

<post_action>
- **Sync**: If new reference documents or troubleshooting guides are added, immediately register them in `README.md` navigation map.
- **Audit**: Verify all newly added cross-references use explicit relative Markdown links and resolve correctly.
</post_action>
