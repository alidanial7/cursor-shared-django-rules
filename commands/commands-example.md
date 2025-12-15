---
alwaysApply: true
---

# Django Command Example

A minimal Django management command template:

```python
from django.core.management.base import BaseCommand


class Command(BaseCommand):
    help = "Description of what this command does"

    def add_arguments(self, parser):
        parser.add_argument('--dry-run', action='store_true', help="Run without making changes")
        parser.add_argument('--verbose', '-v', action='store_true', help="Verbose output")

    def handle(self, *args, **options):
        if options['dry_run']:
            self.stdout.write(self.style.WARNING('Dry run mode - no changes will be made'))

        # Your command logic here

        self.stdout.write(self.style.SUCCESS('Command completed successfully!'))
```

## Running Commands

```bash
python manage.py your_command --dry-run
python manage.py your_command --verbose
```
