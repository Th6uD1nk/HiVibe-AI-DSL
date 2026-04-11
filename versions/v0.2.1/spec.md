# HiVibe Specification
**Version:** 0.2.1  
**File extension:** `.hvibe`  
**Status:** Draft


## Overview

HiVibe is a high-level, structured and maintainable Domain Specific Language for vibe coding, based on JSON.

Its purpose is to:
- **Organize** a vibe coding project in a human-readable and LLM-readable format
- **Maintain** the project over time, regardless of who (or what) reads it next
- **Extend** the project with new features without losing context
- **Modularize** the project by splitting concerns into discrete, named units


## Format

A `.hvibe` file is based on JavaScript object notation, without being strictly JSON nor JavaScript. This distinction is intentional and meaningful.

### String conventions

HiVibe defines two classes of strings with distinct delimiters:

| Delimiter | Type | Priority | Description |
|-----------|------|----------|-------------|
| `` ` ` `` | **LLM-direct** | High | Content the LLM must treat as a primary instruction. These strings are authoritative and should be interpreted literally and faithfully. |
| `" "` | **LLM-indirect** | Normal | Content intended primarily for developers or parsers/validators. The LLM should not ignore these strings, but they are not primary instructions. |

**Object keys are never quoted.** Keys belong to the HiVibe schema and carry structural meaning defined by this specification. The LLM and/or HiVibe interpreters are expected to be aware of it.

```js
// Example of the convention in practice
app: "firework",                                            // indirect: identifier for tooling
description: `a simple firework simulation in the browser`, // direct: LLM instruction
```


## Schema

### Top-level fields

| Field | Required | Type | Description |
|-------|:--------:|------|-------------|
| `hvibe_version` | √ | `string` | HiVibe spec version this file targets (e.g. `"0.2.1"`) |
| `app` | √ | `string` | The name of the application |
| `description` | √ | `` `string` `` | A natural language description of the app. LLM-direct. |
| `stack` | √ | `array` or `object` | Technologies used. See below. |
| `units` | √ | `object` | Named unit definitions. See below. |
| `catalog` | | `string` or `object` | Reusable logic, code and tests. Optional. Can be an inline object or a path to an external `.hvibe` catalog file. See below. |


### `stack`

Can be a flat array when the stack is simple, or a keyed object when the project has distinct layers.

```js
// Flat
stack: [`javascript`]

// Layered
stack: {
  front: [`react`, `tailwind`],
  back:  [`node`, `express`],
  db:    [`sqlite`]
}
```

All values inside `stack` are LLM-direct (backtick strings). They are primary constraints the LLM must respect when generating code.


### `catalog`

An optional top-level object that stores reusable references shared across all units. Catalog entries are referenced anywhere using their dot-path (e.g. `"catalog.logic.rocket_arc"`). Catalog keys are schema keys (unquoted). All values are LLM-direct.

The catalog can be defined inline or referenced as an external file. When defined across multiple files, catalogs are merged and complement each other.

```js
// Inline
catalog: {
  globals: `...`,
  logic: { ... },
  code: { ... },
  tests: { ... }
}

// External reference
catalog: "path/to/shared-catalog.hvibe"
```

| Field | Required | Type | Description |
|-------|:--------:|------|-------------|
| `globals` | | `` `string` `` | Shared constants and variables used across all features. LLM-direct. Declared first inside the catalog. See below. |
| `logic` | | `object` | Named logic entries: algorithms, conditions expressed in natural language. LLM-direct. |
| `code` | | `object` | Named code entries: literal code snippets the LLM must use as close reference. LLM-direct. |
| `tests` | | `object` | Named tests the LLM must verify before finalizing code. See below. |

#### STRICT MODE
If a catalog entry starts with:

```txt
STRICT
```

then it becomes a **deterministic execution contract**.

When STRICT is present:
* The instruction MUST be implemented exactly as written
* No reinterpretation is allowed
* No abstraction or simplification is allowed
* No architectural refactor is allowed
* It must be treated as pseudo-executable specification
* Any deviation = implementation error

Example:

```js
logic: {
  physics_step: `
    STRICT ALGORITHM:

    1. Apply gravity to vertical acceleration (ay += GRAVITY)
    2. Update velocity using acceleration (vx += ax, vy += ay)
    3. Update position using velocity (x += vx, y += vy)

    RULE:
    - Gravity is applied before velocity update
    - Execution order must never be changed
  `
}
```

#### `catalog.globals`

An optional backtick field that declares constants and variables shared across all features. The LLM must read this field first and use the declared values as-is throughout the generated code. Values must not be redeclared.

```js
catalog: {
  globals: `
    const GRAVITY = 0.5
    const JUMP_IMPULSE = -12
    const DEATH_BASE_SPEED = 60
  `,
  logic: { ... },
  tests: { ... }
}
```

