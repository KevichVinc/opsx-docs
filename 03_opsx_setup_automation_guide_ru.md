# Инструкция: как настроить OpenSpec / OPSX под командный SDD-flow

> Цель: настроить OpenSpec так, чтобы workflow поддерживал stage gates:
>
> `target → decomposition → new → proposal → specs → review → design → tasks → apply → verify → archive`
>
> и чтобы master specs обновлялись только через небольшие independently deliverable changes.

---

# 1. Что именно мы автоматизируем

OpenSpec будет автоматически обеспечивать:

- хранение change artifacts;
- порядок `proposal → specs → design → tasks`;
- генерацию одного artifact за шаг через `continue`;
- применение tasks через `apply`;
- update существующих artifacts;
- verify implementation;
- merge delta specs в master при archive;
- проверку структуры change.

Отдельный initiative-level слой:

```text
target-spec.md
delivery-plan.md
```

не является first-class OpenSpec artifact.

Мы стандартизируем его через:

- directory convention;
- templates;
- Explore prompts;
- Requirement IDs;
- mapping на OpenSpec changes.

---

# 2. Требуемая структура

Создайте:

```text
docs/
└── initiatives/
    ├── _templates/
    │   ├── target-spec.template.md
    │   └── delivery-plan.template.md
    ├── active/
    └── archive/

openspec/
├── config.yaml
├── schemas/
│   └── team-sdd/
├── specs/
└── changes/
```

---

# 3. Инициализация OpenSpec

Из корня проекта:

```bash
openspec init
```

Если OpenSpec уже подключён, этот шаг пропускается.

Проверить:

```bash
openspec config list
openspec schemas
openspec list --specs
```

---

# 4. Включить нужные workflows

Default `core` содержит:

```text
explore
propose
apply
update
sync
archive
```

Для нашего flow дополнительно нужны:

```text
new
continue
verify
```

Запустить интерактивную настройку:

```bash
openspec config profile
```

Выбрать workflow:

```text
[x] explore
[x] new
[x] continue
[x] propose        # можно оставить для маленьких ad-hoc changes
[x] apply
[x] update
[x] verify
[x] sync
[x] archive
```

Delivery рекомендуется:

```text
both
```

чтобы были и skills, и slash commands.

После изменения profile применить настройки в проекте:

```bash
openspec update
```

Проверить:

```bash
openspec config list
```

> Profile является machine-level настройкой. Каждый разработчик должен иметь совместимый workflow set, а project integration files должны быть обновлены и закоммичены.

---

# 5. Почему нужен custom schema

Built-in `spec-driven` имеет зависимости:

```text
             ┌─ specs ──┐
proposal ────┤          ├── tasks
             └─ design ─┘
```

То есть specs и design оба становятся available после proposal.

Наш процесс требует:

```text
proposal
   ↓
specs
   ↓
behavioral review
   ↓
design
   ↓
tasks
```

Чтобы schema технически поддерживала этот порядок, делаем project-local fork.

---

# 6. Fork schema

В корне проекта:

```bash
openspec schema fork spec-driven team-sdd
```

Получится:

```text
openspec/schemas/team-sdd/
├── schema.yaml
└── templates/
    ├── proposal.md
    ├── spec.md
    ├── design.md
    └── tasks.md
```

Проверить:

```bash
openspec schema which team-sdd
```

---

# 7. Изменить dependency graph

Открыть:

```text
openspec/schemas/team-sdd/schema.yaml
```

И установить концептуально такую зависимость:

```yaml
name: team-sdd
version: 1
description: Team SDD workflow with behavioral review before technical design

artifacts:
  - id: proposal
    generates: proposal.md
    description: Why and scope of one delivery change
    template: proposal.md
    requires: []

  - id: specs
    generates: "specs/**/*.md"
    description: Behavioral delta specs
    template: spec.md
    requires:
      - proposal

  - id: design
    generates: design.md
    description: Technical design for approved behavioral specs
    template: design.md
    requires:
      - specs

  - id: tasks
    generates: tasks.md
    description: Implementation checklist
    template: tasks.md
    requires:
      - specs
      - design

apply:
  requires:
    - tasks
  tracks: tasks.md
```

