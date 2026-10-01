# Полная карта flow OpenSpec / OPSX для SDD

> Версия для командного обсуждения.  
> Цель: показать полный путь от большой продуктовой идеи до инкрементальной реализации, архивирования change и обновления master specs.

---

## 0. Термины

В этом документе используются четыре уровня:

| Уровень | Что это | Где хранится |
|---|---|---|
| **Initiative / Module Target Spec** | Полное целевое поведение модуля | `docs/initiatives/active/<module>/target-spec.md` |
| **Delivery Plan** | Декомпозиция target-state на поставляемые OpenSpec changes | `docs/initiatives/active/<module>/delivery-plan.md` |
| **OpenSpec Change** | Один независимо поставляемый increment | `openspec/changes/<change>/` |
| **Master Specs** | Текущее реализованное поведение системы | `openspec/specs/**/spec.md` |

Главный принцип:

```text
Target Spec   = куда хотим прийти
Delivery Plan = как будем туда идти
OpenSpec Change = что поставляем сейчас
Master Specs  = что уже действительно поставлено
```

---

# 1. End-to-end карта

```mermaid
flowchart TD

    START["Бизнес-идея / Story / Initiative"]

    subgraph INITIATIVE["Уровень инициативы / модуля"]
        E1["/opsx:explore<br/>Проработка target-state"]
        TARGET["target-spec.md<br/>Полное целевое поведение модуля"]
        TR["Target Spec Review<br/>Аналитик + Product + Dev при необходимости"]

        E2["/opsx:explore<br/>Декомпозиция на delivery slices"]
        PLAN["delivery-plan.md<br/>Changes + Requirement IDs + Dependencies"]
        CAPMODEL["Capability Model<br/>Куда должны попадать требования в master specs"]
        DR["Decomposition Review<br/>Аналитик + Developer"]
    end

    subgraph CHANGE["Один OpenSpec Change"]
        NEW["/opsx:new <change>"]
        PROP["/opsx:continue<br/>proposal.md"]
        SPEC["/opsx:continue<br/>delta specs"]
        CAPGATE{"Capability Review<br/>правильный capability-path?"}
        BEHGATE{"Behavioral Review<br/>поведение однозначно?"}
        DESIGN["/opsx:continue<br/>design.md"]
        TASKS["/opsx:continue<br/>tasks.md"]
        APPLY["/opsx:apply"]
        CODE["Code + Tests"]
        VERIFY["/opsx:verify"]
        READY{"Archive ready?"}
        UPDATE["/opsx:update<br/>актуализация существующих artifacts"]
        ARCHIVE["/opsx:archive"]
    end

    MASTER["openspec/specs/**<br/>CURRENT / AS-BUILT"]
    HISTORY["openspec/changes/archive/**<br/>История изменений"]
    STATUS["Обновить delivery-plan.md<br/>status + link на archived change"]
    NEXT{"Есть ещё planned changes?"}
    DONE["Initiative завершена<br/>target приблизительно совпадает с master"]

    START --> E1
    E1 --> TARGET
    TARGET --> TR
    TR -->|Есть вопросы| E1
    TR -->|Принято| E2

    E2 --> PLAN
    E2 --> CAPMODEL
    PLAN --> DR
    CAPMODEL --> DR
    DR -->|Нужно переразбить| E2
    DR -->|Декомпозиция принята| NEW

    NEW --> PROP
    PROP --> SPEC

    SPEC --> CAPGATE
    CAPGATE -->|Неверный path / дублируется capability| UPDATE
    UPDATE --> SPEC
    CAPGATE -->|OK| BEHGATE

    BEHGATE -->|Неоднозначность| UPDATE
    BEHGATE -->|OK| DESIGN

    DESIGN --> TASKS
    TASKS --> APPLY
    APPLY --> CODE
    CODE --> VERIFY

    VERIFY --> READY
    READY -->|Есть behavioral mismatch| UPDATE
    READY -->|Есть implementation mismatch| UPDATE
    READY -->|Да| ARCHIVE

    ARCHIVE --> MASTER
    ARCHIVE --> HISTORY
    MASTER --> STATUS
    HISTORY --> STATUS
    STATUS --> NEXT

    NEXT -->|Да| NEW
    NEXT -->|Нет| DONE
```

---

# 2. Главный lifecycle одного change

```mermaid
flowchart LR
    NEW["new"] --> P["proposal"]
    P --> S["specs"]
    S --> REVIEW["behavioral + capability review"]
    REVIEW --> D["design"]
    D --> T["tasks"]
    T --> A["apply"]
    A --> V["verify"]
    V --> AR["archive"]
    AR --> M["master specs"]
```

