```python
#!/usr/local/python3
# -*- encoding: utf-8 -*-
import os
import shutil

from util.response import SuccessResponse, FailResponse
from util import common_util
from util.operation_log_util import save_operation_log
from tool.kit_config import KitConfig

LOGGER = common_util.get_logger()

PORT_SCANNING = 0
PORT_AUTOPACK = 1
PORT_WHITELIST = 2
PORT_SOLUTION = 3
PORT_XMLMANAGE = 4
PORT_MIGRATION = 5
PORT_BYTE_ALIGNMENT = 6
PORT_BINARY_SCANNING = 7
PORT_RUN_LOG_DOWNLOAD = 8
PORT_COMPILE_FILE = 9
PORT_WEAK_CONSISTENCY = 10

DEP_SCANNING = 1
DEP_WHITELIST = 2

OP_SUCCESS = 0
OP_FAILED = 1

COUNT = 10

SOURCE_CODE = 'sourcecode'
BYTE_CHECK = 'bytecheck'
PRE_CHECK = 'precheck'
PACKAGE_REBUILD = 'packagerebuild'
DATA = 'data'
PACKAGE = 'package'
WEAK_CONSISTENCY = 'weakconsistency'

if KitConfig.tool_name == KitConfig.DEPENDENCY:
    from scanning.models import Task

    STATUS_MAP = {
        DEP_SCANNING: (2,),  # scanning
        DEP_WHITELIST: ()    # whitelist
    }
    NEED_DELETE_REPORT_TYPE = (DEP_SCANNING,)
    OP_EVENT = 27
    STOPPABLE_TYPE = (DEP_SCANNING,)
    OP_DETAIL_MAP = {
        OP_SUCCESS: {
            DEP_SCANNING: "The scan task has been terminated successfully."},
        OP_FAILED: {
            DEP_SCANNING: "Failed to terminate the scan task."}
    }
    SCAN_TYPE = {
        '0': PACKAGE,
        '1': SOURCE_CODE
    }
    RULE_PATH = (SOURCE_CODE, PACKAGE)
    REPORT_PATH = {DEP_SCANNING: ""}
else:
    from taskmanager.models import Task

    STOPPABLE_TYPE = (
        PORT_SCANNING, PORT_AUTOPACK, PORT_SOLUTION,
        PORT_MIGRATION, PORT_BYTE_ALIGNMENT, PORT_BINARY_SCANNING,
        PORT_RUN_LOG_DOWNLOAD, PORT_COMPILE_FILE,
        PORT_WEAK_CONSISTENCY
    )
    STATUS_MAP = {
        PORT_SCANNING: (1,),  # scanning
        PORT_AUTOPACK: (-1, 1,),  # autopack
        PORT_WHITELIST: (),  # whiltelist
        PORT_SOLUTION: (1, 2),  # solution
        PORT_XMLMANAGE: (),  # xmlmanage
        PORT_MIGRATION: (1,),  # migrationscan
        PORT_BYTE_ALIGNMENT: (1,),
        PORT_BINARY_SCANNING: (2,),
        PORT_RUN_LOG_DOWNLOAD: (2,),
        PORT_COMPILE_FILE: (2,),
        PORT_WEAK_CONSISTENCY: (2,)
    }
    NEED_DELETE_REPORT_TYPE = (PORT_SCANNING, PORT_BINARY_SCANNING)
    OP_EVENT = 35
    OP_DETAIL_MAP = {
        OP_SUCCESS: {
            PORT_SCANNING: 72,  # scanning
            PORT_AUTOPACK: 73,  # autopack
            PORT_WHITELIST: -1,  # whiltelist
            PORT_SOLUTION: 74,  # solution
            PORT_XMLMANAGE: -1,  # xmlmanage
            PORT_MIGRATION: 75,  # migrationscan
            PORT_BYTE_ALIGNMENT: 76,
            PORT_BINARY_SCANNING: 72,
            PORT_RUN_LOG_DOWNLOAD: 108
        },
        OP_FAILED: {
            PORT_SCANNING: 77,  # scanning
            PORT_AUTOPACK: 78,  # autopack
            PORT_WHITELIST: -1,  # whiltelist
            PORT_SOLUTION: 79,  # solution
            PORT_XMLMANAGE: -1,  # xmlmanage
            PORT_MIGRATION: 80,  # migrationscan
            PORT_BYTE_ALIGNMENT: 81,
            PORT_BINARY_SCANNING: 77,
            PORT_RUN_LOG_DOWNLOAD: 109
        }

    }
    SCAN_TYPE = {
        '0': SOURCE_CODE,
        '1': BYTE_CHECK,
        '2': PRE_CHECK,
        '3': PACKAGE_REBUILD,
        '4': DATA,
        '5': PACKAGE,
        '6': WEAK_CONSISTENCY,
        '7': WEAK_CONSISTENCY
    }
    REPORT_PATH = {
        PORT_SCANNING: SOURCE_CODE,
        PORT_BINARY_SCANNING: PACKAGE,
        PORT_WEAK_CONSISTENCY: WEAK_CONSISTENCY
    }
    RULE_PATH = (SOURCE_CODE, BYTE_CHECK, PRE_CHECK,
                 PACKAGE_REBUILD, PACKAGE, DATA, WEAK_CONSISTENCY)


def stop(request, task_id, task_type):
    if task_type not in STOPPABLE_TYPE:
        info = "stoppable task type"
        info_chinese = "当前任务不支持终止"
        return FailResponse(info, info_chinese)

    try:
        if KitConfig.tool_name == KitConfig.DEPENDENCY:
            task = Task.objects.get(
                task_name=task_id, user_id=request.user.id,
                task_type=task_type)
        else:
            task = Task.objects.get(
                task_name=task_id, user_name=request.user.username,
                task_type=task_type)
    except Task.DoesNotExist as error:
        LOGGER.warning("Task not exist. Except:%s.", error)
        info = "task not exist"
        info_chinese = "任务不存在"
        _task_op_log(request, task_type, OP_FAILED)
        return FailResponse(info, info_chinese)

    if task.status not in STATUS_MAP[task_type]:
        info = "task has already finished"
        info_chinese = "任务已结束"
        _task_op_log(request, task_type, OP_FAILED)
        return FailResponse(info, info_chinese)

    _delete_db_obj(task)
    _task_op_log(request, task_type, OP_SUCCESS)
    info = "Task canceled."
    info_chinese = u"任务已取消。"
    return SuccessResponse(info, info_chinese)


def _delete_db_obj(task):
    try:
        task.delete()
    except Task.DoesNotExist as error:
        LOGGER.info("Task not in db. Except:%s.", error)


def delete_report(workspace, task_name, task_type):
    if task_type not in NEED_DELETE_REPORT_TYPE:
        return
    report_path = \
        os.path.join(workspace, "report",
                     REPORT_PATH[task_type], task_name)
    try:
        shutil.rmtree(report_path)
    except OSError or IOError as e:
        LOGGER.info(e)


def delete_scan_temp(workspace):
    """删除dep扫描软件包和已安装软件中间文件"""
    scan_temp_path = os.path.join(workspace, KitConfig.SCAN_TEMP)
    try:
        shutil.rmtree(scan_temp_path)
    except OSError or IOError as e:
        LOGGER.info(e)


def delete_pre_tmp(workspace):
    """删除源码迁移预处理中间文件"""
    pre_tmp_path = os.path.join(workspace, KitConfig.PRE_TEMP)
    try:
        shutil.rmtree(pre_tmp_path)
    except OSError or IOError as e:
        LOGGER.info(e)


def delete_c_make_tmp(user_id, task_name):
    """删除源码迁移Cmake中间文件"""
    dir_name = os.path.join(KitConfig.cmake_bin_dir,
                            '../Build_' + user_id + task_name)
    shutil.rmtree(dir_name, ignore_errors=True)
    output_file = os.path.join(KitConfig.cmake_bin_dir,
                               '../output/%s/rslt.json' % (
                                   user_id + task_name))
    shutil.rmtree(os.path.dirname(output_file), ignore_errors=True)


def _task_op_log(request, task_type, op_result):
    detail_msg = OP_DETAIL_MAP[op_result][task_type]
    if KitConfig.tool_name == KitConfig.DEPENDENCY:
        msg = (detail_msg, None)
    else:
        msg = (None, detail_msg)
    save_operation_log(request.user, OP_EVENT, op_result, msg)


def rm_tree(path):
    """删除64迁移预检中间文件"""
    count = COUNT
    while os.path.isdir(path) and count:
        shutil.rmtree(path, ignore_errors=True)
        count -= 1

```