## Важно

При fork built-in schema содержит более подробные `instruction` и условия для conditional design.

**Не заменяйте весь файл этим сокращённым примером вслепую.**

Правильнее:

1. сохранить built-in instructions из fork;
2. изменить прежде всего dependency для design:

```yaml
requires:
  - specs
```

3. сохранить `tasks.requires`:

```yaml
- specs
- design
```

---

# 8. Валидировать schema

После каждого изменения:

```bash
openspec schema validate team-sdd
```

Если есть dependency cycle, missing template или invalid reference — исправить до использования.

---

# 9. Сделать schema default для проекта

В:

```text
openspec/config.yaml
```

указать:

```yaml
schema: team-sdd
```

Проверить:

```bash
openspec schemas
openspec schema which team-sdd
```

Новые changes будут использовать `team-sdd`.

Старые changes сохраняют schema, с которой были созданы.

---

# 10. Настроить project context

Рекомендуемый базовый `openspec/config.yaml`:

```yaml
schema: team-sdd

context: |
  openspec/specs represents CURRENT implemented system behavior.
  It must not describe unimplemented target-state functionality.

  Large initiatives are described outside OpenSpec changes:
  docs/initiatives/active/<initiative>/target-spec.md
  docs/initiatives/active/<initiative>/delivery-plan.md

  One OpenSpec change represents one independently deliverable increment.

  Target specification describes WHERE the product is going.
  Delivery plan describes HOW the initiative is split into changes.
  OpenSpec change describes WHAT is being delivered now.
  Master specs describe WHAT is already implemented.

  Business-visible behavior is owned by analysts/product.
  Technical implementation is owned by developers.
  Technical observable contracts may be developer-owned specs.

  Before introducing a new capability, inspect existing master specs.
  Reuse an existing exact capability path when the behavior belongs there.
  Avoid near-duplicate capabilities.

  If observable behavior changes during implementation, update specs before
  treating implementation as complete.

  Prefer archive-driven master updates.
  Do not use sync as the normal delivery path.

rules:
  proposal:
    - Keep one change small enough to be independently implemented, verified and archived.
    - Reference source initiative requirement IDs when the change originates from a target spec.
    - State in-scope and out-of-scope behavior.
    - Inspect existing specs before declaring a new capability.
    - Reuse exact existing capability paths when appropriate.
    - Do not include implementation design.

  specs:
    - Describe externally observable behavior, contracts and constraints.
    - Every important behavioral rule must have testable scenarios.
    - Do not include internal classes, database tables or framework choices.
    - Use ADDED only for requirements that do not already exist.
    - Use MODIFIED for an existing requirement whose behavioral contract changes.
    - Preserve the exact existing capability path for modified capabilities.
    - Do not create a new capability only because the change name is different.

  design:
    - Design only after behavioral specs are available.
    - Do not silently redefine product behavior.
    - Cover APIs, persistence, integration, transactions, compatibility and migrations where relevant.
    - If implementation constraints require behavioral changes, update the specs.

  tasks:
    - Tasks must map to approved specs or design.
    - Use markdown checkboxes for every actionable task.
    - Include relevant automated tests for behavioral scenarios.
    - Do not introduce new product behavior only in tasks.

operations:
  apply:
    guidance:
      - Treat specs as the behavioral contract.
      - If implementation reveals ambiguous or changed behavior, stop and revise planning artifacts.
      - Mark task checkboxes complete only when the work is actually implemented.

  archive:
    guidance:
      - Archive only after implementation has been verified against the final change artifacts.
      - Sync delta specs into master specs unless the change intentionally has no spec-level behavior.
      - Confirm the delta capability paths are correct before updating master specs.
```

---

# 11. Важное ограничение `config.yaml`

`context` применяется широко.

`rules` применяются к созданию конкретного artifact.

`operations` относятся к apply/archive.

`verify` не получает artifact rules как инструкцию; verify проверяет уже созданные artifacts против implementation.