#### `catalog.tests`

Tests are behavioral assertions the LLM must verify before finalizing the code of a feature. If a test does not pass, the LLM must adjust its generated code until it does. Tests use uppercase keywords combined with a `given` prefix to describe the condition.

| Keyword | Meaning |
|---------|---------|
| `MUST BE` | the subject must have this property or state |
| `MUST ALWAYS` | the subject must unconditionally respect this constraint |
| `MUST NEVER` | the subject must unconditionally avoid this behavior |
| `OTHERWISE` | consequence to apply if the rule is violated |

```js
tests: {
  generatePosition: `given a click at any point on the screen, the rocket MUST ALWAYS be generated at the bottom center regardless of click position`,
  generateLimit:    `given 10 rockets already active, a new click MUST NEVER generate an 11th rocket`
}
```


### `units`

An object where each key is the name of a unit. Units are the primary organizational structure of a `.hvibe` file. Each unit groups a set of related features.

| Field | Required | Type | Description |
|-------|:--------:|------|-------------|
| `name` | | `string` | The display name of the unit |
| `description` | | `` `string` `` | A natural language description of the unit's purpose. LLM-direct. Optional. |
| `references` | | `array` | A list of external URLs or file paths the LLM must fetch and read before generating any code for this unit. See below. |
| `features` | √ | `object` | Named feature definitions. See below. |

#### `unit.references`

An optional array of URLs or local paths. When present, the LLM must fetch and read every reference listed before generating any code for the unit. References are treated as authoritative external context — the LLM must not simulate or assume their content.

If a reference cannot be fetched, the LLM must explicitly state it before proceeding.

```js
simulation: {
  references: [
    "https://github.com/Th6uD1nk/project/blob/main/src/file.c",
    "https://github.com/Th6uD1nk/project/blob/main/src/file.h"
  ],
  features: { ... }
}
```

```js
units: {
  gameplay: {
    name: "Gameplay",
    description: `core gameplay mechanics`,
    features: {
      character: { ... },
      collision: { ... }
    }
  },
  ui: {
    name: "UI",
    references: ["https://example.com/ui-guidelines.md"],
    features: {
      game_over: { ... },
      restart: { ... }
    }
  }
}
```


### `features`

An object where each key is the name of a feature. Feature names are unquoted (schema keys). Features live inside a unit.

| Field | Required | Type | Description |
|-------|:--------:|------|-------------|
| `description` | √ | `` `string` `` | What the feature does. LLM-direct. |
| `inputs` | √ | `array` | What triggers or feeds the feature. LLM-direct strings. |
| `rules` | | `object` | Local behavioral contracts for this feature. See below. Optional. |
| `tests` | | `array` | Tests the LLM must pass before finalizing this feature. Catalog dot-path references. |
| `blueprints` | | `array` | Implementation hints. Catalog dot-path references or direct LLM instructions in backticks. |
| `depends_on` | | `array` | List of dependency objects. See below. |

#### `feature.rules`

Rules are local to the feature. They are permanent behavioral contracts the LLM must implement in the generated code for that feature. They use the same uppercase keywords as catalog tests.

Rules are organized under two optional sub-objects:

| Field | Description |
|-------|-------------|
| `obligations` | Behavioral properties the LLM must implement |
| `constraints` | Hard limits the LLM must enforce |

```js
character: {
  description: `a square-shaped character controlled by the player`,
  inputs: [`on click`, `on touch`],
  rules: {
    obligations: {
      shape: `MUST BE a square shape at all times`
    },
    constraints: {
      gravity:   `MUST ALWAYS apply a downward acceleration of 0.5px per frame when not on a platform`,
      jump:      `MUST NEVER allow more than one jump while airborne, jump impulse MUST BE exactly -12px`
    }
  },
  tests: ["catalog.tests.jumpInput", "catalog.tests.doubleJump"],
  blueprints: ["catalog.logic.jump_arc"]
}
```

#### `depends_on`

Each entry in `depends_on` is an object with the following fields:

| Field | Required | Type | Description |
|-------|:--------:|------|-------------|
| `feature` | √ | `` `string` `` | The name of the feature this one depends on. LLM-direct. |
| `when` | | `array` | Conditions under which the dependency applies. LLM-direct strings. |


## `.hvibe.plus`

A `.hvibe.plus` file is an enriched version of a `.hvibe` file. It is structurally identical to the source file but replaces backtick string content in specific paths with JavaScript arrow functions. The original natural language content is preserved as a JSDoc comment inside the function body.

### Purpose

- Provide the code generation agent with concrete, syntactically valid implementation sketches rather than natural language descriptions alone.
- Increase the precision and consistency of code generation, especially for smaller or less capable models.
- Serve as an intermediate step between the source `.hvibe` and the final generated code.

