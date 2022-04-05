```python
"""证书Model"""
from django.db import models

# Create your models here.


class Cert(models.Model):
    STATE_VALID = 1  # 证书有效
    STATE_EXPIRING = 0  # 证书即将过期
    STATE_EXPIRED = -1  # 证书已过期

    cert_name = models.CharField(max_length=255, unique=True)  # 证书名称
    cert_expired = models.DateTimeField()  # 证书过期时间
    cert_flag = models.CharField(max_length=2)  # 证书状态: 1未到期,0即将到期,-1过期

```