Поэтому behavioral/capability gates должны быть отражены в artifacts до verify.

---

# 12. Настроить initiative templates

Создать:

```text
docs/initiatives/_templates/target-spec.template.md
```

с содержанием:

```markdown
# <Module> — Target Specification

## 1. Goal

## 2. Context

## 3. Target capabilities

### <CAPABILITY>

#### <REQ-ID> <Requirement name>

<Observable behavior>

##### Scenario: <name>
- WHEN ...
- THEN ...

## 4. Cross-cutting constraints

## 5. Out of scope

## 6. Open questions

## 7. Terminology
```

Создать:

```text
docs/initiatives/_templates/delivery-plan.template.md
```

с содержанием:

```markdown
# <Module> — Delivery Plan

Source:
`docs/initiatives/active/<module>/target-spec.md`

## 1. Capability model

| Capability path | Purpose | Existing/New |
|---|---|---|

## 2. Planned changes

### <change-name>

Status: Planned

Requirement IDs:
- ...

Capability paths:
- ...

Observable outcome:
- ...

Depends on:
- ...

Out of scope:
- ...

Archived change:
- n/a

## 3. Coverage

| Requirement ID | Change | Status |
|---|---|---|
```

---

# 13. Standard prompt: генерация Target Spec

Аналитик использует:

```text
/opsx:explore

Нужно сформировать целевую функциональную спецификацию модуля <MODULE>.

Это initiative-level target-state, а не один OpenSpec change.

Прочитай существующие openspec/specs и релевантный код, чтобы не
противоречить уже реализованным возможностям.

Сформируй:
- цель;
- target capabilities;
- behavioral requirements;
- стабильные Requirement IDs;
- testable scenarios;
- edge cases;
- constraints;
- out of scope;
- open questions;
- терминологию.

Не создавай proposal/design/tasks и не декомпозируй пока на implementation tasks.

После review результат будет сохранён в:
docs/initiatives/active/<module>/target-spec.md
```

После Explore попросить agent сохранить согласованный результат в этот файл.

---

# 14. Standard prompt: декомпозиция

Developer использует:

```text
/opsx:explore

Прочитай:
- docs/initiatives/active/<module>/target-spec.md
- openspec/specs/**
- релевантный код.

Разбей target specification на небольшие независимо поставляемые
OpenSpec changes.

Каждый change должен:
- давать законченное observable behavior;
- отдельно реализовываться;
- отдельно тестироваться;
- отдельно архивироваться;
- по возможности помещаться в одну delivery iteration.

Для каждого предложи:
- kebab-case change name;
- observable outcome;
- Requirement IDs;
- dependencies;
- out of scope;
- предполагаемые capability paths;
- будет capability новой или существующей.

Отдельно предложи устойчивую capability model для master specs.

Не создавай changes и не пиши код.
```

После review сохранить результат:

```text
docs/initiatives/active/<module>/delivery-plan.md
```

---

# 15. Capability Review checklist

Перед созданием каждого child change проверить:

```bash
openspec list --specs
```

Для похожих capabilities:

```bash
openspec show "<capability-path>" --type spec
```

Checklist:

```text
[ ] Existing capability найдена/исключена
[ ] New capability действительно новая
[ ] Capability path устойчивый, а не task-name
[ ] Planned change знает Requirement IDs
[ ] Нет near-duplicate paths
```

---

# 16. Запуск одного delivery change

```text
/opsx:new add-hdfs-copy-file
```

или CLI:

```bash
openspec new change add-hdfs-copy-file \
  --goal "Implement HDFS-FILE-001: copy file"
```

Проверить:

```bash
openspec status --change add-hdfs-copy-file
```

---

# 17. Генерация proposal

Аналитик:

```text
/opsx:continue add-hdfs-copy-file
```

После этого проверить:

```text
proposal.md
```

Особенно:

```text
New Capabilities
Modified Capabilities
Requirement IDs
Out of scope
```

---

# 18. Генерация delta specs

Следующий вызов:

```text
/opsx:continue add-hdfs-copy-file
```

Custom schema должна сделать следующим artifact `specs`.

