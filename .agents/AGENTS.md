<project_philosophy>
Focus: Human-agent shared SSOT knowledge architecture, minimal context overhead, and progressive disclosure.
</project_philosophy>

<engineering_rules>
- **SSOT**: Every policy, workflow, and standard lives in exactly one canonical document. Reference via cross-links; never duplicate.
- **Naming & Style**: ASCII only, zero spaces/non-ASCII. PascalCase for project folders (`Phalanx`, `GRC`) and core guidelines (`Agent_Runtime_Operations_Protocol.md`). Snake_case for docs and troubleshooting (`01_architecture.md`, `phalanx.md`). Technical markdown; zero decorative emojis.
- **Frontmatter**: Include YAML frontmatter with concise `description` and relative `related` links for progressive disclosure.
- **Links & Navigation**: Use standard Markdown syntax (`.md`) exclusively; NEVER use Obsidian wikilinks (`[[...]]`). Traverse projects sequentially via `README.md` and frontmatter `related` links.
</engineering_rules>

<critical_rules>
- **Relative Paths & Integrity**:
  - From repo root: `ai_agent_refs/`, `troubleshooting/`, `<Project>/`, `../<Repo>`.
  - From `ai_agent_refs/` and `troubleshooting/`: `../README.md`, `../../<SiblingRepo>`.
  - From `<Project>/docs/`: `../README.md`, `../../README.md`, `../../../<SiblingRepo>`.
  - Verify all relative paths resolve to physical files on disk.
</critical_rules>

<context_triggers>
- **Document Authoring**: Structure, YAML frontmatter, naming, formatting -> `ai_agent_refs/Knowledge_Base_Authoring_Guidelines.md`
- **Project Onboarding**: External repo integration, architecture specs -> `ai_agent_refs/AI_Project_Integration_Guidelines.md`
</context_triggers>

<post_action>
- **Sync**: Register newly created documents in root `README.md` and corresponding `<Project>/README.md`.
- **Audit**: Ensure valid YAML frontmatter (`description`, `related`) and verify all relative Markdown links resolve to existing physical files.
</post_action>
