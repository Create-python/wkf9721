## 修改settings 文件中的路径信息

我们将Django的应用放到了`工程目录/mall/apps`目录下，如果创建一个应用，比如users，那么在配置文件的INSTALLED_APPS中注册应用应该如下：

```
INSTALLED_APPS = [
    ...
    'users.apps.UsersConfig',
]
```

因为我们将应用放到了apps文件件，所以要告知django去apps里查找，我们需要向Python解释器的导包路径中添加apps应用目录的路径。

使用`sys.path`添加`<BASE_DIR>/apps`目录，即可添加apps应用的导包路径。

```
import os
import sys

BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))

#让django找到 apps这个包
sys.path.insert(0,os.path.join(BASE_DIR,'apps'))
```

## INSTALLED_APPS

在INSTALLED_APPS中添加rest_framework

```
INSTALLED_APPS = [
    ...
    'rest_framework',
]
```

## 数据库

```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'HOST': '127.0.0.1',  # 数据库主机
        'PORT': 3306,  # 数据库端口
        'USER': 'root',  # 数据库用户名
        'PASSWORD': 'mysql',  # 数据库用户密码
        'NAME': 'meiduo_mall'  # 数据库名字
    }
}
```

注意：

记得在工程目录`/mall/__init__.py`文件中添加

```
import pymysql

pymysql.install_as_MySQLdb()
```

## Redis

安装django-redis，并配置

```
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/0",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        }
    },
    "session": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        }
    }
}
SESSION_ENGINE = "django.contrib.sessions.backends.cache"
SESSION_CACHE_ALIAS = "session"
```

除了名为default的redis配置外，还补充了名为session的redis配置，分别使用两个不同的redis库。

同时修改了Django的Session机制使用redis保存，且使用名为'session'的redis配置。

此处修改Django的Session机制存储主要是为了给Admin站点使用。

**关于django-redis 的使用，说明文档可见**[**http://django-redis-chs.readthedocs.io/zh_CN/latest/**](http://django-redis-chs.readthedocs.io/zh_CN/latest/)

## 本地化语言与时区

```
LANGUAGE_CODE = 'zh-Hans'


TIME_ZONE = 'Asia/Shanghai'
```

##  日志

创建在mall文件夹下创建logs包,用户存在日志文件

[文档](https://docs.djangoproject.com/en/1.11/topics/logging/)

```
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '%(levelname)s %(asctime)s %(module)s %(lineno)d %(message)s'
        },
        'simple': {
            'format': '%(levelname)s %(module)s %(lineno)d %(message)s'
        },
    },
    'filters': {
        'require_debug_true': {
            '()': 'django.utils.log.RequireDebugTrue',
        },
    },
    'handlers': {
        'console': {
            'level': 'DEBUG',
            'filters': ['require_debug_true'],
            'class': 'logging.StreamHandler',
            'formatter': 'simple'
        },
        'file': {
            'level': 'INFO',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': os.path.join(BASE_DIR, "logs/meiduo.log"),  # 日志文件的位置
            'maxBytes': 300 * 1024 * 1024,
            'backupCount': 10,
            'formatter': 'verbose'
        },
    },
    'loggers': {
        'django': {  # 定义了一个名为django的日志器
            'handlers': ['console', 'file'],
            'propagate': True,
        },
    }
}
```

##  异常处理

修改Django REST framework的默认异常处理方法，补充处理数据库异常和Redis异常。

新建utils/exceptions.py

```
from rest_framework.views import exception_handler as drf_exception_handler
import logging
from django.db import DatabaseError
from redis.exceptions import RedisError
from rest_framework.response import Response
from rest_framework import status

# 获取在配置文件中定义的logger，用来记录日志
logger = logging.getLogger('meiduo')

def exception_handler(exc, context):
    """
    自定义异常处理
    :param exc: 异常
    :param context: 抛出异常的上下文
    :return: Response响应对象
    """
    # 调用drf框架原生的异常处理方法
    response = drf_exception_handler(exc, context)

    if response is None:
        view = context['view']
        if isinstance(exc, DatabaseError) or isinstance(exc, RedisError):
            # 数据库异常
            logger.error('[%s] %s' % (view, exc))
            response = Response({'message': '服务器内部错误'}, status=status.HTTP_507_INSUFFICIENT_STORAGE)

    return response
```

配置文件中添加

```
REST_FRAMEWORK = {
    # 异常处理
    'EXCEPTION_HANDLER': 'utils.exceptions.exception_handler',
}
```

EXCEPTION_HANDLER设置

数据库连接错误，redis的异常

### REST framework定义的异常

- APIException 所有异常的父类
- ParseError 解析错误
- AuthenticationFailed 认证失败
- NotAuthenticated 尚未认证
- PermissionDenied 权限决绝
- NotFound 未找到
- MethodNotAllowed 请求方式不支持
- NotAcceptable 要获取的数据格式不支持
- Throttled 超过限流次数
- ValidationError 校验失败