В целевом командном процессе рекомендуется **не использовать `/opsx:propose` как основной stage-gated workflow**, потому что `propose` создаёт весь planning-пакет сразу.

Используем:

```text
/opsx:new
/opsx:continue   # proposal
/opsx:continue   # specs
REVIEW
/opsx:continue   # design
/opsx:continue   # tasks
/opsx:apply
/opsx:verify
/opsx:archive
```

---

# 3. Где находится глобальная спецификация

Глобальная функциональная спека модуля не является master spec.

```mermaid
flowchart LR
    TARGET["docs/initiatives/.../target-spec.md<br/>TARGET"] --> DECOMP["decomposition"]
    DECOMP --> C1["Change A"]
    DECOMP --> C2["Change B"]
    DECOMP --> C3["Change C"]

    C1 --> MASTER["openspec/specs/**<br/>CURRENT"]
    C2 --> MASTER
    C3 --> MASTER
```

Пример:

```text
docs/
└── initiatives/
    ├── active/
    │   └── hdfs/
    │       ├── target-spec.md
    │       └── delivery-plan.md
    └── archive/
```

---

# 4. Формирование target-spec аналитиком

```mermaid
sequenceDiagram
    participant A as Аналитик
    participant O as OPSX Explore
    participant R as Reviewer

    A->>O: Исходная бизнес-идея + существующий контекст
    O-->>A: Capabilities, requirements, scenarios, edge cases, open questions
    A->>A: Формирует target-spec.md
    A->>R: Review целевого поведения
    R-->>A: Замечания / approval
    A->>A: Финализирует target-spec.md
```

`target-spec.md` описывает **полный целевой результат**, даже если он будет реализован за несколько итераций.

Рекомендуется использовать стабильные Requirement IDs:

```text
HDFS-NAV-001
HDFS-NAV-002
HDFS-FILE-001
HDFS-FILE-002
HDFS-DIR-001
```

---

# 5. Декомпозиция target-spec

```mermaid
flowchart TD
    TARGET["target-spec.md"]
    CODEBASE["Текущий код"]
    MASTER["openspec/specs/**"]
    EXPLORE["/opsx:explore<br/>delivery decomposition"]

    PLAN["delivery-plan.md"]
    MODEL["Capability Model"]
    CHANGES["Набор небольших OpenSpec changes"]

    TARGET --> EXPLORE
    CODEBASE --> EXPLORE
    MASTER --> EXPLORE

    EXPLORE --> PLAN
    EXPLORE --> MODEL
    EXPLORE --> CHANGES
```

Критерии хорошего change:

- даёт законченное observable behavior;
- может быть реализован и протестирован отдельно;
- может быть отдельно archived;
- не требует ждать завершения всей initiative;
- достаточно мал для одной итерации;
- явно знает, какие Requirement IDs из target-spec он реализует.

---

# 6. Пример HDFS-декомпозиции

Исходный target:

```text
HDFS
├── Navigation
├── Copy file
├── Rename file
├── Copy directory
├── Rename directory
└── Move directory
```

Capability model:

```text
hdfs/
├── navigation
├── file-operations
└── directory-operations
```

Delivery plan:

```mermaid
flowchart TD
    N["add-hdfs-navigation<br/>HDFS-NAV-*"]
    CF["add-hdfs-copy-file<br/>HDFS-FILE-001"]
    RF["add-hdfs-rename-file<br/>HDFS-FILE-002"]
    CD["add-hdfs-copy-directory<br/>HDFS-DIR-001"]
    RD["add-hdfs-rename-directory<br/>HDFS-DIR-002"]
    MD["add-hdfs-move-directory<br/>HDFS-DIR-003"]

    N --> CF
    N --> RF
    CF --> CD
    CD --> RD
    CD --> MD
```

---

# 7. Capability mapping — ключ к master specs

Имя change **не определяет**, какой master spec будет изменён.

```text
change name:
add-hdfs-copy-file
```

не означает автоматически:

```text
openspec/specs/hdfs/copy-file/spec.md
```

Связь задаёт delta path:

```text
openspec/changes/add-hdfs-copy-file/
└── specs/
    └── hdfs/
        └── file-operations/
            └── spec.md
```

Это означает target:

```text
openspec/specs/
└── hdfs/
    └── file-operations/
        └── spec.md
```

---

# 8. Capability Review

Перед design обязательна проверка:

