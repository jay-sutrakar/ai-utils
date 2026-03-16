# AI Utils — GitHub Copilot Custom Instructions

Reusable GitHub Copilot instruction files for Java/Spring Boot projects and Vert.x 4→5 migration.

## Files

| File | Scope | Purpose |
|---|---|---|
| `.github/copilot-instructions.md` | Repo-wide (all files) | Coding conventions, project structure, build/test/run commands, migration rules |
| `.github/instructions/java.instructions.md` | `**/*.java` | Java source code patterns — class structure, DI, DTOs, entity design, null safety |
| `.github/instructions/tests.instructions.md` | `**/test/**/*.java` | Test conventions — naming, Arrange-Act-Assert, AssertJ, Mockito, Testcontainers |
| `.github/instructions/dependencies.instructions.md` | `**/pom.xml`, `**/build.gradle*` | Dependency analysis — Vert.x migration, removed/sunsetted artifacts, version checks |

## Usage

1. Copy the `.github/` folder into your project root
2. Customize the placeholder sections in `copilot-instructions.md` (project overview, tech stack)
3. Reload Copilot in your IDE
4. Copilot will automatically apply these rules based on the file you're editing

## Vert.x Migration

The instructions cover the full Vert.x 4.0.0 → 5.0.8 migration path:
- Phase 1: Upgrade to 4.5.25 and resolve deprecations
- Phase 2: Upgrade to 5.0.8 with all breaking changes handled
- Dependency analysis with removed, sunsetted, and renamed artifacts
- Java code patterns with old → new API mappings

## References

- [GitHub Copilot Custom Instructions Docs](https://docs.github.com/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot)
- [Vert.x 5 Migration Guide](https://vertx.io/docs/guides/vertx-5-migration-guide/)
- [Vert.x 5.0.8 Release](https://vertx.io/blog/eclipse-vert-x-5-0-8/)
