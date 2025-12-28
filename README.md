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

> **Note:** External references (`$xRef`) are not supported.

## Installation

### Claude Code

```bash
# Clone to your global skills folder
git clone https://github.com/Okyline/okyline-skill ~/.claude/skills/okyline

# Or for project-specific use
git clone https://github.com/Okyline/okyline-skill .claude/skills/okyline
```

### OpenAI Codex CLI

```bash
git clone https://github.com/Okyline/okyline-skill ~/.codex/skills/okyline
```

### Claude.ai

1. Download this repository as a ZIP file
2. Go to **Settings > Capabilities > Skills**
3. Click **Upload skill** and select the ZIP file

## Usage

Simply ask Claude to create an Okyline schema:

- "Create an Okyline schema for a user profile"
- "Write an okyline for an invoice with line items"
- "Generate a JSON validation schema using Okyline"

You can also provide more context:

- "Create an Okyline schema for a product with name (required, 2-100 chars), price (positive), and optional description"
- "Convert this JSON Schema to Okyline"
- "Transform this Avro schema into Okyline format"

Or ask questions about the language itself:

- "How do I make a field nullable in Okyline?"
- "How do I define an enum in Okyline?"
- "Explain how references work"

Claude will automatically invoke this skill when relevant.

## Skill Structure

```
okyline-skill/
├── SKILL.md                              # Main skill instructions
├── marketplace.json                      # Metadata for skill marketplaces
├── LICENSE                               # Apache 2.0
└── references/
    ├── syntax-reference.md               # Complete constraint syntax
    ├── internal-references.md            # $defs and $ref usage
    ├── conditional-directives.md         # Conditional logic
    └── expression-language.md            # $compute expressions
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

- [Okyline Free Studio](https://community.studio.okyline.io/)
- [Okyline Getting Started](https://public-docs.okyline.io/Okyline-getting-started.pdf)
- [Okyline Java library on Maven Central](https://central.sonatype.com/artifact/io.akwatype/okyline)
- [Okyline Specification](https://github.com/Okyline/okyline-spec)
- [Agent Skills Standard](https://agentskills.io)

## License

This skill is licensed under the [Apache License 2.0](LICENSE).
