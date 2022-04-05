```python
#! /usr/bin/env python
# -*- coding: utf-8 -*-

"""
Copyright: Copyright (c) Huawei Technologies Co., Ltd. 2020-2020.
All rights reserved.
Create: 2020-11-04
Content: 运行日志压缩任务类及任务队列
"""

import os
import shutil
import threading
import multiprocessing
from datetime import datetime

from taskmanager.models import Task
from base_class.base_task import BaseTask
from base_class.base_queue import BaseTaskQueue
from resource.utils import insert_operationlog, LOGGER
from util import task_util
from .run_log_compress import RunLogCompress, RunLogCompressStatus
from tool.util.common_util import CommonUtil


_FINISH_PROGRESS = 100


class RunLogCompressTask(BaseTask):
    """
    运行日志压缩任务类
    """

    def __init__(self, user_id, task_name):
        scan_param = None
        BaseTask.__init__(self, user_id, scan_param, task_name,
                          task_util.PORT_RUN_LOG_DOWNLOAD)
        self.task_api = None

    def start(self, queue):
        """
        下发日志压缩任务
        :param queue: 任务队列
        :return:
        """
        self.task_api = RunLogCompress(self.task_name, self.user_id, queue)
        self.task_api.compress_run_log()

    def clear_temp_file(self):
        """
        清除临时文件
        :return:
        """
        install_path = CommonUtil.get_customize_path().get_tool_root_dir()
        download_temp_path = \
            os.path.join(install_path, RunLogCompress.DOWNLOAD_TEMP_DIRECTORY)
        self._clear_temp_file(download_temp_path)
        download_path = \
            os.path.join(install_path, RunLogCompress.DOWNLOAD_DIRECTORY)
        self._clear_temp_file(download_path)
        download_package = \
            os.path.join(install_path, RunLogCompress.FINAL_LOG_PACKAGE)
        self._clear_temp_file(download_package)

    @staticmethod
    def _clear_temp_file(filepath):
        """
        清除可能存在的临时文件
        :param filepath:
        :return:
        """
        try:
            if os.path.isdir(filepath):
                shutil.rmtree(filepath)
            elif os.path.isfile(filepath):
                os.remove(filepath)
        except (OSError, IOError) as err:
            LOGGER.error("Failed to clear temporary files: %s.", err)

    def add_task(self, task):
        """
        添加任务
        :param task:
        :return:
        """
        self._cond.acquire()
        if len(self._pending_tasks) + \
                len(self._running_tasks) >= self.MAX_TASKS:
            self._cond.release()
            return False
        self._pending_tasks[task.task_id] = task
        self._cond.notify()
        self._cond.release()
        return True
        
        
class RunLogCompressTaskQueue(BaseTaskQueue):
    """
    运行日志压缩任务队列
    """
    # 待处理的任务
    _pending_tasks = dict()
    # 正在进行的任务
    _running_tasks = dict()
    # 控制任务的条件变量
    _cond = threading.Condition()
    # 消息队列
    _queue = multiprocessing.Queue()
    # 进程控制列表
    _running_processes = dict()

    def _inner_run(self):
        """ 循环处理任务 """
        while True:
            task_id, task = None, None
            try:
                self._cond.acquire()
                if not self._pending_tasks:
                    # 如果没有待定的任务，则休眠等待
                    self._cond.wait()

                if self._stopped:
                    break

                task_id, task = self.get_pending_task()
                self._cond.release()

                # 开启进程下发任务
                process = multiprocessing.Process(
                    target=run_log_compress_process_handler,
                    args=(self._queue, task))
                process.daemon = True
                self._cond.acquire()
                self._running_processes[task_id] = {
                    'process': process, 'timeout': 0
                }
                self._cond.release()
                process.start()
                LOGGER.info("Create children process[%s] for "
                            "run log compress task[%s]"
                            % (process.pid, task.task_name))
            except Exception as base_ex:
                LOGGER.error(base_ex)
                self._handle_exception_task(task_id, task)

    def _handle_exception_task(self, task_id, task):
        """ 异常类处理 """
        if task_id and task:
            self._running_tasks.pop(task_id)
            insert_operationlog(
                task.user_id, 54, 1, datetime.now(), [107, ''])
            self._cond.acquire()
            if task_id in self._running_processes:
                self._running_processes.pop(task_id)
            self._cond.release()

    def _update_status(self):
        """
        循环更新扫描状态
        """
        while True:
            try:
                if self._stopped:
                    break
                scan_status = self._queue.get(True)
                task = Task.objects.get(
                    task_name=scan_status.get('task_name'),
                    user_name=scan_status.get('user_name'),
                    task_type=task_util.PORT_RUN_LOG_DOWNLOAD
                )
                if task.status != RunLogCompressStatus.RUNNING:
                    continue
                task.status = \
                    scan_status.get('status', RunLogCompressStatus.RUNNING)
                task.save()

                self._update_status_logger(task)
            except Exception as ex:
                LOGGER.error(ex)

    def _update_status_logger(self, task):
        """ 更新结束状态的日志 """
        if task.status in (RunLogCompressStatus.SUCCESS,
                           RunLogCompressStatus.PART_LOG_COMPRESSION):
            LOGGER.info('Compress run logs successful.')
            insert_operationlog(
                task.user_name, 54, 0, datetime.now(), [106, ''])
        elif task.status == RunLogCompressStatus.FAILED:
            LOGGER.error('Compress run log failed.')
            insert_operationlog(
                task.user_name, 54, 1, datetime.now(), [107, ''])
        else:
            return
        # 扫描结束将进度置位100，兼容其他模块逻辑
        task.progress = _FINISH_PROGRESS
        task.save()
        self.finish_task(task.task_name, task.user_name)

    @staticmethod
    def _abnormal_process_control(task):
        """ 异常进程处理
            1.将数据库扫描状态置成失败置成失败状态
            2.插入扫描失败日志
        """
        try:
            running_task = Task.objects.get(
                task_name=task.task_name,
                user_name=task.user_id,
                task_type=task_util.PORT_RUN_LOG_DOWNLOAD
            )
            if running_task.status == RunLogCompressStatus.RUNNING:
                running_task.status = RunLogCompressStatus.FAILED
                running_task.progress = _FINISH_PROGRESS
                running_task.save()
                LOGGER.error('Finish the run log download task failed.')
                insert_operationlog(
                    task.user_id, 54, 1, datetime.now(), [107, ''])
        except Task.DoesNotExist as error:
            LOGGER.info("Task [%s] dose not exist in db. Except:%s.",
                        task.task_name, error)
        
def run_log_compress_process_handler(queue, task):
    """
    日志压缩任务进程控制
    :param queue: 消息队列
    :param task: 日志压缩任务
    :return:
    """
    task.start(queue)


RunLogCompressTaskQueue()
```

