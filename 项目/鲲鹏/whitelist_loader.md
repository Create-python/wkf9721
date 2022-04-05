```Python
#! /usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Copyright: Copyright (c) Huawei Technologies Co., Ltd. 2020-2020.
All rights reserved.
Create: 2020-08-20
Content: 白名单加载类"""
import logging
import os

from tool.logger.logger import Logger
from tool.util.common_decode import DataParse
from tool.kit_config import KitConfig

LOGGER = logging.getLogger(Logger.LOGGER_NAME)


class WhitelistLoader:
    """
    白名单加载类，与白名单匹配类独立，降低耦合    """
    WHITELIST_FORMAT = {
        'libname': 0,
        'libversion': 1,
        'release': 2,
        'url': 3,
        'level': 4,
        'os_name': 5,
        'os_version': 6,
        'kernel_version': 7,
        'so_name': 8
    }

    def load_whitelist(self, *args, **kwargs):
        """ 加载so匹配格式的白名单 """
    @staticmethod
    def whitelist_exist(whitelist_file):
        """
        判断白名单是否存在        :param whitelist_file: 白名单路径        :return: True/False
        """
        if not os.path.exists(whitelist_file):
            LOGGER.warning(
                "Whitelist path does not exist, create a blank whitelist.")
            return False
        if not os.path.getsize(whitelist_file):
            LOGGER.warning("The whitelist is empty, "
                           "returning to the blank scan dictionary.")
            return False
        return True
    def whitelist_line_check(self, line):
        """
        白名单格式校验        :param line: 白名单的每一行        :return: True/False
        """
        if len(line) < self.WHITELIST_FORMAT.get('kernel_version'):
            return True
        if not line[self.WHITELIST_FORMAT.get('libname')] or \
                not line[self.WHITELIST_FORMAT.get('os_name')] or \
                not line[self.WHITELIST_FORMAT.get('os_version')] or \
                not line[self.WHITELIST_FORMAT.get('level')]:
            return True
        return False
    @staticmethod
    def decode_whitelist(whitelist_file):
        """
        解密白名单        :param whitelist_file: 白名单文件        :return: 解密后的白名单文件        """
        DataParse.decode_config(KitConfig.root_dir)
        input_whitelist = DataParse.parse_binary(whitelist_file)[0]
        whitelist = DataParse.str_to_list(input_whitelist)
        return whitelist

```