```mermaid
flowchart TD
    DELTA["Delta spec path"]
    Q{"Capability уже существует?"}

    EXIST["Переиспользовать exact existing path"]
    NEW["Создать новый capability-path"]
    DUP["Не создавать near-duplicate capability"]

    DELTA --> Q
    Q -->|Да| EXIST
    Q -->|Нет, действительно новая область| NEW
    Q -->|Есть похожая capability| DUP
    DUP --> EXIST
```

Developer/Analyst должны проверить:

```bash
openspec list --specs
openspec show "<capability>" --type spec
```

---

# 9. Два уровня изменений

Важно различать:

```text
Capability: hdfs/file-operations
Requirement: Copy File
```

Если capability уже есть, а requirement новый:

```text
Modified Capability
+
ADDED Requirement
```

Если requirement уже существует и меняется его контракт:

```text
Modified Capability
+
MODIFIED Requirement
```

Если capability новая:

```text
New Capability
+
ADDED Requirements
```

---

# 10. Как archive обновляет master

```mermaid
flowchart LR
    DELTA["changes/.../specs/hdfs/file-operations/spec.md"]
    PATH["capability-path = hdfs/file-operations"]
    OPS["ADDED / MODIFIED / REMOVED / RENAMED"]
    MASTER["openspec/specs/hdfs/file-operations/spec.md"]

    DELTA --> PATH
    DELTA --> OPS
    PATH --> MASTER
    OPS --> MASTER
```

Archive не должен заново угадывать, куда положить изменение.  
Он применяет delta к path, который уже был зафиксирован во время planning.

---

# 11. Как master нарастает инкрементально

```text
После change 1:
hdfs/navigation
  - Browse directory

После change 2:
hdfs/navigation
  - Browse directory

hdfs/file-operations
  - Copy file

После change 3:
hdfs/file-operations
  - Copy file
  - Rename file
```

Каждая итерация оставляет master specs в корректном as-built состоянии.

---

# 12. Что делать при изменении требования во время реализации

```mermaid
flowchart TD
    ISSUE["Обнаружена проблема во время apply"]
    Q{"Меняется observable behavior?"}
    SPEC["/opsx:update<br/>Обновить specs"]
    TARGET{"Меняется общий target initiative?"}
    TGT["Обновить target-spec.md"]
    DESIGN["Обновить design/tasks"]
    CONT["Продолжить apply"]

    ISSUE --> Q

    Q -->|Да| SPEC
    SPEC --> TARGET
    TARGET -->|Да| TGT
    TARGET -->|Нет| DESIGN
    TGT --> DESIGN

    Q -->|Нет| DESIGN

    DESIGN --> CONT
```

---

# 13. Когда использовать `/opsx:update`

Использовать, если change уже создан и нужно поправить существующие planning artifacts:

```text
proposal
specs
design
tasks
```

Примеры:

- уточнилось бизнес-поведение;
- изменилась граница scope;
- найдено противоречие между spec и design;
- техническое решение поменялось;
- появились дополнительные implementation tasks.

Не использовать update как замену созданию следующего delivery change.

---

# 14. Verify gate

```mermaid
flowchart TD
    V["/opsx:verify"]
    C["Completeness"]
    R["Correctness"]
    H["Coherence"]
    OK{"Готово к archive?"}
    U["/opsx:update или исправление кода"]
    A["/opsx:archive"]

    V --> C
    V --> R
    V --> H
    C --> OK
    R --> OK
    H --> OK
    OK -->|Нет| U
    U --> V
    OK -->|Да| A
```

`verify` не меняет файлы — это report-only gate.

---

# 15. Archive gate

Перед archive:

- tasks завершены;
- code соответствует финальным specs;
- behavioral gaps закрыты;
- design отражает фактическое техническое решение;
- delta capability paths корректны;
- change валиден.

```text
/opsx:archive
   ↓
sync outstanding deltas
   ↓
openspec/specs/** обновлены
   ↓
change перемещён в archive
```

---

# 16. Почему `sync` не является обычным этапом

Нормальный flow:

```text
apply → verify → archive → master update
```

`sync` использовать только сознательно:

```text
long-running change
или
нужно раньше обновить main specs
```

Риск:

```text
master spec = target
production = ещё current
```

Поэтому для коротких delivery changes рекомендуется не использовать `sync` до archive.

---

# 17. Технические changes без behavioral delta

Для чистого:

- refactoring;
- tooling;
- infrastructure;
- docs-only change;

можно использовать:

```yaml
skip_specs: true
```

Тогда change не должен придумывать искусственный behavioral requirement.

