# 用户模型类

Django提供了认证系统，文档资料可参考此链接https://yiyibooks.cn/xx/Django_1.11.6/topics/auth/index.html

Django认证系统同时处理认证和授权。简单地讲，认证验证一个用户是否它们声称的那个人，授权决定一个通过了认证的用户被允许做什么。 这里的词语“认证”同时指代这两项任务，即Django的认证系统同时提供了认证机制和权限机制。

Django的认证系统包含：

- 用户
- 权限：二元（是/否）标志指示一个用户是否可以做一个特定的任务。
- 组：对多个用户运用标签和权限的一种通用的方式。
- 一个可配置的密码哈希系统
- 用户登录或内容显示的表单和视图
- 一个可插拔的后台系统

Django默认提供的认证系统中，用户的认证机制依赖Session机制，我们在本项目中将引入JWT认证机制，将用户的身份凭据存放在Token中，然后对接Django的认证系统，帮助我们来实现：

- 用户的数据模型
- 用户密码的加密与验证
- 用户的权限系统

### Django用户模型类

Django认证系统中提供了用户模型类User保存用户的数据，默认的User包含以下常见的基本字段：

- `username`

  必选。 150个字符以内。 用户名可能包含字母数字，`_`，`@`，`+.`和`-`个字符。在Django更改1.10：`max_length`从30个字符增加到150个字符。

- `first_name`

  可选（`blank=True`）。 少于等于30个字符。

- `last_name`

  可选（`blank=True`）。 少于等于30个字符。

- `email`

  可选（`blank=True`）。 邮箱地址。

- `password`

  必选。 密码的哈希及元数据。 （Django 不保存原始密码）。 原始密码可以无限长而且可以包含任意字符。

- `groups`

  与`Group`之间的多对多关系。

- `user_permissions`

  与`Permission`之间的多对多关系。

- `is_staff`

  布尔值。 指示用户是否可以访问Admin 站点。

- `is_active`

  布尔值。 指示用户的账号是否激活。 我们建议您将此标志设置为`False`而不是删除帐户；这样，如果您的应用程序对用户有任何外键，则外键不会中断。它不是用来控制用户是否能够登录。 在Django更改1.10：在旧版本中，默认is_active为False不能进行登录。

- `is_superuser`

  布尔值。 指定这个用户拥有所有的权限而不需要给他们分配明确的权限。

- `last_login`

  用户最后一次登录的时间。

- `date_joined`

  账户创建的时间。 当账号创建时，默认设置为当前的date/time。

##### 常用方法：

- `set_password`(*raw_password*)

  设置用户的密码为给定的原始字符串，并负责密码的。 不会保存`User`对象。当`None`为`raw_password`时，密码将设置为一个不可用的密码。

- `check_password`(*raw_password*)

  如果给定的raw_password是用户的真实密码，则返回True，可以在校验用户密码时使用。

##### 管理器方法：

管理器方法即可以通过`User.objects.`进行调用的方法。

- `create_user`(*username*,*email=None*,*password=None*,_*_extra_fields*)

  创建、保存并返回一个`User`对象。

- `create_superuser`(*username*,*email*,*password*,_*_extra_fields*)

  与`create_user()`相同，但是设置`is_staff`和`is_superuser`为`True`。

### 创建自定义的用户模型类

Django认证系统中提供的用户模型类及方法很方便，我们可以使用这个模型类，但是字段有些无法满足项目需求，如本项目中需要保存用户的手机号，需要给模型类添加额外的字段。

Django提供了`django.contrib.auth.models.AbstractUser`用户抽象模型类允许我们继承，扩展字段来使用Django认证系统的用户模型类。

**我们现在在**`工程目录/meiduo_mall/apps`**中创建Django应用users，并在配置文件中注册users应用。**

在创建好的应用models.py中定义用户的用户模型类。

```
from django.db import models
from django.contrib.auth.models import AbstractUser
# Create your models here.
class User(AbstractUser):
    """用户模型类"""
    mobile = models.CharField(max_length=11, unique=True, verbose_name='手机号')

    class Meta:
        db_table = 'tb_users'
        verbose_name = '用户'
        verbose_name_plural = verbose_name
```

我们自定义的用户模型类还不能直接被Django的认证系统所识别，需要在配置文件中告知Django认证系统使用我们自定义的模型类。

在配置文件中进行设置

[文档](https://yiyibooks.cn/xx/Django_1.11.6/topics/auth/customizing.html)

```
AUTH_USER_MODEL = 'users.User'
```

`AUTH_USER_MODEL`参数的设置以`点.`来分隔，表示`应用名.模型类名`。只能有一个点。

**注意：Django建议我们对于**`AUTH_USER_MODEL`**参数的设置一定要在第一次数据库迁移之前就设置好，否则后续使用可能出现未知错误。**

执行数据库迁移

```
python manage.py makemigrations
python manage.py migrate
```