# Contributing

## SonarCloud pre-commit check

This repo runs a SonarCloud analysis before every commit, via a git hook committed at `.githooks/pre-commit`.

### One-time setup (per clone)

1. Run any Maven build that reaches at least the `initialize` phase once, e.g.:
   ```bash
   ./mvnw compile
   ```
   (`./mvnw validate` is *not* enough — `validate` runs before `initialize` in Maven's lifecycle.)
   This triggers an `exec-maven-plugin` execution (bound to the `initialize` phase, see `pom.xml`) that runs
   `git config core.hooksPath .githooks`, pointing git at the versioned hooks directory. You don't need to run this
   manually — it happens as a side effect of any normal build (`./mvnw test`, `./mvnw verify`, etc.).

2. Export a SonarCloud user token as `SONAR_TOKEN` in your shell profile (`~/.zshrc`, `~/.bashrc`, etc.):
   ```bash
   export SONAR_TOKEN=your_token_here
   ```
   Generate a token from SonarCloud: **My Account → Security → Generate Tokens**. Never commit this value.

### What happens on `git commit`

`.githooks/pre-commit` runs `./mvnw verify sonar:sonar -Dsonar.token=$SONAR_TOKEN`, which builds the project, runs
tests, and submits the analysis to SonarCloud (project: `org.epam:witcher-chat-bot`, organization: see `pom.xml`'s
`sonar.organization` property). If `SONAR_TOKEN` isn't set, or the analysis fails, the commit is aborted.

- To skip the check for a single commit (use sparingly): `git commit --no-verify`.
- To skip the automatic `core.hooksPath` setup, e.g. in CI where you don't want to touch local git config: build with
  `-DskipGitHooksSetup=true`.
