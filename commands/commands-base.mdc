---
alwaysApply: true
---

# Django Management Commands

Django provides a system for creating custom management commands that can be executed via `manage.py`. These commands are useful for database maintenance, data import/export, automation scripts, and repetitive tasks.

## Command Types

### 1. Project-Scoped Commands

For commands that operate at the project level (not tied to a specific app):

- Create a dedicated app named `commands`
- Place commands in `commands/management/commands/`
- Each command should be in its own Python file

```
project/
├── commands/
│   ├── __init__.py
│   └── management/
│       ├── __init__.py
│       └── commands/
│           ├── __init__.py
│           ├── cleanup_sessions.py
│           └── generate_report.py
```

### 2. App-Scoped Commands

For commands specific to an app's functionality:

- Place commands in `<app_name>/management/commands/`
- Each command should be in its own Python file

```
myapp/
├── management/
│   ├── __init__.py
│   └── commands/
│       ├── __init__.py
│       └── sync_data.py
```

## Best Practices

- One command per file
- Use descriptive command names (snake_case)
- Include help text in the command class
- Handle errors gracefully with proper exit codes
- Use `self.stdout.write()` for output instead of `print()`
- Use `self.style` helpers for colored output:
  - `self.style.SUCCESS('message')` - green
  - `self.style.WARNING('message')` - yellow
  - `self.style.ERROR('message')` - red
  - `self.style.NOTICE('message')` - cyan

## Command Base Classes

- **`BaseCommand`** - The standard base class for most commands. Implement `handle()` and optionally `add_arguments()`.

- **`LabelCommand`** - For commands that operate on a list of strings. Implement `handle_label(label, **options)` instead of `handle()`. Example: `python manage.py process_files file1.txt file2.txt`

- **`AppCommand`** - For commands that operate on Django apps. Validates arguments are installed apps. Implement `handle_app_config(app_config, **options)`. Example: `python manage.py migrate_app users orders`
