## .hvibe Generation

You can either generate a `.hvibe` file by hand or use an LLM after it has consumed the version of the spec you want to target. You can let the LLM fully generate the file by giving your requirements in natural language, or you can use it to assist you in improving your own `.hvibe` file.  

It is better to understand the specification to have more control over the generated file.

## Simple Prompt to Generate .hvibe

You must generate `.hvibe` files according to the following specification: [SPEC FILE]

* Always use only characters that are universally supported by simple ASCII-based LLMs
* Always prioritize natural language over code
* Use code only to encode logic (in `catalog.logics`) when it cannot be clearly expressed in natural language
* Never include unnecessary characters that do not convey meaning to the LLM or that are not part of the specification

You are now a `.hvibe` agent and must generate an `.hvibe` file for this project:

* Implement this feature: ...
* Ensure that: ...

## Build Pipelines

There are three possible pipelines to build a project via an LLM using HiVibe:

1. Prompt to LLM → LLM answer → `.hive` to LLM → LLM build.
2. Prompt to LLM → LLM answer → `.hvibe.plus` to LLM → LLM build.
3. Prompt to LLM → LLM answer → `.hive` to LLM → LLM `.hvibe.plus` IR generation  
   → Prompt to LLM → LLM answer → `.hvibe.plus` to LLM → LLM build.

## Files

- `prompt-generate-project.hvibe.txt`: prompt for the first pipeline.  
- `prompt-generate-project.hvibe.plus.txt`: prompt for the second pipeline and the second prompt of the third pipeline.  
- `prompt-generate.hvibe.plus.txt`: first prompt of the third pipeline.
