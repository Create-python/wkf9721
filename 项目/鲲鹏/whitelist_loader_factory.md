```python
#! /usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Copyright: Copyright (c) Huawei Technologies Co., Ltd. 2020-2020.
All rights reserved.
Create: 2020-08-20
Content: 白名单加载工厂类"""
from enum import unique, IntEnum

from .so_whitelist_loader import SoWhitelistLoader
from .jar_whitelist_loader import JarWhitelistLoader
from .makefile_whitelist_loader import MakefileWhitelistLoader
from .none_whitelist_loader import NoneWhitelistLoader


@unique
class WhitelistLoaderType(IntEnum):
    """
    白名单加载器类型    """
    SO = 1
    JAR = 2
    MAKEFILE = 3
class WhitelistLoaderFactory:
    """
    白名单加载工厂类    """
    @staticmethod
    def generate_loader(loader_type):
        """ 获取白名单加载类 """
        if loader_type == WhitelistLoaderType.SO:
            return SoWhitelistLoader()
        elif loader_type == WhitelistLoaderType.JAR:
            return JarWhitelistLoader()
        elif loader_type == WhitelistLoaderType.MAKEFILE:
            return MakefileWhitelistLoader()
        else:
            return NoneWhitelistLoader()

```

