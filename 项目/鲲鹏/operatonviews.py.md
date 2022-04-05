```python
"""
operation log
"""

from __future__ import unicode_literals
import os
import time
import csv
import threading
from datetime import datetime, timedelta
from rest_framework.decorators import api_view

from resource import utils
from operationlog.models import Operation
from util.response import SuccessResponse
from util import common_util
from tool.util.common_util import CommonUtil
from usermanager.models import User

customize_path = CommonUtil.get_customize_path()

# 请求状态码
STATUS_SUCCESS = 0
STATUS_FAILED = 1

# 操作日志清理天数
OPLOG_RETAIN_DAYS = 30
# 操作日志数目阈值
OPLOG_MAX_NUM = 2000

LOGGER = utils.get_logger()


# Create your views here.
@api_view(['GET'])
@common_util.check_if_user_login()
def getlogs(request):
    """
    get log
    :param request:
    :return:
    """
    LOGGER.debug('Log management')
    # 查询日志个数
    user = request.user
    log_list = get_logs(user, Operation.objects.all())
    totalcounts = len(log_list)
    # 构建响应体日志数据
    data = {'totalcount': totalcounts, 'loglist': log_list}

    info = "Operation log query succeeded."
    info_chinese = u"查询操作日志成功。"

    # 构建响应体数据
    return SuccessResponse(info, info_chinese, data)


def get_logs(user, logs):
    """
    :日志列表内容转换
    :param logs: 查询数据库获取到的操作日志
    :return: 处理后的操作日志list
    """
    # 查询日志列表
    log_queryset = logs.values("username",
                               "event",
                               "result",
                               "time",
                               "detail",
                               "ip_detail",
                               "op_username").order_by('-time')
    # queryset -> list
    if user.get_id() != 1:
        log_queryset = log_queryset.filter(username=user.username)
    log_list = list(log_queryset)

    # 把datetime 类型的转换成对应的显示字符串
    for opt_iter in log_list:
        opt_iter['time'] = opt_iter['time'].strftime('%Y-%m-%d %H:%M:%S')
        opt_iter['event'] = Operation.EVENT_CHOIECES[int(opt_iter['event'])][1]
        opt_iter['result'] = Operation.RESULT_CHOIECES[int(
            opt_iter['result'])][1]
        opt_iter['detail'] = Operation.DETAIL_CHOIECES[int(
            opt_iter['detail'])][1]
        opt_iter['detail'] = opt_iter['detail']\
            .replace('__', opt_iter['ip_detail'])\
            .replace('--', opt_iter['op_username'])

    return log_list


def startlogcleaner():
    """
    :return:start log clean
    """
    try:
        cleanthread = threading.Thread(target=logcleanjob, args=())
        cleanthread.setDaemon(True)
        cleanthread.start()
    except SystemError as error:
        LOGGER.error("Start log clean thread failed. Except:%s.", error)


def del_timeout_operationlog():
    """
    删除30天前零点之前的操作记录
    """
    # 删除30天前零点之前的操作记录
    now = datetime.now()
    delete_time_point = now - timedelta(days=OPLOG_RETAIN_DAYS,
                                        hours=now.hour,
                                        minutes=now.minute,
                                        seconds=now.second,
                                        microseconds=now.microsecond)
    Operation.objects.filter(time__lt=delete_time_point).delete()


def logcleanjob():
    """
    每天死循环等到凌晨1点，触发清理日志函数
    """
    while True:
        time.sleep(300)
        # 清理超量日志
        # 安全要求，对操作日志表进行数量限制
        del_oversize_operationlog()

        # 清理超时日志
        now = datetime.now()
        if now.hour == 1:
            del_timeout_operationlog()


def del_oversize_operationlog():
    """
    oversize operation log clean job
    """
    count = Operation.objects.count()
    if count >= OPLOG_MAX_NUM:
        try:
            # 获取logs文件夹路径，将清理日志生成的临时文件存在此处
            path = os.path.dirname(customize_path.get_log_path())
            # 如果路径不存在
            if not os.path.exists(path):
                os.makedirs(path)
            # 将老数据写入到csv中保存
            with open(os.path.join(path, 'his_operation_log.csv'), mode='w+',
                      encoding='utf-8', newline='') as log_file:
                logs = Operation.objects.all()
                header = ["username", "event", "result", "time", "detail"]
                csv_writer = csv.writer(log_file, quoting=1)
                user = User.objects.filter(role='Admin').first()
                log_list = get_logs(user, logs)
                csv_writer.writerow(header)
                for row in log_list:
                    csv_writer.writerow(
                        [row["username"], row["event"], row["result"],
                         row["time"], row["detail"]])
        except Exception as error:
            LOGGER.error("Log clean failed. Except:%s.", error)
        finally:
            # 清除老数据
            logs.delete()
```

