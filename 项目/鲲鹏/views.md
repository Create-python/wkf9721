```python
urlpatterns = [
    path('', views.get_log_infos),
    path('create_log/', compress_log_views.create_download_logs_task),
    path('zip_log/', compress_log_views.compress_logs_task)
]

#!/usr/local/python3
# -*- encoding: utf-8 -*-
import configparser
import logging
import os
import re
import threading
import time

from rest_framework.decorators import api_view

from tool.logger.logger import Logger
from util.response import SuccessResponse, FailResponse
from util import common_util

LOGGER = common_util.get_logger()

MAX_COUNT = 99
LOG_DIR, LOG_FILE = os.path.split(Logger.log_output_path)

run_log_level = ""


@api_view(["GET"])
@common_util.check_if_user_login()
@common_util.check_is_admin_user()
def get_log_infos(_):
    """
    获取运行日志信息
    @param _:
    @return:
    """
    # 文件查询
    try:
        backup_count = get_backup_count()
        pattern, count = get_name_pattern_and_count(backup_count)
        file_list = [
            f for f in os.listdir(LOG_DIR) if file_condition(pattern, count, f)
        ]
    except (ValueError or FileNotFoundError) as error:
        LOGGER.warning("Run log configuration error. Except:%s", error)
        info = "Run log configuration error."
        info_chinese = "工作日志配置异常。"
        return FailResponse(info, info_chinese)

    # 构建响应体日志数据
    data = {'totalcount': len(file_list), 'loglist': file_list}

    info = "Run log query succeeded."
    info_chinese = u"查询运行日志成功。"

    return SuccessResponse(info, info_chinese, data)


def get_backup_count():
    """
    获取日志文件备份次数
    :return:
    """
    cp = configparser.ConfigParser(
        defaults={'filename': '%s' % Logger.log_output_path})
    cp.read(Logger.log_config_path)
    section = cp["handler_fileHandler"]
    args = section.get("args")
    backup_count = min(int(args.split(",")[3].strip()), MAX_COUNT)
    return backup_count


def get_name_pattern_and_count(backup_count):
    """
    获取日志文件匹配正则
    :param backup_count:
    :return:
    """
    try:
        if backup_count > 0:
            ret = r"^%s($|\.\d{1,%s}\.zip$)" \
                  % (LOG_FILE.replace(".", r"\."), len(str(backup_count)))
        else:
            ret = r"^%s$" % LOG_FILE.replace(".", r"\.")
        return ret, backup_count
    except Exception as e:
        LOGGER.error("Run log configuration error:%s.", e)
        raise ValueError("Run log configuration error: %s" % e)


def file_condition(pattern, count, file_name):
    """
    对日志文件名进行正则匹配
    :param pattern:
    :param count:
    :param file_name:
    :return:
    """
    if re.match(pattern, file_name):
        match = re.search(r"\d+", file_name)
        if not match or int(match.group()) <= count:
            return True

    return False


def start_log_level_sync():
    """
    修改运行日志级别主函数
    :return:
    """
    try:
        sync_thread = threading.Thread(target=run_log_level_sync_fun, args=())
        sync_thread.setDaemon(True)
        sync_thread.start()
    except SystemError as error:
        LOGGER.error("Start log level sync thread failed. Except:%s.", error)


def run_log_level_sync_fun():
    """
    修改运行日志配置文件
    :return:
    """
    global run_log_level
    while True:
        try:
            # 修改log配置文件
            log_config_path = Logger.log_config_path
            config = configparser.ConfigParser()
            config.read(log_config_path)
            level = config.get("logger_" + Logger.LOGGER_NAME, "level")
            if level != run_log_level:
                run_log_level = level
                Logger.reset_log_level(logging.getLogger(Logger.LOGGER_NAME),
                                       trans_log_level(run_log_level))
        except Exception as e:
            _ = e

        time.sleep(1)


def trans_log_level(level):
    """
    转换日志级别词条
    :param level:
    :return:
    """
    level_map = {"ERROR": "err", "WARNING": "warn"}
    return level_map[level] if level in level_map else level.lower()
```