---

# 18. Исправление master spec без behavioral change

Если master ошибочно описывает уже существующее поведение:

```text
Code/production = 30 дней
Master spec = 14 дней
```

и подтверждено, что поведение менять не нужно:

```text
direct spec correction PR
```

Рекомендуемый маркер:

```text
Spec correction only.
No behavior change.
```

Если intended behavior неясен — сначала reconciliation, затем либо doc correction, либо новый change.

---

# 19. Как обновляется initiative после archive

```mermaid
flowchart LR
    AR["Archived change"]
    DP["delivery-plan.md"]
    RID["Requirement IDs"]
    NEXT["Следующий planned change"]

    AR --> DP
    DP --> RID
    RID --> NEXT
```

В delivery plan обновляются:

- статус change;
- ссылка на archived change;
- реализованные Requirement IDs;
- зависимости;
- следующий кандидат на реализацию.

---

# 20. Полный flow в sequence diagram

```mermaid
sequenceDiagram
    participant A as Аналитик
    participant E as OPSX Explore
    participant D as Developer
    participant O as OPSX Change Workflow
    participant M as Master Specs

    A->>E: Проработать target-state
    E-->>A: Requirements + scenarios + open questions
    A->>A: target-spec.md
    A->>D: Review target

    D->>E: Декомпозировать target + code + master specs
    E-->>D: Changes + dependencies + capability model
    D->>A: Review decomposition
    A-->>D: Approved
    D->>D: delivery-plan.md

    D->>O: /opsx:new <change>
    A->>O: /opsx:continue
    O-->>A: proposal.md
    A->>O: /opsx:continue
    O-->>A: delta specs

    D->>O: Capability + Behavioral Review
    O-->>D: Исправленный change при необходимости

    D->>O: /opsx:continue
    O-->>D: design.md
    D->>O: /opsx:continue
    O-->>D: tasks.md

    D->>O: /opsx:apply
    O-->>D: Code + completed tasks

    D->>O: /opsx:verify
    O-->>D: Verification report

    alt Есть расхождения
        D->>O: /opsx:update / code fix
        D->>O: /opsx:verify
    end

    D->>O: /opsx:archive
    O->>M: Merge delta specs
    O-->>D: Archived change

    D->>D: Update delivery-plan.md
```

---

# 21. Основные анти-паттерны

## Mega-change на всю initiative

```text
НЕ НАДО:
implement-hdfs
  - navigation
  - copy file
  - rename file
  - copy folder
  - ...
```

Лучше несколько independently archivable changes.

## Target-state в master specs

```text
НЕ НАДО:
openspec/specs = всё, что мы планируем
```

`openspec/specs` должен отражать current/as-built.

## Capability по имени каждой задачи

```text
НЕ НАДО:
hdfs/copy-file
hdfs/rename-file
hdfs/copy-folder
```

если это requirements внутри устойчивых capabilities:

```text
hdfs/file-operations
hdfs/directory-operations
```

## Технические детали в behavioral specs

```text
Redis
Java class
DB table
Spring Retry
```

обычно относятся к design, а не spec.

---

# 22. Рекомендуемый happy path

```text
1. /opsx:explore → target specification
2. сохранить docs/initiatives/active/<module>/target-spec.md
3. review target
4. /opsx:explore → decomposition
5. сохранить delivery-plan.md
6. capability review
7. /opsx:new <small-change>
8. /opsx:continue → proposal
9. /opsx:continue → specs
10. behavioral + capability review
11. /opsx:continue → design
12. /opsx:continue → tasks
13. /opsx:apply
14. /opsx:verify
15. /opsx:update + re-verify при необходимости
16. /opsx:archive
17. обновить delivery-plan
18. повторить для следующего change
19. закрыть initiative
20. перенести initiative docs в docs/initiatives/archive/
```

---

# 23. Источники OpenSpec

Актуальная документация OpenSpec, использованная для этой карты:

- https://openspec.dev/docs/skills
- https://openspec.dev/docs/profiles
- https://openspec.dev/docs/schemas/spec-driven
- https://openspec.dev/docs/customize-schemas
- https://openspec.dev/docs/configuration/config-yaml
- https://openspec.dev/docs/configuration/change-metadata
- https://openspec.dev/docs/cli
- https://openspec.dev/docs/quickstart

> Важно: initiative-level `target-spec.md` и `delivery-plan.md` — рекомендуемый командный слой поверх OpenSpec. OpenSpec пока не имеет first-class lifecycle для такого module target artifact.