### Enriched paths

Enrichment applies exclusively to the following paths:

| Path | Description |
|------|-------------|
| `catalog.globals` | Shared constants declared as a backtick field |
| `catalog.logic.$name` | Logic entries replaced with arrow functions |
| `catalog.tests.$name` | Test entries replaced with arrow functions asserting passing conditions |
| `feature.$name.description` | Feature descriptions replaced with arrow functions |
| `feature.$name.rules.constraints.$name` | Constraint rules replaced with arrow functions |
| `feature.$name.rules.obligations.$name` | Obligation rules replaced with arrow functions |

All other paths are left exactly as-is. No fields are added, removed, or renamed outside the enriched paths.

### Format

The enrichment language is always JavaScript, regardless of the stack. Each enriched backtick string becomes an arrow function. The function name matches the field key. The original natural language content becomes a JSDoc comment inside the function body.

```js
fieldKey: `
  // original natural language content preserved here as comment
  const fieldKey = () => {
    // implementation sketch in JS
  }
`
```

Global variables are declared in `catalog.globals` as a backtick field, before `logic` and `tests`:

```js
catalog: {
  globals: `
    const GRAVITY = 0.5
    const JUMP_IMPULSE = -12
  `,
  logic: { ... },
  tests: { ... }
}
```

### Formatting rules

- The overall object structure uses 2 spaces indentation at every nesting level.
- Every backtick string content is indented with 2 spaces relative to its parent key.
- Every JS arrow function body is indented with 2 spaces relative to the function opening brace.
- Multiple lines are never collapsed into one. Line breaks between fields are never omitted.
- No markdown fences. No text outside the hvibe structure.

### Reading rules for the generation agent

When receiving a `.hvibe.plus` file, the generation agent must:

1. Read `catalog.globals` first and treat all declared variables as global constants. Do not redeclare them.
2. Treat every arrow function body as the authoritative implementation reference. Code takes priority over its JSDoc comment.
3. Adapt the code only when strictly required by global implementation constraints such as naming conflicts, integration with `depends_on`, or stack coherence. Never reinterpret, simplify, or ignore the intent. Any adaptation must preserve the original logic exactly.


## `.hvibe.lock`

A `.hvibe.lock` file is an automatically generated companion file that records which features have been implemented and verified in a previous session. It acts as a protection layer for the next agent: locked features must not be reimplemented or reinterpreted.

### Purpose

- Prevent inter-session drift: a new agent must not reinterpret a rule that a previous agent has already resolved and validated.
- Reduce token consumption: the agent reads the lock first and skips locked features, focusing only on what remains to implement.
- Preserve user-validated decisions without encoding them in the `.hvibe` file itself.

### Format

The lock file uses the same JavaScript object notation as `.hvibe`. Keys are unquoted. Feature paths follow the `unit.feature` dot-path convention of the parent `.hvibe` file.

```js
{
  hvibe_lock_version: "0.1.0",
  locked_at: "2026-04-07",
  features: {
    "simulation.generation": { locked: true },
    "simulation.motion":     { locked: true },
    "simulation.explosion":  { locked: false },
    "output.output":         { locked: true }
  }
}
```

### Lock rules for the LLM

- **If a feature is `locked: true`**, the LLM must treat its existing implementation as final. It must not regenerate, refactor, or reinterpret it. This applies to both generation and enrichment agents.
- **If a feature needs to change despite being locked**, the LLM must flag it explicitly and request user validation before modifying anything. This must be indicated in the first prompt of the session.
- **If no lock file is present**, the LLM proceeds normally with full generation or enrichment.

### Fields

| Field | Required | Type | Description |
|-------|:--------:|------|-------------|
| `hvibe_lock_version` | √ | `string` | Version of the lock format |
| `locked_at` | √ | `string` | ISO date of the last lock update |
| `features` | √ | `object` | Map of `unit.feature` dot-paths to lock entries |

Each lock entry:

| Field | Required | Type | Description |
|-------|:--------:|------|-------------|
| `locked` | √ | `boolean` | Whether this feature is locked |


## Minimal valid `.hvibe` file

```js
{
  hvibe_version: "0.2.1",
  app: "my-app",
  description: `what this app does`,
  stack: [`javascript`],
  units: {
    core: {
      name: "Core",
      features: {
        main: {
          description: `the main feature of the app`,
          inputs: [`user interaction`]
        }
      }
    }
  }
}
```


## Structure tree

