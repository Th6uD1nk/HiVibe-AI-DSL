# HiVibe Specification
**Version:** 0.2.0  
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
| `hvibe_version` | √ | `string` | HiVibe spec version this file targets (e.g. `"0.2.0"`) |
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
  logic: { ... },
  code: { ... },
  tests: { ... }
}

// External reference
catalog: "path/to/shared-catalog.hvibe"
```

| Field | Required | Type | Description |
|-------|:--------:|------|-------------|
| `logic` | | `object` | Named logic entries: algorithms, conditions expressed in natural language. LLM-direct. |
| `code` | | `object` | Named code entries: literal code snippets the LLM must use as-is or as close reference. LLM-direct. |
| `tests` | | `object` | Named tests the LLM must verify before finalizing code. See below. |

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
| `features` | √ | `object` | Named feature definitions. See below. |

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


## Minimal valid `.hvibe` file

```js
{
  hvibe_version: "0.2.0",
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

See [tree.html](https://htmlpreview.github.io/?https://github.com/Th6uD1nk/HiVibe/blob/main/versions/v0.2.0/tree.html) for the interactive version.

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
```


## Design principles

- **Keys are schema.** Unquoted keys belong to the HiVibe schema. They are structural, not instructional.
- **Quoted strings are content.** Double-quoted strings carry developer-facing or tooling-facing information. The LLM reads them but does not treat them as primary instructions.
- **Backtick strings are instructions.** Backtick strings are direct contracts with the LLM. They must be interpreted literally and faithfully, and take priority over all other content.
- **Units are the organizational structure.** Each unit groups related features under a named, optionally described container.
- **Rules are local contracts.** Rules are declared inside the feature they govern. They are permanent behavioral constraints, not suggestions.
- **Tests are pre-finalization checks.** Tests defined in `catalog.tests` must be verified by the LLM before finalizing a feature. If a test fails, the LLM must adjust its output.
- **The catalog is global and composable.** The catalog is shared across all units. It can be defined inline or referenced as an external file, and multiple catalogs across files complement each other.
- **Features are the unit of modularity.** Each feature is a self-contained, describable and extensible unit. Features may declare dependencies on other features, the exact mechanism is reserved for a future version of this spec.
- **The file is the project memory.** A `.hvibe` file should contain enough context to reconstruct or extend the project without additional documentation.


## Changelog

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

Current version: `0.2.0`


*HiVibe is an open specification. Contributions and discussions are welcome.*
