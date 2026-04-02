# Okyline Skill for Claude

A skill that turns Claude into an Okyline expert — able to create, edit, and convert [Okyline](https://github.com/Okyline/okyline-spec) schemas, a declarative, example-driven syntax for describing and validating JSON data.

## Features

Full Okyline core language support:

- **Example-driven**: structure and types inferred from realistic example values
- Inline constraints: ranges `(0..100)`, length `{2,50}`, enums `('A','B','C')`, regex `~pattern~`, required `@`, nullable `?`...
- Arrays and maps: size `[1,10]`, element validation `->`, uniqueness `!`, pattern keys
- Custom formats `$format` and nomenclatures `$nomenclature`
- Built-in formats: `$Email`, `$Date`, `$DateTime`...
- Conditional directives `$appliedIf`, `$requiredIf`, `$forbiddenIf`...
- Expression language for computed validations `$compute`
- Reusable definitions with `$defs` and `$ref` (internal references)

## Installation

### Claude.ai

1. Go to **Settings > Customize > Skills**
2. Click **+** > **Create skill** and upload the ZIP file

### Claude Code

Copy or extract this folder to your skills directory:

```bash
# Global (all projects)
~/.claude/skills/okyline/

# Or project-specific
.claude/skills/okyline/
```

### OpenAI Codex CLI

Copy or extract this folder to:

```bash
~/.codex/skills/okyline/
```

## Usage

Simply ask Claude to create an Okyline schema:

- "Create an Okyline schema for a user profile"
- "Write an okyline for an invoice with line items"
- "Generate a JSON validation schema using Okyline"

You can also provide more context:

- "Create an Okyline schema for a product with name (required, 2-100 chars), price (positive), and optional description"
- "Convert this JSON Schema to Okyline"
- "Transform this Avro schema into Okyline format"

For best results, provide a JSON example along with your existing validation rules or data specifications — Claude will translate them into Okyline constraints automatically.

Or ask questions about the language itself:

- "How do I make a field nullable in Okyline?"
- "How do I define an enum in Okyline?"
- "Explain how references work"

Claude will automatically invoke this skill when relevant.

## Skill Structure

```
okyline-skill/
├── README.md                             # This file
├── LICENSE                               # Apache 2.0
├── SKILL.md                              # Main skill instructions
└── references/
    ├── syntax-reference.md               # Complete constraint syntax
    ├── internal-references.md            # $defs and $ref usage
    ├── conditional-directives.md         # Conditional logic
    ├── expression-language.md            # $compute expressions
    └── virtual-fields.md                 # $field virtual fields
```

## Example

```json
{
  "$oky": {
    "name|@ {2,50}|User name": "Alice",
    "email|@ ~$Email~": "alice@example.com",
    "age|(18..120)": 30,
    "status|@ ('ACTIVE','INACTIVE')": "ACTIVE"
  }
}
```

## Links

- [Okyline Free Resources](https://community.okyline.design-hub.okyline.io/)
- [Okyline Free Studio](https://community.studio.okyline.io/)
- [Okyline Java library on Maven Central](https://central.sonatype.com/artifact/io.akwatype/okyline)
- [Okyline Specification](https://github.com/Okyline/okyline-spec)
- [Agent Skills Standard](https://agentskills.io)

## License

This skill is licensed under the [Apache License 2.0](LICENSE).