Проверить:

```text
openspec/changes/add-hdfs-copy-file/specs/<capability-path>/spec.md
```

---

# 19. Capability + Behavioral review gate

До design:

```bash
openspec validate add-hdfs-copy-file
```

При замечаниях:

```text
/opsx:update add-hdfs-copy-file
```

Типовой запрос:

```text
Проведи coherence review change.
Проверь особенно:
- соответствие target Requirement IDs;
- корректность capability paths;
- отсутствие near-duplicate capabilities;
- testability scenarios;
- отсутствие implementation details в specs.
```

---

# 20. Генерация design

Developer:

```text
/opsx:continue add-hdfs-copy-file
```

Благодаря `team-sdd` schema design не должен быть ready до specs.

---

# 21. Генерация tasks

Developer:

```text
/opsx:continue add-hdfs-copy-file
```

Проверить:

```bash
openspec status --change add-hdfs-copy-file
openspec validate add-hdfs-copy-file
```

---

# 22. Apply

```text
/opsx:apply add-hdfs-copy-file
```

Во время apply:

### Если изменился implementation only

```text
/opsx:update add-hdfs-copy-file
```

обновить design/tasks.

### Если изменилось behavior

1. скорректировать target-spec, если изменение влияет на initiative target;
2. `/opsx:update`;
3. обновить specs;
4. обновить design/tasks;
5. продолжить apply.

---

# 23. Verify

```text
/opsx:verify add-hdfs-copy-file
```

Verify report-only.

Если есть проблемы:

```text
code fix
или
/opsx:update
```

после чего снова:

```text
/opsx:verify
```

---

# 24. Archive

После готовности:

```text
/opsx:archive add-hdfs-copy-file
```

Archive должен обновить:

```text
openspec/specs/<capability-path>/spec.md
```

из delta:

```text
openspec/changes/add-hdfs-copy-file/specs/<capability-path>/spec.md
```

и переместить change:

```text
openspec/changes/archive/YYYY-MM-DD-add-hdfs-copy-file/
```

---

# 25. Почему change name не управляет master path

Неправильная ментальная модель:

```text
add-hdfs-copy-file
→ hdfs/copy-file
```

Правильная:

```text
change name:
add-hdfs-copy-file

delta path:
specs/hdfs/file-operations/spec.md

master target:
openspec/specs/hdfs/file-operations/spec.md
```

Archive применяет delta operations к этому path.

---

# 26. Как выбирать ADDED vs MODIFIED

## Capability существует, requirement новый

```text
Capability: Modified
Requirement delta: ADDED
```

## Requirement уже существует и меняется behavior

```text
Capability: Modified
Requirement delta: MODIFIED
```

## Capability новая

```text
Capability: New
Requirement delta: ADDED
```

---

# 27. Sync policy

Командное правило:

```text
sync = exception
archive = default master update
```

Не включать sync в happy path.

Использовать:

```text
/opsx:sync <change>
```

только для осознанного long-running сценария.

---

# 28. Pure technical change

Создать change, затем в:

```text
openspec/changes/<change>/.openspec.yaml
```

установить:

```yaml
skip_specs: true
```

Только если действительно нет spec-level behavior.

---

# 29. CI / pre-merge automation

Минимальный CI-check:

```bash
openspec validate --all --strict
```

Если `--strict` слишком жёсткий для старого репозитория, начать с:

```bash
openspec validate --all
```

Полезно отдельно валидировать active changes:

```bash
openspec validate --changes
```

и master specs:

```bash
openspec validate --specs
```

---

# 30. Recommended pull request gates

Для PR с planning artifacts:

```text
[ ] target Requirement IDs указаны
[ ] capability mapping проверен
[ ] delta specs testable
[ ] no implementation detail leakage
[ ] openspec validate проходит
```

Для implementation PR:

```text
[ ] tasks выполнены
[ ] tests добавлены
[ ] design актуален
[ ] /opsx:verify выполнен
[ ] archive-ready
```

---

# 31. Что коммитить

Коммитить в repository:

