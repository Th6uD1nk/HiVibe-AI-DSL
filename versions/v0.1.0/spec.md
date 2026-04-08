# HiVibe Specification
**Version:** 0.1.0  
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
| `hvibe_version` | √ | `string` | HiVibe spec version this file targets (e.g. `"0.1.0"`) |
| `app` | √ | `string` | The name of the application |
| `description` | √ | `` `string` `` | A natural language description of the app. LLM-direct. |
| `stack` | √ | `array` or `object` | Technologies used. See below. |
| `features` | √ | `object` | Named feature definitions. See below. |
| `catalog` | | `object` | Reusable logic, code, rules and tests. Optional. See below. |


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

An optional top-level object that stores reusable references. Catalog entries are referenced anywhere using their dot-path (e.g. `"catalog.logic.rocket_arc"`). Catalog keys are schema keys (unquoted). All values are LLM-direct.

| Field | Required | Type | Description |
|-------|:--------:|------|-------------|
| `logic` | | `object` | Named logic entries: algorithms, conditions expressed in natural language. LLM-direct. |
| `code` | | `object` | Named code entries: literal code snippets the LLM must use as-is or as close reference. LLM-direct. |
| `rules` | | `object` | Named rules the LLM must implement and enforce. See below. |
| `tests` | | `object` | Named tests the LLM must verify before finalizing code. See below. |

#### `catalog.rules`

Rules are constraints the LLM must implement in the generated code. They are not hints — they are permanent behavioral contracts. Rules use uppercase keywords to signal their imperative nature.

| Keyword | Meaning |
|---------|---------|
| `MUST BE` | the subject must have this property or state |
| `MUST ALWAYS` | the subject must unconditionally respect this constraint |
| `MUST NEVER` | the subject must unconditionally avoid this behavior |
| `OTHERWISE` | consequence to apply if the rule is violated |

Rules are organized under two optional sub-objects:

| Field | Description |
|-------|-------------|
| `obligations` | Behavioral properties the LLM must implement |
| `constraints` | Hard limits the LLM must enforce |

```js
rules: {
  obligations: {
    explosion: `MUST BE disk-shaped, growing and then explode to particles`,
    limit:     `MUST ALWAYS stay inside the visible screen area, OTHERWISE explode before reaching screen boundary`
  },
  constraints: {
    step:       `MUST ALWAYS apply a step of no more than 10 per increment`,
    concurrent: `MUST NEVER exceed 10 simultaneous rockets, OTHERWISE ignore the input`
  }
}
```

#### `catalog.tests`

Tests are behavioral assertions the LLM must verify before finalizing the code of a feature. If a test does not pass, the LLM must adjust its generated code until it does. Tests use the same uppercase keywords as rules, combined with a `given` prefix to describe the condition.

```js
tests: {
  generatePosition: `given a click at any point on the screen, the rocket MUST ALWAYS be generated at the bottom center regardless of click position`,
  generateLimit:    `given 10 rockets already active, a new click MUST NEVER generate an 11th rocket`,
  motionUpward:     `verify the motion is always upward, the rocket MUST NEVER move sideways or downward`,
  motionDirection:  `given a rocket in motion, its y position MUST ALWAYS strictly decrease at every step`
}
```


### `features`

An object where each key is the **name of a feature**. Feature names are unquoted (schema keys).

Each feature has the following fields:

| Field | Required | Type | Description |
|-------|:--------:|------|-------------|
| `description` | √ | `` `string` `` | What the feature does. LLM-direct. |
| `inputs` | √ | `array` | What triggers or feeds the feature. LLM-direct strings. |
| `directives` | | `array` | Rules to apply for this feature. Catalog dot-path references or direct LLM instructions in backticks. |
| `tests` | | `array` | Tests the LLM must pass before finalizing this feature. Catalog dot-path references. |
| `blueprints` | | `array` | Implementation hints. Catalog dot-path references or direct LLM instructions in backticks. |
| `depends_on` | | `array` | List of dependency objects. See below. |

#### `depends_on`

Each entry in `depends_on` is an object with the following fields:

| Field | Required | Type | Description |
|-------|:--------:|------|-------------|
| `feature` | √ | `` `string` `` | The name of the feature this one depends on. LLM-direct. |
| `when` | | `array` | Conditions under which the dependency applies. LLM-direct strings. |

