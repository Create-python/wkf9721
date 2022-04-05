```python
#! /usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Copyright: Copyright (c) Huawei Technologies Co., Ltd. 2020-2020.
All rights reserved.
Create: 2020-08-20
Content: 白名单加载类"""
import logging
from collections import defaultdict

from tool.logger.logger import Logger
from .whitelist_loader import WhitelistLoader

LOGGER = logging.getLogger(Logger.LOGGER_NAME)


class SoWhitelistLoader(WhitelistLoader):
    """
    白名单加载类子类，用于so匹配格式的加载    """
    def load_whitelist(self, whitelist_file):
        """
        加载so匹配格式的白名单        :param whitelist_file: 白名单文件路径        :return: 白名单字典        """
        LOGGER.info("Start loading whitelist for so libraries.")
        whitelist_dict = dict()
        so_name_dict = defaultdict(list)
        if not self.whitelist_exist(whitelist_file):
            return whitelist_dict, so_name_dict
        try:
            whitelist = self.decode_whitelist(whitelist_file)
            LOGGER.info('Decode the whitelist successful.')
            for line in whitelist:
                if self.whitelist_line_check(line):
                    continue
                # for full match
                lib_info_key = (
                    line[self.WHITELIST_FORMAT.get('libname')],
                    line[self.WHITELIST_FORMAT.get('libversion')])
                whitelist_dict[lib_info_key] = [
                    str(line[self.WHITELIST_FORMAT.get('level')]),
                    line[self.WHITELIST_FORMAT.get('url')]]
                # for fuzzy match
                if line[self.WHITELIST_FORMAT.get('libversion')] != 'd':
                    so_name = line[self.WHITELIST_FORMAT.get('libname')]
                    so_name_dict[so_name].append([
                        line[self.WHITELIST_FORMAT.get('libversion')],
                        str(line[self.WHITELIST_FORMAT.get('level')]),
                        line[self.WHITELIST_FORMAT.get('url')]
                    ])
            LOGGER.info("Loading whitelist for so libraries successful.")
        except IOError as err:
            whitelist_dict = {}
            so_name_dict = defaultdict(list)
            LOGGER.error("Load whitelist error: %s", err)
        return whitelist_dict, so_name_dict

```

