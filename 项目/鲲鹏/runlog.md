```python
#!/usr/local/python3
# -*- encoding: utf-8 -*-
import configparser
import logging
import os
import re
import threading
import time
from wsgiref.util import FileWrapper

from rest_framework.decorators import api_view
from django.http import FileResponse

from tool.logger.logger import Logger
from util.response import SuccessResponse, FailResponse
from util import common_util


LOGGER = common_util.get_logger()

STATUS_SUCCESS = 0
STATUS_FAILED = 1
ADMIN_USER_ID = 1
MAX_COUNT = 99
LOG_DIR, LOG_FILE = os.path.split(Logger.log_output_path)

run_log_level = ""


def get_name_pattern_and_count():
    try:
        cp = configparser.ConfigParser(
            defaults={'filename': '%s' % Logger.log_output_path})
        cp.read(Logger.log_config_path)
        section = cp["handler_fileHandler"]
        args = section.get("args")
        backup_count = min(int(args.split(",")[3].strip()), MAX_COUNT)
        if backup_count > 0:
            ret = r"^%s($|\.\d{1,%s}\.zip$)" \
                  % (LOG_FILE.replace(".", r"\."), len(str(backup_count)))
        else:
            ret = r"^%s$" % LOG_FILE.replace(".", r"\.")
        return ret, backup_count
    except Exception as e:
        raise ValueError("run log configuration error: %s" % e)


def file_condition(file_name):
    pattern, count = get_name_pattern_and_count()
    if re.match(pattern, file_name):
        match = re.search(r"\d+", file_name)
        if not match or int(match.group()) <= count:
            return True

    return False


@api_view(["GET"])
@common_util.check_if_user_login()
@common_util.check_is_admin_user()
def get_log_infos(_):
    """

    @param request:
    @return:
    """
    # 文件查询
    try:
        file_list = [
            f for f in os.listdir(LOG_DIR) if file_condition(f)
        ]
    except (ValueError or FileNotFoundError):
        info = "Run log configuration error."
        info_chinese = "工作日志配置异常。"
        return FailResponse(info, info_chinese)

    # 构建响应体日志数据
    data = {'totalcount': len(file_list), 'loglist': file_list}

    info = "Run log query succeeded."
    info_chinese = u"查询运行日志成功。"

    return SuccessResponse(info, info_chinese, data)


@api_view(["GET"])
@common_util.check_if_user_login()
@common_util.check_is_admin_user()
def download(request):
    """

    @param request:
    @return:
    """
    file_name = request.GET.get("filename")
    if not file_name:
        file_name = LOG_FILE

    file_path = os.path.join(LOG_DIR, file_name)

    try:
        if not file_condition(file_name) \
                or not os.path.isfile(file_path):
            info = "File not found"
            info_chinese = u"文件不存在"
            return FailResponse(info, info_chinese)
    except (ValueError or FileNotFoundError):
        info = "run log configuration error"
        info_chinese = "工作日志配置异常。"
        return FailResponse(info, info_chinese)

    wrapper = FileWrapper(open(file_path, "rb"))
    response = FileResponse(wrapper)
    response['Content-Type'] = 'application/octet-stream'
    response['Content-Length'] = os.path.getsize(file_path)
    response['Content-Disposition'] = 'attachment;filename="%s"' % file_name

    return response


def trans_log_level(level):
    level_map = {"ERROR": "err", "WARNING": "warn"}
    return level_map[level] if level in level_map else level.lower()


def run_log_level_sync_fun():
    global run_log_level
    while True:
        # 修改log配置文件
        log_config_path = Logger.log_config_path
        config = configparser.ConfigParser()
        config.read(log_config_path)
        level = config.get("logger_" + Logger.LOGGER_NAME, "level")
        if level != run_log_level:
            run_log_level = level
            Logger.reset_log_level(logging.getLogger(Logger.LOGGER_NAME),
                                   trans_log_level(run_log_level))

        time.sleep(1)


def start_log_level_sync():
    try:
        sync_thread = threading.Thread(target=run_log_level_sync_fun, args=())
        sync_thread.setDaemon(True)
        sync_thread.start()
    except SystemError:
        LOGGER.error("start log level sync thread failed")


@api_view(["GET"])
@common_util.check_if_user_login()
@common_util.check_is_admin_user()
def download_all(request):
    """

    @param request:
    @return:
    """

```