```js
features: {
  generation: {
    description: `generate a rocket at the bottom center of the screen`,
    inputs: [`on click`, `on touch`],
    directives: [
      `apply catalog.rules.constraints.concurrent for generation`,
      `apply catalog.rules.constraints.step for generation`
    ],
    tests: [
      "catalog.tests.generatePosition",
      "catalog.tests.generateLimit"
    ],
    blueprints: [
      "catalog.logic.rocket_arc"
    ]
  },
  motion: {
    description: `rocket rises from bottom to a random height`,
    inputs: [`triggered by generation`],
    directives: [
      `apply catalog.rules.constraints.step for motion`,
      `apply catalog.rules.obligations.limit for motion`
    ],
    tests: [
      "catalog.tests.motionUpward",
      "catalog.tests.motionDirection"
    ],
    blueprints: [
      "catalog.code.animation_loop"
    ]
  },
  explosion: {
    description: `explode the rocket into particles when it reaches its target height`,
    inputs: [`triggered by motion`],
    directives: [
      `apply catalog.rules.obligations.explosion for explosion`,
      `apply catalog.rules.obligations.limit for explosion`
    ],
    depends_on: [
      {
        feature: `motion`,
        when: [`rocket has reached target height`]
      }
    ],
    blueprints: [
      "catalog.logic.particle_easing",
      "catalog.logic.explosion_trigger",
      `80 particles per explosion, spread 360 degrees`
    ]
  }
}
```


## Minimal valid `.hvibe` file

```js
{
  hvibe_version: "0.1.0",
  app: "my-app",
  description: `what this app does`,
  stack: [`javascript`],
  features: {
    core: {
      description: `the main feature of the app`,
      inputs: [`user interaction`]
    }
  }
}
```


## Structure tree

See [tree.html](https://htmlpreview.github.io/?https://github.com/Th6uD1nk/HiVibe/blob/main/versions/v0.1.0/tree.html) for the interactive version.

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
├── catalog (optional)
│   ├── logic:
│   │   └── logic_name: `description or condition`
│   ├── code:
│   │   └── code_name: `literal code`
│   ├── rules:
│   │   ├── obligations:
│   │   │   └── rule_name: `MUST BE / MUST ALWAYS ... OTHERWISE ...`
│   │   └── constraints:
│   │       └── rule_name: `MUST ALWAYS / MUST NEVER ... OTHERWISE ...`
│   └── tests:
│       └── test_name: `given ... MUST BE / MUST ALWAYS / MUST NEVER ...`
└── features
    ├── feature_name
    │   ├── description
    │   ├── inputs: [`trigger`, `trigger`]
    │   ├── directives: [`apply catalog.rules.x for feature`]
    │   ├── tests: ["catalog.tests.name"]
    │   ├── blueprints: ["catalog.logic.name", `llm instruction`]
    │   └── depends_on:
    │       ├── feature: `feature_name`
    │       └── when: [`condition`, `condition`]
    └── feature_name
        ├── description
        └── inputs: [`trigger`]
```


## Design principles

- **Keys are schema.** Unquoted keys belong to the HiVibe schema. They are structural, not instructional.
- **Quoted strings are content.** Double-quoted strings carry developer-facing or tooling-facing information. The LLM reads them but does not treat them as primary instructions.
- **Backtick strings are instructions.** Backtick strings are direct contracts with the LLM. They must be interpreted literally and faithfully, and take priority over all other content.
- **Rules are implemented contracts.** Rules defined in `catalog.rules` must be implemented in the generated code. They are permanent behavioral constraints, not suggestions.
- **Tests are pre-finalization checks.** Tests defined in `catalog.tests` must be verified by the LLM before finalizing a feature. If a test fails, the LLM must adjust its output.
- **Features are the unit of modularity.** Each feature is a self-contained, describable and extensible unit. Features may declare dependencies on other features, the exact mechanism is reserved for a future version of this spec.
- **The file is the project memory.** A `.hvibe` file should contain enough context to reconstruct or extend the project without additional documentation.


## Versioning

The `hvibe_version` field must be present in every file. It refers to the version of the HiVibe specification the file is written against, not the version of the app itself.

Current version: `0.1.0`


*HiVibe is an open specification. Contributions and discussions are welcome.*
