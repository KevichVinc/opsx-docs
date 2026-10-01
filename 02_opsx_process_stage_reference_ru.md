# OpenSpec / OPSX SDD — подробное описание процесса по этапам

> Этот документ описывает **каждый этап процесса**, его владельца, входы, выходы, место хранения, команды OPSX, настройки, review-gates и критерии завершения.

---

# 1. Модель ответственности

| Область | Primary owner | Обязательный reviewer |
|---|---|---|
| Business target | Аналитик / Product | Developer для feasibility |
| Delivery decomposition | Developer + Analyst | Команда |
| Capability model | Developer / Architect + Analyst | Команда |
| `proposal.md` | Аналитик / Product | Developer |
| Behavioral delta specs | Аналитик | Developer |
| Technical/API/event contract specs | Developer / API owner | Analyst/consumer owner при необходимости |
| `design.md` | Developer / Tech Lead | Developer reviewers |
| `tasks.md` | Developer | Developer/Lead |
| Implementation | Developer | Code review / QA |
| Verification | Developer + QA | Команда |
| Archive | Developer / change owner | по командным правилам |
| Master specs | Команда | owners затронутых capabilities |

---

# 2. Структура репозитория

Рекомендуемая структура:

```text
repo/
├── docs/
│   └── initiatives/
│       ├── active/
│       │   └── hdfs/
│       │       ├── target-spec.md
│       │       └── delivery-plan.md
│       └── archive/
│
├── openspec/
│   ├── config.yaml
│   ├── schemas/
│   │   └── team-sdd/
│   │       ├── schema.yaml
│   │       └── templates/
│   ├── specs/
│   └── changes/
│       └── archive/
│
└── src/
```

---

# Этап 0. Инициализация OpenSpec в проекте

## Цель

Подключить OpenSpec/OPSX и установить нужные workflow-команды.

## Ответственный

Tech Lead / Developer, отвечающий за процесс.

## Команды

```bash
openspec init
```

После настройки profile:

```bash
openspec update
```

## Где хранится

OpenSpec создаёт:

```text
openspec/config.yaml
openspec/specs/
openspec/changes/
```

Workflow integration files создаются для выбранного AI-tool.

## Что должно быть настроено

Нужны workflow:

```text
explore
new
continue
apply
update
verify
sync
archive
```

`propose` можно оставить для ad-hoc небольших задач, но не использовать как основной stage-gated flow.

## Done

- команды доступны;
- `openspec/config.yaml` закоммичен;
- команда использует одинаковую project schema.

---

# Этап 1. Intake инициативы

## Цель

Зафиксировать, что появилась новая крупная продуктовая задача/модуль, которая не помещается в один change.

## Пример

```text
HDFS module:
- navigation
- copy file
- rename file
- copy directory
- rename directory
- move directory
```

## Ответственный

Аналитик / Product.

## Где хранится

До формализации это может быть Jira/Confluence/issue/brief.

После начала работы создаётся:

```text
docs/initiatives/active/hdfs/
```

## Done

Понятна цель модуля и есть исходный scope для проработки.

---

# Этап 2. Исследование target-state

## Цель

Сформировать целевую функциональную модель модуля, не смешивая её с планом реализации.

## Ответственный

Аналитик.

Developer подключается для проверки ограничений и существующего контекста.

## OPSX

```text
/opsx:explore
```

## Входы

- бизнес-идея;
- существующие master specs;
- при необходимости текущий код;
- ограничения;
- интеграционные договорённости.

## Что просить у Explore

- capabilities;
- requirements;
- scenarios;
- edge cases;
- errors;
- ограничения;
- out of scope;
- open questions.

## Что НЕ просить

- создавать один большой OpenSpec change;
- генерировать tasks всего модуля;
- заранее фиксировать технический design всего модуля.

## Выход

```text
docs/initiatives/active/<module>/target-spec.md
```

## Done

Target-state можно читать независимо от delivery plan, и он отвечает на вопрос:

> Что модуль должен уметь после завершения инициативы?

---

# Этап 3. Нормализация требований target-spec

## Цель

Сделать глобальную spec пригодной для traceability и декомпозиции.

## Ответственный

Аналитик.

## Обязательные элементы

### Stable Requirement IDs

Например:

```text
HDFS-NAV-001
HDFS-FILE-001
HDFS-FILE-002
HDFS-DIR-001
```

### Scenarios

Каждое важное правило должно быть проверяемым.

### Out of Scope

Чтобы decomposition не расширял инициативу случайно.

### Open Questions

Не прятать неопределённость в prose.

## Где хранится

`target-spec.md`.

## Done

