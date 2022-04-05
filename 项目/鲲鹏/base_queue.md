```python
"""
实现任务队列的基类
"""
import os
import time
import _thread
import threading
import multiprocessing

from tool.kit_config import KitConfig
from util import common_util


if KitConfig.tool_name == KitConfig.DEPENDENCY:
    from scanning.models import Task
    from user_manager.models import User, UserConfig
else:
    from taskmanager.models import Task
    from usermanager.models import User, UserConfig


LOGGER = common_util.get_logger()


class BaseTaskQueue:
    """
    任务队列, 单例
    """
    # 队列里的最大任务数
    MAX_TASKS = 1
    # 进程超时时间
    MAX_PROCESS_TIMEOUT = 5
    # 待处理的任务
    _pending_tasks = dict()
    # 正在进行的任务
    _running_tasks = dict()
    # 控制任务的条件变量
    _cond = threading.Condition()
    # 队列是否被停止
    _stopped = False
    # 单例
    _instance = None
    # 消息队列
    _queue = multiprocessing.Queue()
    # 进程控制列表: {'task_id': {'process': '', 'timeout': ''}}
    _running_processes = dict()

    def __new__(cls, *args, **kw):
        """ 实例化单例 """
        if cls._instance is None:
            cls._instance = object.__new__(cls, *args, **kw)
            cls._instance.run()
        return cls._instance

    def run(self):
        """
        启动队列
        :return:
        """
        _thread.start_new_thread(self._inner_run, ())
        _thread.start_new_thread(self._update_status, ())
        _thread.start_new_thread(self._processes_controller, ())

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

    def get_user_config(self, task):
        """ 读取数据库获取扫描参数 """
        user = User.objects.get(username=task.user_id)
        workspace_path = user.get_workspace()

        user_config_dict = {"user_id": task.user_id}
        user_config = UserConfig.objects.filter(
            user_id=user.get_id()).first()
        if not user_config:
            user_config_dict['c_line'] = 500
            user_config_dict['asm_line'] = 250
            user_config_dict['keep_going'] = True
            user_config_dict['p_month_flag'] = True
        else:
            user_config_dict['c_line'] = user_config.get_c_line()
            user_config_dict['asm_line'] = user_config.get_asm_line()
            user_config_dict['keep_going'] = \
                user_config.get_keep_going()
            user_config_dict['p_month_flag'] = \
                user_config.get_p_month_flag()

        return workspace_path, user_config_dict

    def get_running_task(self):
        """
        获得正在运行的任务
        :return:
        """
        try:
            if len(self._pending_tasks) >= 1:
                _, task = list(self._pending_tasks.items())[0]
                return True, task
            if len(self._running_tasks) >= 1:
                _, task = list(self._running_tasks.items())[0]
                return True, task
            return False, None
        except Exception as error:
            LOGGER.warning("Except:%s.", error)
            return False, None

    def get_pending_task(self):
        """
        获取等待队列中的任务
        :return:
        """
        task_id, task = list(self._pending_tasks.items())[0]
        self._pending_tasks.pop(task_id)
        self._running_tasks[task_id] = task
        return task_id, task

    def stop(self):
        """
        停止队列
        :return:
        """
        self._cond.acquire()
        self._stopped = True
        self._cond.notify()
        self._cond.release()

    def finish_task(self, task_name, user_id):
        """
        结束任务时将任务从运行列表弹出
        :param task_name:
        :param user_id:
        :return:
        """
        self._cond.acquire()
        for item in list(self._running_tasks.keys()):
            if self._running_tasks[item].task_name == task_name \
                    and self._running_tasks[item].user_id == user_id:
                if not self.exist_in_db(self._running_tasks[item]):
                    LOGGER.info("Task[%s] not in db" % task_name)
                    break
                task = self._running_tasks.pop(item)
                process_info = self._running_processes.pop(item)
                process = process_info.get('process')
                LOGGER.info("Task[%s] finished" % task.task_name)
                if process.is_alive():
                    pid = process.pid
                    LOGGER.info("Terminate process[%s]" % pid)
                    process.terminate()
                    os.waitpid(pid, 0)
                break
        self._cond.release()

    def _inner_run(self):
        """
        循环处理任务
        :return:
        """

    def _update_status(self):
        """
        循环更新状态
        :return:
        """

    def _processes_controller(self):
        """
        扫描进程控制
        :return:
        """
        while True:
            self._cond.acquire()
            if self._running_processes:
                self._running_processes_controller()
            self._cond.release()
            time.sleep(1)

    def _running_processes_controller(self):
        """
        扫描进程控制函数主体
        :return:
        """
        for task_id in list(self._running_processes):
            task = self._running_tasks.get(task_id)
            if not self.exist_in_db(task):
                LOGGER.error('The task not exists in database.')
                self._running_processes_handle(task_id)
                continue
            process = self._running_processes.get(task_id).get('process')
            if not process.is_alive():
                self._running_processes[task_id]['timeout'] += 1
                if self._running_processes[task_id]['timeout'] == \
                        self.MAX_PROCESS_TIMEOUT and \
                        not self._judge_task_progress(task):
                    LOGGER.error('The task process is not alive.')
                    self._running_processes_handle(task_id)

    def _running_processes_handle(self, task_id):
        """
        扫描进程处理
        :param task_id: 扫描任务类id
        :return:
        """
        process_info = self._running_processes.pop(task_id)
        process = process_info.get('process')
        self._check_process_alive(process)
        task = self._running_tasks.pop(task_id)
        self._clear_temp_file(task)
        self._abnormal_process_control(task)

    @staticmethod
    def _judge_task_progress(task):
        """
        判断扫描任务进度是否为100
        :param task: 扫描任务类
        :return:
        """
        try:
            if KitConfig.tool_name == KitConfig.DEPENDENCY:
                progress = Task.objects.get(
                    task_name=task.task_name,
                    user_id=task.user_id,
                    task_type=task.task_type).progress
            else:
                progress = Task.objects.get(
                    task_name=task.task_name,
                    user_name=task.user_id,
                    task_type=task.task_type).progress
        except Task.DoesNotExist as error:
            progress = 0
            LOGGER.info("Task [%s] does not exist in db. Except:%s.",
                        task.task_name, error)
        return progress == 100

    def _clear_temp_file(self, task):
        """
        判断扫描任务是否存在数据库并清空临时文件
        :param task: 扫描任务类
        :return:
        """
        if not self.exist_in_db(task):
            task.clear_temp_file()
            LOGGER.info("Clear temp file")

    @staticmethod
    def _check_process_alive(process):
        """
        查询进程是否存活并关闭该进程
        :param process: 进程类
        :return:
        """
        if process.is_alive():
            pid = process.pid
            process.kill()
            os.waitpid(pid, 0)
            LOGGER.info("Terminate process[%s]" % pid)

    @staticmethod
    def exist_in_db(task):
        """
        判断扫描任务是否存在于数据库
        :param task: 扫描任务类
        :return:
        """
        task_type = task.task_type
        if KitConfig.tool_name == KitConfig.DEPENDENCY:
            return Task.objects.filter(
                task_name=task.task_name, user_id=task.user_id,
                task_type=task_type)
        return Task.objects.filter(
            task_name=task.task_name, user_name=task.user_id,
            task_type=task_type)

    def _abnormal_process_control(self, *args):
        """
        异常进程处理
            1.将数据库扫描状态置成失败置成失败状态
            2.插入扫描失败日志
        :return:
        """
```

