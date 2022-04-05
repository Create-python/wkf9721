```python
#! /usr/bin/env python
# -*- coding: utf-8 -*-

"""
Copyright: Copyright (c) Huawei Technologies Co., Ltd. 2020-2020.
All rights reserved.
Create: 2020-11-04
Content: 运行日志压缩、下载接口
"""

from datetime import datetime

from django.db.models import Q
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response

from resource import utils
from taskmanager.models import Task
from util import task_util, common_util
from util.check_space import check_space
from util.response import CommonResponse, SuccessResponse, FailResponse
from .run_log_task import RunLogCompressTask, RunLogCompressTaskQueue
from .run_log_compress import RunLogCompressStatus

LOGGER = utils.get_logger()
# 请求状态码
_STATUS_SUCCESS = 0
_STATUS_FAILED = 1


RESPONSE_INFO = {
    "create_successful": {
        "en": "Task creation successful.",
        "cn": u"创建任务成功。"
    },
    "create_failed": {
        "en": "Task creation failed.",
        "cn": u"创建任务失败。"
    },
    "another_task_running": {
        "en": "A task is running.",
        "cn": u"有任务正在运行。"
    },
    "all_task_complete": {
        "en": "Task execution complete.",
        "cn": u"任务已全部执行完成。"
    },
    "task_not_exist": {
        "en": "The task does not exist.",
        "cn": u"任务不存在。"
    },
    "compression_success": {
        "en": "Log file compression successfully.",
        "cn": u"日志文件压缩成功。"
    },
    "compression_failure": {
        "en": "Log file compression failed.",
        "cn": u"日志文件压缩失败。"
    },
    "in_the_compression": {
        "en": "Log file is in the compression.",
        "cn": u"日志文件压缩中。"
    },
    "part_log_compression": {
        "en": "Some log files are being dumped and compressed and "
              "cannot be downloaded. Try again later.",
        "cn": u"部分日志文件转储压缩中，当前无法下载，请稍后再试。"
    }
}


@api_view(["POST"])
@common_util.check_if_user_login()
@common_util.check_is_admin_user()
def create_download_logs_task(request):
    """
    创建运行日志下载任务
    :param request:
    :return:
    """
    request.user.update_op_time()
    response = check_space()
    if response:
        return Response(response, status.HTTP_403_FORBIDDEN)
    return task_build(request)


@api_view(["GET", "DELETE"])
@common_util.check_if_user_login()
@common_util.check_is_admin_user()
def compress_logs_task(request):
    """
    获取压缩日志任务状态或中止压缩日志任务
    :param request:
    :return:
    """
    request.user.update_op_time()
    task_name = request.GET.get("task_id", "")
    if request.method == "GET":
        return get_compress_logs_task_status(request, task_name)
    return abort_compress_logs_task(request, task_name)


def get_compress_logs_task_status(request, task_name):
    """
    获取压缩日志任务状态
    :param request:
    :param task_name:
    :return:
    """
    if not task_name:
        return check_download_logs_task(request)
    return get_running_compress_logs_task_status(request, task_name)


def check_download_logs_task(request):
    """
    查询当前是否有日志压缩下载任务在进行
    :param request:
    :return:
    """
    request.user.update_op_time()
    LOGGER.debug('Query download run logs task undone.')
    info_key = "all_task_complete"
    data = _generate_response_data()
    task_dict = Task.objects.filter(
        user_name=request.user,
        task_type=task_util.PORT_RUN_LOG_DOWNLOAD,
        status=RunLogCompressStatus.RUNNING
    ).first()
    if task_dict:
        info_key = "another_task_running"
        data = _generate_response_data(
            task_dict.task_name, task_dict.os_mapping_dir)
        LOGGER.info('Download run logs task %s is running.',
                    task_dict.task_name)
    return SuccessResponse(
        RESPONSE_INFO[info_key]['en'], RESPONSE_INFO[info_key]['cn'], data)


def get_running_compress_logs_task_status(request, task_name):
    """
    获取正在压缩日志任务的状态
    :param request:
    :param task_name:
    :return:
    """
    task = Task.objects.filter(
        task_name=task_name,
        user_name=request.user,
        task_type=task_util.PORT_RUN_LOG_DOWNLOAD
    ).first()
    if not task:
        info_key = 'task_not_exist'
        return FailResponse(RESPONSE_INFO[info_key]['en'],
                            RESPONSE_INFO[info_key]['cn'])
    data = {'status': task.status}
    if task.status == RunLogCompressStatus.SUCCESS:
        status_code = _STATUS_SUCCESS
        info_key = 'compression_success'
    elif task.status == RunLogCompressStatus.FAILED:
        status_code = _STATUS_FAILED
        info_key = 'compression_failure'
    elif task.status == RunLogCompressStatus.PART_LOG_COMPRESSION:
        data = {'status': _STATUS_SUCCESS}
        status_code = _STATUS_SUCCESS
        info_key = 'part_log_compression'
    else:
        status_code = _STATUS_SUCCESS
        info_key = 'in_the_compression'
    return CommonResponse(status_code, RESPONSE_INFO[info_key]['en'],
                          RESPONSE_INFO[info_key]['cn'], data)


def abort_compress_logs_task(request, task_name):
    """
    中止压缩日志任务
    :param request:
    :param task_name:
    :return:
    """
    return task_util.stop(request, task_name, task_util.PORT_RUN_LOG_DOWNLOAD)


def task_build(request):
    """
    创建运行日志下载主体函数
    :param request:
    :return:
    """
    LOGGER.info('Run log download task begin.')
    download_path = request.data.get('download_path', '')
    user_name = request.user.username
    task_name = datetime.now().strftime('%Y%m%d%H%M%S')
    status_code = _STATUS_FAILED
    data = {'task_id': ''}
    info_key = 'create_failed'

    download_task = Task.objects.filter(
        Q(user_name=user_name) &
        Q(task_type=task_util.PORT_RUN_LOG_DOWNLOAD) &
        (Q(status=RunLogCompressStatus.RUNNING) | Q(task_name=task_name))
    ).exists()
    if download_task:
        LOGGER.error('Another run log download task is running, '
                     'task create failed.')
        info_key = 'another_task_running'
        utils.insert_operationlog(
            user_name, 54, 1, datetime.now(), [107, ''])
    else:
        task = Task(
            task_name=task_name,
            user_name=user_name,
            status=RunLogCompressStatus.RUNNING,
            task_type=task_util.PORT_RUN_LOG_DOWNLOAD,
            os_mapping_dir=download_path
        )
        task.save()
        try:
            run_task = RunLogCompressTask(user_name, task_name)
            if RunLogCompressTaskQueue().add_task(run_task):
                status_code = _STATUS_SUCCESS
                info_key = 'create_successful'
                data = {'task_id': task_name}
            else:
                task.delete()
                utils.insert_operationlog(
                    user_name, 54, 1, datetime.now(), [107, ''])
        except (IOError, OSError, ValueError, TypeError) as exp:
            LOGGER.error('Task created failed: %s', str(exp))
            utils.insert_operationlog(
                user_name, 54, 1, datetime.now(), [107, ''])
    return CommonResponse(status_code, RESPONSE_INFO[info_key]['en'],
                          RESPONSE_INFO[info_key]['cn'], data)


def _generate_response_data(task_id="", download_path=""):
    """
    生成返回的data
    :param task_id:
    :param download_path:
    :return:
    """
    data = {
        'task_id': task_id,
        'download_path': download_path
    }
    return data

```