Каждое значимое target requirement имеет стабильный ID и понятный смысл.

---

# Этап 4. Target Spec Review

## Цель

Не передавать в декомпозицию логически противоречивый target.

## Ответственный

Аналитик — за бизнес-смысл.

Developer — за техническую выполнимость и выявление скрытых ограничений.

## Проверяем

- scope;
- терминологию;
- behavior;
- edge cases;
- конфликтующие requirements;
- зависимости от уже существующих возможностей;
- что является target, а что уже current.

## Gate

Если вопрос меняет observable behavior — target-spec должен быть исправлен до decomposition.

## Done

Target spec принят как baseline для planning.

---

# Этап 5. Декомпозиция initiative

## Цель

Разбить target-state на independently deliverable OpenSpec changes.

## Ответственный

Developer + Analyst.

AI делает первый вариант, люди принимают границы.

## OPSX

```text
/opsx:explore
```

## Explore должен читать

```text
docs/initiatives/active/<module>/target-spec.md
openspec/specs/**
текущий код
```

## Для каждого предлагаемого change определить

- change name;
- observable outcome;
- Requirement IDs;
- зависимости;
- out of scope;
- предполагаемый capability-path;
- порядок реализации.

## Критерии размера

Change должен:

- давать законченный результат;
- быть separately testable;
- быть separately archivable;
- по возможности помещаться в одну итерацию.

## Выход

```text
docs/initiatives/active/<module>/delivery-plan.md
```

---

# Этап 6. Формирование Capability Model

## Цель

Заранее определить устойчивую структуру master specs.

## Ответственный

Developer / Architect совместно с аналитиком.

## Почему это отдельный этап

Без него каждый новый change может создать near-duplicate capability.

Пример плохой структуры:

```text
hdfs/copy-file
hdfs/rename-file
hdfs/copy-folder
```

Предпочтительная структура:

```text
hdfs/navigation
hdfs/file-operations
hdfs/directory-operations
```

## Проверка существующих specs

```bash
openspec list --specs
openspec show "hdfs/file-operations" --type spec
```

## Где хранится

Capability mapping фиксируется в `delivery-plan.md`.

## Done

Каждый planned change знает, к какому capability-path он относится или какую новую capability создаёт.

---

# Этап 7. Decomposition Review

## Цель

Принять план поставки.

## Ответственный

Developer + Analyst.

## Проверяем

- change не слишком большой;
- нет pure technical slices вместо behavioral increments;
- dependencies разумны;
- каждый target Requirement ID покрывается;
- нет requirements без будущего change;
- нет change без target outcome;
- capability paths не дублируют существующие.

## Done

Delivery plan считается рабочей очередью будущих changes.

---

# Этап 8. Создание одного OpenSpec change

## Цель

Создать container для одного delivery increment.

## Ответственный

Change owner, обычно Developer.

## OPSX

```text
/opsx:new add-hdfs-copy-file
```

## CLI-эквивалент

```bash
openspec new change add-hdfs-copy-file
```

## Где хранится

```text
openspec/changes/add-hdfs-copy-file/
└── .openspec.yaml
```

## Metadata

Рекомендуется указывать goal и affected areas, если команда использует их.

`initiative` metadata можно использовать только как дополнительную справочную связь: текущие OpenSpec-команды её не используют для orchestration.

## Done

Change создан, artifacts ещё не сгенерированы.

---

# Этап 9. Создание `proposal.md`

## Цель

Зафиксировать WHY, scope и affected capabilities.

## Ответственный

Аналитик / Product.

## OPSX

```text
/opsx:continue add-hdfs-copy-file
```

При строгой custom schema первым artifact будет `proposal.md`.

## Где хранится

```text
openspec/changes/<change>/proposal.md
```

## Proposal должен содержать

- Why;
- What Changes;
- in scope;
- out of scope;
- New Capabilities;
- Modified Capabilities;
- ссылки/Requirement IDs из target-spec.

## Не должен содержать

- классы;
- таблицы;
- framework choices;
- подробный implementation plan.

## Done

Developer понимает, зачем существует change и какие capabilities он затрагивает.

---

# Этап 10. Создание delta specs

## Цель

Зафиксировать WHAT — observable behavioral delta.

## Ответственный

Аналитик для business behavior.

Developer для developer-owned technical contracts.

## OPSX

```text
/opsx:continue <change>
```

## Где хранится

```text
openspec/changes/<change>/specs/<capability-path>/spec.md
```

## Delta operations

```text
ADDED
MODIFIED
REMOVED
RENAMED
```

## Правило MODIFIED

