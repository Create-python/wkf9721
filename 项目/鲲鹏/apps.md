```python
#!/usr/local/python3
# -*- encoding: utf-8 -*-
from django.apps import AppConfig
from django.utils.module_loading import autodiscover_modules


class RunlogConfig(AppConfig):
    """
    set app
    """
    name = 'runlog'

    def ready(self):
        autodiscover_modules('sync.py')
```

