```python
#! /usr/bin/env python
# -*- coding: utf-8 -*-

"""
Copyright: Copyright (c) Huawei Technologies Co., Ltd. 2020-2020.
All rights reserved.
Create: 2020-11-04
Content: 运行日志压缩类
"""

import re
import os
import time
import json
import shutil
import _thread

from util import common_util, task_util
from tool.util.common_util import CommonUtil
from tool.kit_config import LogModuleConfig

LOGGER = common_util.get_logger()


class RunLogCompressStatus:
    """
    运行日志压缩状态码
    """
    SUCCESS = 0
    FAILED = 1
    RUNNING = 2
    PART_LOG_COMPRESSION = 3


class RunLogName:
    """
    各模块运行日志文件名
    """
    PORT_LOG = 'logs/porting.log'
    MAKE_LOG = 'logs/makefile.log'
    # 用于存放日志配置缺少后的日志
    DEFAULT_LOG = 'logs/default.log'

    @staticmethod
    def list_log_files():
        """
        遍历各模块日志文件名
        :return:
        """
        install_path = CommonUtil.get_customize_path().get_tool_root_dir()
        log_files = [os.path.join(install_path, RunLogName.MAKE_LOG),
                     os.path.join(install_path, RunLogName.PORT_LOG)]
        return log_files

    @staticmethod
    def get_other_log_paths():
        """
        获取Cmake、预编译、内嵌/全汇编、弱内存序模块日志文件路径
        :return:
        """
        original_log_paths = []
        install_path = CommonUtil.get_customize_path().get_tool_root_dir()
        for config_path in LogModuleConfig.log_cfg_list:
            log_config_path = os.path.join(install_path, config_path)
            log_file_path = RunLogName.read_log_config(log_config_path)
            if log_file_path:
                original_log_paths.append(log_file_path)
        default_log = os.path.join(install_path, RunLogName.DEFAULT_LOG)
        original_log_paths.append(default_log)
        return original_log_paths

    @staticmethod
    def read_log_config(log_config_path):
        """
        读取日志配置文件，拼接日志文件路径
        :param log_config_path: 日志配置文件路径
        :return:
        """
        if os.path.isfile(log_config_path):
            LOGGER.info('Decode log config: %s.', log_config_path)
            try:
                with open(log_config_path, 'r', errors='ignore') \
                        as log_config_file:
                    log_info = json.load(log_config_file)
                    log_path = log_info.get('logPath')
                    log_file = log_info.get('logFile')
                    if log_path and log_file:
                        log_file_path = os.path.join(log_path, log_file)
                        return log_file_path
            except json.decoder.JSONDecodeError as ex:
                LOGGER.error('Decode log config error: %s.', ex)
        return ''


class RunLogCompress:
    """
    拷贝所有的日志文件到指定目录，并进行压缩
    因统一日志模块使用bz2压缩耗时较长，下载的日志存在一定滞后性
    """

    DOWNLOAD_DIRECTORY = 'downloadlog'
    DOWNLOAD_TEMP_DIRECTORY = 'log'

    LOG_PACKAGE_FORMAT = 'zip'
    FINAL_LOG_PACKAGE = 'log.zip'

    LOGROTATE_FORMAT = 'bz2'

    def __init__(self, task_name, user_id, queue):
        self.task_name = task_name
        self.user_id = user_id
        self.queue = queue
        self.target_temp_path = ''
        self.target_path = ''
        self.pack_file = ''
        self.finish_compress = False
        self.compressing_log = []

    def compress_run_log(self):
        """
        压缩运行日志主函数
        :return:
        """
        LOGGER.info("Start to compress run logs.")
        _thread.start_new_thread(self._update_task_status, ())
        self._init_log_path()
        try:
            self._copy_log_files()
            self._compress_log_files()
        except Exception as ex:
            LOGGER.error("Compress log files error: %s.", ex)
            self._end_task_status(RunLogCompressStatus.FAILED)
        LOGGER.info("End compress run logs task.")

    def _init_log_path(self):
        """
        初始化日志文件路径
        :return:
        """
        install_path = CommonUtil.get_customize_path().get_tool_root_dir()
        self.target_temp_path = \
            os.path.join(install_path, self.DOWNLOAD_TEMP_DIRECTORY)
        if os.path.isdir(self.target_temp_path):
            task_util.rm_tree(self.target_temp_path)
        os.mkdir(self.target_temp_path)
        self.target_path = \
            os.path.join(install_path, self.DOWNLOAD_DIRECTORY)
        if os.path.isdir(self.target_path):
            task_util.rm_tree(self.target_path)
        elif os.path.isfile(self.target_path):
            os.remove(self.target_path)
        os.mkdir(self.target_path)
        self.pack_file = \
            os.path.join(install_path, self.FINAL_LOG_PACKAGE)

    def _copy_log_files(self):
        """
        拷贝日志文件到临时目录
        :return:
        """
        other_log_files = RunLogName.get_other_log_paths()
        original_log_files = RunLogName.list_log_files()
        for log_file in other_log_files + original_log_files:
            # 拷贝日志文件
            self._copy_file(log_file)
            # 拷贝日志回滚文件
            log_path, file = os.path.split(log_file)
            if log_file in original_log_files:
                pattern = r'^%s.\d.%s$' % (file, self.LOG_PACKAGE_FORMAT)
            else:
                pattern = r'^%s.\d{14}.%s$' % (file, self.LOGROTATE_FORMAT)
            self._copy_roll_files(log_path, pattern)
        if not os.listdir(self.target_temp_path):
            LOGGER.warning("No log file need to download.")

    def _copy_file(self, file):
        """
        拷贝文件
        :param file:
        :return:
        """
        if os.path.isfile(file):
            try:
                shutil.copy(file, self.target_temp_path)
                LOGGER.info('Copy file: %s to temporary directory.', file)
            except PermissionError as err:
                LOGGER.error('Copy log files error: %s.', err)

    def _copy_roll_files(self, log_path, pattern):
        """
        拷贝日志回滚文件
        :param log_path: 日志文件路径
        :param pattern: 回滚文件格式正则
        :return:
        """
        for log_file in os.listdir(log_path):
            if re.match(pattern, log_file):
                file_path = os.path.join(log_path, log_file)
                # bz2压缩需要十多秒，通过临时文件判断是否压缩完成
                backup_temp_log, _ = os.path.splitext(file_path)
                if not os.path.isfile(backup_temp_log):
                    self._copy_file(file_path)
                else:
                    self.compressing_log.append(file_path)
                    LOGGER.info('Log file: %s is being compressed, skip '
                                'the current file.', file_path)

    def _compress_log_files(self):
        """
        打包make、cmake、预处理、全汇编、内嵌汇编服务器运行日志成 log.zip
        :return:
        """
        try:
            shutil.make_archive(self.target_temp_path,
                                self.LOG_PACKAGE_FORMAT,
                                self.target_temp_path)
            LOGGER.info('Compress the log directory to log.zip successful.')
        except Exception as ex:
            LOGGER.error("Compress the log directory to log.zip error: "
                         "%s.", ex)
            self._end_task_status(RunLogCompressStatus.FAILED)
            return
        task_util.rm_tree(self.target_temp_path)
        if not os.path.isdir(self.target_path):
            os.mkdir(self.target_path)
        shutil.move(self.pack_file, self.target_path)
        if self.compressing_log:
            LOGGER.warning('Log files: %s are being compressed. Please '
                           'try again later.', self.compressing_log)
            self._end_task_status(RunLogCompressStatus.PART_LOG_COMPRESSION)
        else:
            LOGGER.info('All log files are compressed to log.zip.')
            self._end_task_status(RunLogCompressStatus.SUCCESS)

    def _update_task_status(self):
        """
        更新任务状态
        :return:
        """
        while True:
            if self.finish_compress:
                break
            task_status = \
                self._generate_task_status_json(RunLogCompressStatus.RUNNING)
            self.queue.put(task_status)
            time.sleep(1)

    def _end_task_status(self, status):
        """
        更新任务终止状态
        :return:
        """
        task_status = self._generate_task_status_json(status)
        self.queue.put(task_status)
        self.finish_compress = True

    def _generate_task_status_json(self, status):
        """
        生成任务状态字典
        :return:
        """
        task_status = {
            'task_name': self.task_name,
            'user_name': self.user_id,
            'status': status
        }
        return task_status
```