Если меняется существующий requirement, delta должна содержать его полную обновлённую версию, а не только одну изменённую строку.

## Done

Spec может быть превращён в тесты/проверки без угадывания продуктового поведения.

---

# Этап 11. Capability Review Gate

## Цель

Проверить, что delta будет обновлять правильный master spec.

## Ответственный

Developer / Architect + Analyst.

## Проверяем

Например change:

```text
add-hdfs-copy-file
```

может относиться к:

```text
hdfs/file-operations
```

а не создавать:

```text
hdfs/copy-file
```

## Источник истины

```text
changes/<change>/specs/<capability-path>/spec.md
```

именно этот path определяет target:

```text
openspec/specs/<capability-path>/spec.md
```

## Done

Нет случайно созданной duplicate capability.

---

# Этап 12. Behavioral Review Gate

## Цель

Убедиться, что Developer одинаково с аналитиком понимает требование.

## Ответственный

Developer — review.

Аналитик — решение по бизнес-неоднозначностям.

## Примеры вопросов

- с какого момента считается timeout;
- какие статусы разрешены;
- что происходит при конфликте имени;
- кто может выполнять операцию;
- что считается успешным завершением;
- как выглядит observable error.

## Если нашли проблему

```text
/opsx:update <change>
```

Обновляются существующие artifacts.

## Done

Нет открытых вопросов, способных изменить observable behavior.

---

# Этап 13. Создание `design.md`

## Цель

Описать HOW.

## Ответственный

Developer / Tech Lead.

## OPSX

```text
/opsx:continue <change>
```

## Где хранится

```text
openspec/changes/<change>/design.md
```

## Содержит

- архитектурный подход;
- компоненты;
- API implementation;
- persistence;
- integrations;
- transactions;
- events;
- migration;
- compatibility;
- trade-offs.

## Не должен менять product behavior молча

Если design требует другого behavior — сначала update spec.

## Done

Команда понимает, как реализовать approved behavior.

---

# Этап 14. Создание `tasks.md`

## Цель

Превратить spec + design в исполнимый implementation checklist.

## Ответственный

Developer.

## OPSX

```text
/opsx:continue <change>
```

## Где хранится

```text
openspec/changes/<change>/tasks.md
```

## Правила

Tasks должны быть чекбоксами:

```markdown
- [ ] 1.1 ...
- [ ] 1.2 ...
```

Иначе OpenSpec не сможет корректно считать progress.

## Не допускается

Новый business requirement впервые появляется только в task.

## Done

Change ready for apply.

---

# Этап 15. Definition of Ready

Перед apply:

- [ ] proposal согласован;
- [ ] capability mapping согласован;
- [ ] delta specs согласованы;
- [ ] behavioral questions закрыты;
- [ ] design готов, если требуется;
- [ ] tasks готовы;
- [ ] OpenSpec change валиден.

Дополнительно полезно:

```bash
openspec validate <change>
```

---

# Этап 16. Apply

## Цель

Реализовать tasks.

## Ответственный

Developer.

## OPSX

```text
/opsx:apply <change>
```

## Что изменяет

- project code;
- tests;
- `tasks.md` — completed checkboxes.

## Что не должно происходить

AI не должен молча переопределять requirements.

## При блокере

Остановиться и решить, требуется ли `update`.

---

# Этап 17. Update loop во время реализации

## Цель

Сохранять согласованность spec/design/tasks/code.

## Ответственный

Зависит от типа изменения.

### Меняется behavior

Owner: Analyst + Developer review.

Обновить:

```text
specs
при необходимости target-spec
design
tasks
```

### Меняется implementation only

Owner: Developer.

Обновить:

```text
design
tasks
```

## OPSX

```text
/opsx:update <change>
```

## Важное ограничение

`update` редактирует уже существующие planning artifacts. Он не является заменой `continue` для ещё не созданного artifact.

---

# Этап 18. Verify

## Цель

Сравнить implementation и change artifacts.

## Ответственный

Developer + QA.

## OPSX

```text
/opsx:verify <change>
```

## Результат

Report по:

- Completeness;
- Correctness;
- Coherence;
- archive readiness.

## Что изменяет

Ничего.

Verify — report-only.

## Done

Нет CRITICAL проблем; команда согласна с archive readiness.

---

# Этап 19. Definition of Done

- [ ] tasks выполнены;
- [ ] implementation соответствует specs;
- [ ] tests покрывают существенные scenarios;
- [ ] design соответствует фактической реализации;
- [ ] capability paths корректны;
- [ ] target/delivery plan не противоречат выполненному change;
- [ ] verify пройден.

---

# Этап 20. Archive

## Цель

