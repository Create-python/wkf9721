```python
import os
import logging

from tool.util.common_util import CommonUtil
from tool.logger.logger import Logger

LOGGER = logging.getLogger(Logger.LOGGER_NAME)

STATUS_NORMAL = 0
STATUS_WORK_SPACE_ERR = 1
STATUS_DISC_SPACE_ERR = 2
STATUS_WORK_SPACE_WARNING = 3
STATUS_DISC_SPACE_WARNING = 4

_GATE_WORK_SPACE_ERR = 95
_GATE_WORK_SPACE_WARNING = 80
_GATE_DISC_SPACE_ERR = 95
_GATE_DISC_SPACE_WARNING = 80

_CONTENT_MAP = {
    STATUS_NORMAL: "[Info]:No warning.",
    STATUS_WORK_SPACE_ERR: "[Info]: The remaining workspace is insufficient."
                           "Delete historical analysis records "
                           "to release space.",
    STATUS_DISC_SPACE_ERR: "[Info]: The remaining drive space is insufficient."
                           "Release the drive space.",
    STATUS_WORK_SPACE_WARNING: "[Info]: Workspace is warning.Delete historical"
                               "analysis records to release space.",
    STATUS_DISC_SPACE_WARNING: "[Info]: Disc is warning. "
                               "Release the drive space."
}


def disk_usage_info():
    """返回某个路径下的磁盘使用情况
    读取磁盘使用情况
    total 读取磁盘总容量
    used  读取磁盘已用大小
    free  读取磁盘剩余空间
    percent 读取磁盘已用百分比
    """

    st = os.statvfs(os.getcwd())
    #  f_bavail:非超级用户可获取的块数   f_frsize: 分栈大小
    free = float(st.f_bavail * st.f_frsize)
    #  f_blocks: 文件系统数据块总数
    total = float(st.f_blocks * st.f_frsize)
    #  f_bfree: 可用块数
    used = float(st.f_blocks - st.f_bavail) * st.f_frsize
    total = total / 1024. / 1024. / 1024.
    used = used / 1024. / 1024. / 1024.
    free = free / 1024. / 1024. / 1024.
    try:
        percent = (used / total) * 100.
    except ZeroDivisionError:
        percent = 0
    return total, used, free, percent


def workspace_usageinfo(path):
    """    读取工作空间使用情况
    spacetotal 工作空间总容量 预设为100G
    memory  工作空间已用大小
    spacefree  工作空间剩余空间
    spacepercent 工作空间已用百分比"""

    total = 0
    for dir_path, _, filename_list in os.walk(path):
        for filename in filename_list:
            file_path = os.path.join(dir_path, filename)
            try:
                total += os.path.getsize(file_path)
            except OSError:
                LOGGER.info("No permission to use.")
    memory = float(total) / 1024. / 1024. / 1024.
    space_total = 100
    space_free = space_total - memory
    space_percent = memory / space_total * 100.
    return space_total, memory, space_free, space_percent


def query_status():
    """判断磁盘状态逻辑"""
    customize_path = CommonUtil.get_customize_path()
    install_path = customize_path.get_tool_root_dir()
    total, used, free, percent = disk_usage_info()
    space_total, memory, space_free, space_percent = workspace_usageinfo(
        install_path)
    if space_percent > _GATE_WORK_SPACE_ERR:
        return STATUS_WORK_SPACE_ERR
    elif _GATE_WORK_SPACE_WARNING <= space_percent <= _GATE_WORK_SPACE_ERR:
        if percent > _GATE_DISC_SPACE_ERR:
            return STATUS_DISC_SPACE_ERR
        else:
            return STATUS_WORK_SPACE_WARNING
    else:
        if percent > _GATE_DISC_SPACE_ERR:
            return STATUS_DISC_SPACE_ERR
        elif _GATE_DISC_SPACE_WARNING <= percent <= _GATE_DISC_SPACE_ERR:
            return STATUS_DISC_SPACE_WARNING
        else:
            return STATUS_NORMAL


def query_space():
    param = query_status()
    print(_CONTENT_MAP[param])

```

