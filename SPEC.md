# HighVibe+ Specification
**Version:** 0.1.0  
**File extension:** `.hvibe`  
**Status:** Draft


## Overview

HighVibe+ is a high-level, structured and maintainable Domain Specific Language for vibe coding, based on JSON.

A `.hvibe` file aims to be a single source of truth for a vibe-coded module or project.  

Its purpose is to:
- **Organize** a vibe coding project in a human-readable and LLM-readable format
- **Maintain** the project over time, regardless of who (or what) reads it next
- **Extend** the project with new features without losing context
- **Modularize** the project by splitting concerns into discrete, named units


## Format

A `.hvibe` file is designed to be a **JavaScript object notation** (not strict JSON). This distinction is intentional and meaningful.

### String conventions

HighVibe+ defines two classes of strings with distinct delimiters:

| Delimiter | Type | Priority | Description |
|-----------|------|----------|-------------|
| `` ` ` `` | **LLM-direct** | High | Content the LLM must treat as a primary instruction. These strings are authoritative and should be interpreted literally and faithfully. |
| `" "` | **LLM-indirect** | Normal | Content intended primarily for developers or parsers/validators. The LLM should not ignore these strings, but they are not primary instructions. |

**Object keys are never quoted.** Keys belong to the HighVibe+ schema and carry structural meaning defined by this specification. The LLM and/or HighVibe+ interpreters are expected to be aware of it.

```js
// Example of the convention in practice
app: "firework",                                            // indirect: identifier for tooling
description: `a simple firework simulation in the browser`, // direct: LLM instruction
```


## Schema

### Top-level fields

| Field | Required | Type | Description |
|-------|:--------:|------|-------------|
| `app` | √ | `string` | The name of the application |
| `description` | √ | `` `string` `` | A natural language description of the app. LLM-direct. |
| `stack` | √ | `array` or `object` | Technologies used. See below. |
| `features` | √ | `object` | Named feature definitions. See below. |
| `hvibe_version` | √ | `string` | HighVibe+ spec version this file targets (e.g. `"0.1.0"`) |


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


### `features`

An object where each key is the **name of a feature**. Feature names are unquoted (schema keys).

Each feature has the following fields:

| Field | Required | Type | Description |
|-------|:--------:|------|-------------|
| `description` | √ | `` `string` `` | What the feature does. LLM-direct. |
| `inputs` | √ | `array` | What triggers or feeds the feature. LLM-direct strings. |

```js
features: {
  generation: {
    description: `generate a firework at the bottom center of the screen`,
    inputs: [`on click`, `on touch`]
  },
  motion: {
    description: `rocket rises from bottom to a random height then explodes into particles`,
    inputs: [`triggered by generation`]
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
└── features
    ├── feature_name
    │   ├── description
    │   └── inputs: [`trigger`, `trigger`]
    └── feature_name
        ├── description
        └── inputs: [`trigger`]
```


## Design principles

- **Keys are schema.** Unquoted keys belong to the HighVibe+ schema. They are structural, not instructional.
- **Quoted strings are content.** Double-quoted strings carry developer-facing or tooling-facing information. The LLM reads them but does not treat them as primary instructions.
- **Backtick strings are instructions.** Backtick strings are direct contracts with the LLM. They must be interpreted literally and faithfully, and take priority over all other content.
- **Features are the unit of modularity.** Each feature is a self-contained, describable and extensible unit. Features may declare dependencies on other features, the exact mechanism is reserved for a future version of this spec.
- **The file is the project memory.** A `.hvibe` file should contain enough context to reconstruct or extend the project without additional documentation.


## Versioning

The `hvibe_version` field must be present in every file. It refers to the version of the HighVibe+ specification the file is written against, not the version of the app itself.

Current version: `0.1.0`


*HighVibe+ is an open specification. Contributions and discussions are welcome.*