Закрыть change и превратить delta в новое current-state.

## Ответственный

Change owner / Developer.

## OPSX

```text
/opsx:archive <change>
```

## Что происходит

1. OpenSpec проверяет change.
2. Показывает spec updates.
3. Outstanding deltas синхронизируются с main specs.
4. Change переносится:

```text
openspec/changes/archive/YYYY-MM-DD-<change>/
```

## Master update

Delta:

```text
openspec/changes/<change>/specs/<capability-path>/spec.md
```

обновляет:

```text
openspec/specs/<capability-path>/spec.md
```

## Done

Master specs отражают реализованное поведение.

---

# Этап 21. Обновление Delivery Plan

## Цель

Сохранить traceability initiative → change → master.

## Ответственный

Developer / Initiative owner.

## Где хранится

```text
docs/initiatives/active/<module>/delivery-plan.md
```

## Обновить

- change status → `Shipped`;
- Requirement IDs → `Implemented`;
- archived path;
- зависимости;
- следующий change.

## Done

Любой участник может понять, что из target уже реализовано.

---

# Этап 22. Следующая итерация

Создаётся **новый** change:

```text
/opsx:new add-hdfs-rename-file
```

Новый change должен читать уже обновлённые master specs.

Это позволяет следующей итерации планироваться поверх реального current state.

---

# Этап 23. Изменение target-spec до реализации requirement

Если requirement ещё не имеет active change:

- обновить `target-spec.md`;
- обновить `delivery-plan.md`;
- OpenSpec change не нужен.

---

# Этап 24. Изменение target-spec при active change

Если requirement уже входит в active change:

1. обновить `target-spec.md`;
2. `/opsx:update <change>`;
3. обновить delta specs;
4. обновить design/tasks;
5. повторить review.

---

# Этап 25. Изменение уже archived behavior

Если behavior уже находится в master specs:

1. обновить target-state;
2. создать **новый** OpenSpec change;
3. сделать MODIFIED delta;
4. apply;
5. verify;
6. archive.

Нельзя переписывать master как будущее состояние.

---

# Этап 26. Sync — исключительный путь

## OPSX

```text
/opsx:sync <change>
```

## Когда допустимо

- long-running change;
- осознанная необходимость синхронизировать часть specs раньше archive.

## Почему не default

Change остаётся active, а master уже меняется.

Это усложняет понимание:

```text
что уже shipped?
что только planned?
```

Для небольших independently archivable changes sync не нужен.

---

# Этап 27. Pure technical change

Если нет spec-level behavior:

```yaml
skip_specs: true
```

Примеры:

- refactor;
- tooling;
- internal infra;
- docs-only.

Не создавать fake requirement только ради validation.

---

# Этап 28. Spec correction

Если master содержит документальную ошибку, а реальное intended behavior не меняется:

```text
direct correction PR
```

с явным маркером:

```text
Spec correction only.
No behavior change.
```

Если есть сомнение, что intended behavior — именно такой, сначала провести reconciliation.

---

# Этап 29. Закрытие initiative

Когда все target Requirement IDs реализованы или сознательно исключены:

1. проверить target-spec против master specs;
2. обновить delivery plan final status;
3. перенести:

```text
docs/initiatives/active/hdfs/
```

в:

```text
docs/initiatives/archive/hdfs/
```

Master specs продолжают жить как актуальная системная документация.

---

# Этап 30. Что настраивается где

| Что | Где |
|---|---|
| Набор OPSX workflows | machine-level OpenSpec profile |
| Delivery: skills/commands/both | machine-level OpenSpec profile |
| Project schema | `openspec/config.yaml` |
| Общие правила проекта | `openspec/config.yaml -> context` |
| Правила конкретных artifacts | `openspec/config.yaml -> rules` |
| Guidance для apply/archive | `openspec/config.yaml -> operations` |
| Порядок artifacts | `openspec/schemas/<name>/schema.yaml` |
| Шаблон proposal/spec/design/tasks | `openspec/schemas/<name>/templates/` |
| Change schema/goal/skip_specs | `openspec/changes/<change>/.openspec.yaml` |
| Target module spec | `docs/initiatives/.../target-spec.md` |
| Delivery decomposition | `docs/initiatives/.../delivery-plan.md` |

---

# Этап 31. Источники OpenSpec

- https://openspec.dev/docs/skills
- https://openspec.dev/docs/profiles
- https://openspec.dev/docs/schemas/spec-driven
- https://openspec.dev/docs/configuration/config-yaml
- https://openspec.dev/docs/configuration/change-metadata
- https://openspec.dev/docs/customize-schemas
- https://openspec.dev/docs/cli
