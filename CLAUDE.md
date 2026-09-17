# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project state

This is a freshly-scaffolded Spring Boot application (`witcher-chat-bot`) with no business logic yet — only the
default `SpringApplication` bootstrap class and default config. There is no README describing product intent.
Treat any architectural conventions below as defaults to follow once real code is added, not as documentation of
existing structure.

## Commands

Build and run (Maven wrapper, no local Maven install required):

```bash
./mvnw clean install       # build, run tests
./mvnw spring-boot:run     # run the app locally
./mvnw test                # run all tests
./mvnw test -Dtest=WitcherChatBotApplicationTests   # run a single test class
./mvnw test -Dtest=WitcherChatBotApplicationTests#contextLoads   # run a single test method
```

## Architecture

- **Language/runtime**: Java 21, Spring Boot 4.1.1 (via `spring-boot-starter-parent`).
- **Build**: Maven, using the `mvnw` wrapper — always invoke through `./mvnw`, not a system-installed `mvn`.
- **Base package**: `org.epam.witcherchatbot`, entry point `WitcherChatBotApplication`.
- **Config**: `src/main/resources/application.yaml` (YAML, not `.properties`).
- **Dependencies currently present**: `spring-boot-starter` (core only — no web, data, or AI/chat starters yet) and
  Lombok (annotation processing wired into the compiler plugin for both `compile` and `test-compile`).

## Spec-driven workflow (Speckit)

This repo is initialized with GitHub Speckit (`.specify/`) integrated with Claude Code skills (`.claude/skills/speckit-*`).
Feature work is expected to flow through these slash-command skills in order:

1. `speckit-constitution` — establish/update project principles (`.specify/memory/constitution.md` is currently an
   unfilled template — populate it before relying on it for governance decisions).
2. `speckit-specify` — write a feature spec.
3. `speckit-clarify` — resolve ambiguities in the spec.
4. `speckit-plan` — produce an implementation plan.
5. `speckit-tasks` — generate dependency-ordered tasks.
6. `speckit-analyze` / `speckit-checklist` — cross-check consistency/quality before implementing.
7. `speckit-implement` — execute tasks.
8. `speckit-converge` — reconcile codebase against spec/plan/tasks and append any remaining work.

Feature numbering is sequential (`.specify/init-options.json`); specs/plans/tasks for a feature live together once
created (no features have been created yet).