```
.hvibe
├── hvibe_version
├── app
├── description
├── stack
│   ├── flat:    [`tech`, `tech`]
│   └── layered:
│       ├── front: [`tech`, `tech`]
│       ├── back:  [`tech`, `tech`]
│       └── db:    [`tech`]
├── catalog (optional, inline or external path)
│   ├── globals: `const VAR = value`
│   ├── logic:
│   │   └── logic_name: `description or condition`
│   ├── code:
│   │   └── code_name: `literal code`
│   └── tests:
│       └── test_name: `given ... MUST BE / MUST ALWAYS / MUST NEVER ...`
└── units
    └── unit_name
        ├── name (optional)
        ├── description (optional)
        ├── references (optional): ["url_or_path", ...]
        └── features
            └── feature_name
                ├── description
                ├── inputs: [`trigger`, `trigger`]
                ├── rules (optional)
                │   ├── obligations:
                │   │   └── rule_name: `MUST BE / MUST ALWAYS ... OTHERWISE ...`
                │   └── constraints:
                │       └── rule_name: `MUST ALWAYS / MUST NEVER ... OTHERWISE ...`
                ├── tests: ["catalog.tests.name"]
                ├── blueprints: ["catalog.logic.name", `llm instruction`]
                └── depends_on:
                    ├── feature: `feature_name`
                    └── when: [`condition`, `condition`]

.hvibe.plus (enriched variant)
├── [same structure as .hvibe]
└── catalog
    ├── globals: `const VAR = value  ...`
    ├── logic:
    │   └── logic_name: `const logic_name = () => { ... }`
    └── tests:
        └── test_name: `const test_name = () => { console.assert(...) }`

.hvibe.lock
├── hvibe_lock_version
├── locked_at
└── features
    └── unit.feature: { locked: true | false }
```


## Design principles

- **Keys are schema.** Unquoted keys belong to the HiVibe schema. They are structural, not instructional.
- **Quoted strings are content.** Double-quoted strings carry developer-facing or tooling-facing information. The LLM reads them but does not treat them as primary instructions.
- **Backtick strings are instructions.** Backtick strings are direct contracts with the LLM. They must be interpreted literally and faithfully, and take priority over all other content.
- **Units are the organizational structure.** Each unit groups related features under a named, optionally described container.
- **References are mandatory external context.** When a unit declares references, the LLM must fetch and read them before generating any code for that unit. References are authoritative.
- **Rules are local contracts.** Rules are declared inside the feature they govern. They are permanent behavioral constraints, not suggestions.
- **Tests are pre-finalization checks.** Tests defined in `catalog.tests` must be verified by the LLM before finalizing a feature. If a test fails, the LLM must adjust its output.
- **The catalog is global and composable.** The catalog is shared across all units. It can be defined inline or referenced as an external file, and multiple catalogs across files complement each other.
- **`catalog.globals` is read first.** Global constants declared in `catalog.globals` are shared across all features and must not be redeclared.
- **`.hvibe.plus` is the enriched intermediate.** A `.hvibe.plus` file replaces natural language backtick content with JavaScript arrow functions, providing concrete implementation sketches for the generation agent.
- **Features are the unit of modularity.** Each feature is a self-contained, describable and extensible unit. Features may declare dependencies on other features, the exact mechanism is reserved for a future version of this spec.
- **The file is the project memory.** A `.hvibe` file should contain enough context to reconstruct or extend the project without additional documentation.
- **The lock file is the session memory.** A `.hvibe.lock` file records validated implementation decisions across sessions. Locked features are immutable unless the user explicitly approves a change. The lock applies to both generation and enrichment agents.


## Changelog

### 0.2.1
- Added `references` field to units. When present, the LLM must fetch and read all listed URLs or paths before generating code for the unit.
- Introduced the `.hvibe.lock` companion file format. The lock records which features have been implemented and verified, preventing inter-session drift and reducing token consumption.
- Added `catalog.globals`: an optional backtick field declaring shared constants and variables used across all features. Must be declared first inside the catalog. The generation agent reads it first and must not redeclare its values.
- Introduced the `.hvibe.plus` file format: a structurally identical enriched variant of `.hvibe` where backtick content in specific paths is replaced with JavaScript arrow functions. Serves as an intermediate step between the source `.hvibe` and final generated code.

### 0.2.0
- Introduced `units` as the primary organizational structure. Features now live inside units.
- Each unit has a required `name` and an optional `description`.
- Rules are now local to each feature via the `rules` field. The `directives` field is removed.
- `catalog.rules` is removed. The catalog now contains only `logic`, `code`, and `tests`.
- The `catalog` field now accepts either an inline object or a path string to an external `.hvibe` catalog file. Multiple catalogs across files are merged and complement each other.

### 0.1.0
- Initial specification.


## Versioning

The `hvibe_version` field must be present in every file. It refers to the version of the HiVibe specification the file is written against, not the version of the app itself.

Current version: `0.2.1`


*HiVibe is an open specification. Contributions and discussions are welcome.*