```text
openspec/config.yaml
openspec/schemas/team-sdd/**
workflow integration files, созданные openspec init/update
docs/initiatives/_templates/**
docs/initiatives/active/**
openspec/specs/**
openspec/changes/**
```

Не полагаться только на machine-level profile как на единственный источник процесса — profile у каждого разработчика локальный.

---

# 32. Обновление OpenSpec

Периодически:

```bash
openspec update
```

Но учитывать:

> `openspec update` обновляет installed skills/commands, но не перезаписывает project-local forked schemas.

Поэтому `team-sdd` является вашей поддерживаемой копией.

При заметных изменениях built-in schema:

1. fork свежего `spec-driven` под временным именем;
2. сравнить с `team-sdd`;
3. перенести upstream improvements;
4. validate;
5. удалить временную копию.

---

# 33. Как проверить, какая schema реально используется

```bash
openspec schema which team-sdd
```

Для active change schema зафиксирована в:

```text
openspec/changes/<change>/.openspec.yaml
```

Например:

```yaml
schema: team-sdd
created: 2026-10-01
```

Если project default schema потом изменится, уже созданный change сохраняет старую schema.

---

# 34. Опциональная автоматизация initiative metadata

`.openspec.yaml` поддерживает поле:

```yaml
initiative:
  store: <store-id>
  id: <initiative-id>
```

Но текущие OpenSpec-команды эту связь не используют.

Поэтому **не строить orchestration процесса на `initiative` metadata**.

Можно использовать только как дополнительную машиночитаемую ссылку, если у команды есть свой tooling.

---

# 35. Recommended team command cheat sheet

## Initiative planning

```text
/opsx:explore
```

## Начать delivery change

```text
/opsx:new <change>
```

## Следующий planning artifact

```text
/opsx:continue <change>
```

## Пересмотреть artifacts

```text
/opsx:update <change>
```

## Реализовать

```text
/opsx:apply <change>
```

## Проверить implementation

```text
/opsx:verify <change>
```

## Закрыть и обновить master

```text
/opsx:archive <change>
```

## Исключительно: sync до archive

```text
/opsx:sync <change>
```

---

# 36. Happy-path script для человека

```text
# initiative
/opsx:explore
→ target-spec.md

/opsx:explore
→ delivery-plan.md

# один child change
/opsx:new add-hdfs-copy-file

# analyst
/opsx:continue add-hdfs-copy-file
→ proposal

/opsx:continue add-hdfs-copy-file
→ specs

# analyst + developer
review capability + behavior

# developer
/opsx:continue add-hdfs-copy-file
→ design

/opsx:continue add-hdfs-copy-file
→ tasks

/opsx:apply add-hdfs-copy-file

/opsx:verify add-hdfs-copy-file

/opsx:archive add-hdfs-copy-file

# initiative tracking
update delivery-plan.md

# repeat
```

---

# 37. Настройка считается завершённой, если

- [ ] OpenSpec инициализирован;
- [ ] workflows `new`, `continue`, `verify` включены;
- [ ] `openspec update` выполнен;
- [ ] schema `team-sdd` создана;
- [ ] `design.requires = specs`;
- [ ] schema validation проходит;
- [ ] project `schema: team-sdd`;
- [ ] project context/rules добавлены;
- [ ] initiative templates созданы;
- [ ] capability review принят как gate;
- [ ] archive является default master-sync path;
- [ ] CI валидирует OpenSpec;
- [ ] команда понимает разницу target / change / master.

---

# 38. Источники OpenSpec

- https://openspec.dev/docs/setup
- https://openspec.dev/docs/profiles
- https://openspec.dev/docs/skills
- https://openspec.dev/docs/schemas/spec-driven
- https://openspec.dev/docs/customize-schemas
- https://openspec.dev/docs/schemas/schema-yaml
- https://openspec.dev/docs/configuration/config-yaml
- https://openspec.dev/docs/configuration/change-metadata
- https://openspec.dev/docs/cli

> Schema APIs помечены OpenSpec как experimental, поэтому после обновлений CLI стоит перепроверять `team-sdd` через `openspec schema validate team-sdd`.
