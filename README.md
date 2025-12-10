# AI Django Rules

A collection of coding style guidelines and best practices tailored specifically for Cursor to follow when generating and reviewing Django code. This repository contains markdown documentation files (`.md` and `.mdc`) that provide context and rules for maintaining consistent code style across Django development projects.

## Overview

This project serves as a centralized style guide that Cursor can reference to:

- ✨ **Generate code** that adheres to your preferred coding standards and patterns
- 🔍 **Review code** against your established style guidelines
- 🔄 **Maintain consistency** across different projects and codebases
- 📋 **Enforce best practices** specific to your Django development workflow

## Structure

The repository contains markdown files (`.md` and `.mdc`) organized by Django topics:

### Commands (`commands/`)

- **`commands-base.mdc`** - Overview of Django management commands, command types (project-scoped vs app-scoped), base classes, and best practices
- **`commands-structure.mdc`** - Command structure standards, output phases (starting, success, skip, error), required try/except blocks, and comment guidelines
- **`commands-example.mdc`** - Minimal command template with examples

### Serializers (`serializers/`)

- **`serializers-base.mdc`** - Serializer field requirements, `help_text` guidelines, and best practices

## File Format

Files use the `.mdc` extension (Markdown with Cursor context) and include frontmatter:

```yaml
---
alwaysApply: true
---
```

- **`alwaysApply: true`** - Ensures Cursor always references these rules when generating code
- Files are organized by topic in directories
- Each file focuses on a specific aspect of Django development

## Usage

Configure Cursor to reference these files so it understands your coding preferences and generates code that matches your style. Simply point Cursor to this repository or include it in your project context.

### Setting Up in Cursor

1. Add this repository to your Cursor workspace or reference it in your project settings
2. Cursor will automatically read `.mdc` files with `alwaysApply: true`
3. The AI will follow these guidelines when generating Django code

## Contributing

We welcome contributions to improve and expand these coding standards. Here's how to contribute:

### Adding New Rules

1. **Choose the right directory** - Create or use existing directories organized by topic (e.g., `models/`, `views/`, `tests/`)

2. **Create `.mdc` files** - Use the `.mdc` extension for files that should always be applied:

   ```yaml
   ---
   alwaysApply: true
   ---
   ```

3. **Follow the structure**:

   - Clear title and description
   - Examples showing ✅ correct and ❌ incorrect usage
   - Best practices section
   - Code examples with proper formatting

4. **Document patterns**:
   - Include real-world examples
   - Show before/after comparisons
   - Explain the reasoning behind rules
   - Add emoji indicators for visual clarity (✨, ✅, ❌, ⏭️)

### Updating Existing Rules

1. **Read existing files** - Understand the current structure and style
2. **Maintain consistency** - Follow the same format and tone
3. **Add examples** - Include code examples for clarity
4. **Update related files** - If changing a pattern, update all relevant files

### Best Practices for Contributions

- **Be specific** - Provide clear, actionable guidelines
- **Include examples** - Show both correct and incorrect usage
- **Explain why** - Help developers understand the reasoning
- **Keep it practical** - Focus on real-world scenarios
- **Use consistent formatting** - Follow the existing markdown style
- **Test examples** - Ensure code examples are valid and runnable

### File Naming Convention

- Use descriptive names: `commands-base.mdc`, `serializers-base.mdc`
- Use kebab-case for file names
- Group related files in directories
- Use `-base.mdc` for foundational rules
- Use `-structure.mdc` for structural guidelines
- Use `-example.mdc` for template examples

### Example Contribution

````markdown
---
alwaysApply: true
---

# Topic Name

Brief description of the rule or pattern.

## Rule Name

**Requirement**: Description of what must be done.

### ✅ Correct Usage

```python
# Example code
```
````

### ❌ Incorrect Usage

```python
# Example code
```

## Best Practices

- Point 1
- Point 2

```

### Submitting Changes

1. Create a clear commit message describing the change
2. Update this README if adding new sections
3. Ensure all examples are valid Django code
4. Test that the guidelines work as expected in Cursor

---

**Note**: These rules are designed to work with Cursor's AI code generation. Keep guidelines practical and focused on patterns that improve code quality and consistency.
```
