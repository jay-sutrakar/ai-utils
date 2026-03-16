# AI Utils — GitHub Copilot Custom Instructions

Reusable GitHub Copilot instruction files for Vert.x projects and Vert.x 4 → 5 migration.

## Files

| File | Scope | Purpose |
|---|---|---|
| `.github/copilot-instructions.md` | Repo-wide (all files) | Vert.x coding conventions, project structure, build/test/run commands, migration rules |
| `.github/instructions/java.instructions.md` | `**/*.java` | Vert.x Java patterns — verticles, futures, handlers, services, SQL client, EventBus |
| `.github/instructions/tests.instructions.md` | `**/test/**/*.java` | Vert.x testing — vertx-junit5, VertxTestContext, WebClient tests, Testcontainers |
| `.github/instructions/dependencies.instructions.md` | `**/pom.xml`, `**/build.gradle*` | Dependency analysis — removed/sunsetted artifacts, version checks, BOM management |

## Usage

1. Copy the `.github/` folder into your Vert.x project root
2. Customize the placeholder sections in `copilot-instructions.md` (project overview)
3. Reload Copilot in your IDE
4. Copilot will automatically apply these rules based on the file you're editing

## Vert.x Migration

The instructions cover the full Vert.x 4.0.0 → 5.0.8 migration path:
- Phase 1: Upgrade to 4.5.25 and resolve deprecations
- Phase 2: Upgrade to 5.0.8 with all breaking changes handled
- Dependency analysis with removed, sunsetted, and renamed artifacts
- Java code patterns with old → new API mappings
- Test migration from vertx-unit to vertx-junit5

## References

- [GitHub Copilot Custom Instructions Docs](https://docs.github.com/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot)
- [Vert.x 5 Migration Guide](https://vertx.io/docs/guides/vertx-5-migration-guide/)
- [Vert.x 5 Breaking Changes Wiki](https://github.com/eclipse-vertx/vert.x/wiki/Vert.x-5)
- [Vert.x 5.0.8 Release](https://vertx.io/blog/eclipse-vert-x-5-0-8/)
