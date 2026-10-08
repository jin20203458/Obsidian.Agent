<project_philosophy>
Focus: Human-agent shared SSOT knowledge architecture, minimal context overhead, and progressive disclosure.
</project_philosophy>

<engineering_rules>
- **SSOT**: Every policy, workflow, and standard lives in exactly one canonical document. Reference via cross-links; never duplicate.
- **Naming**: ASCII only, zero spaces. Guidelines in `ai_agent_refs/` use `Title_Snake_Case.md` (`Agent_Runtime_Operations_Protocol.md`). Project docs and runbooks use `snake_case.md` (`01_system_architecture.md`, `phalanx.md`). Project folders use canonical names (`Phalanx/`, `LLVM/`).
- **Formatting**: Technical markdown; zero decorative emojis.
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
