```python
#!/usr/local/python3
# -*- encoding: utf-8 -*-
from datetime import datetime

from rest_framework.decorators import api_view
from rest_framework import status
from rest_framework.response import Response

from tool.kit_config import KitConfig
from tool.util import calculate_space
from tool.util.common_util import CommonUtil
if KitConfig.tool_name == KitConfig.DEPENDENCY:
    from log_manager import common
    PARAM_TO_DEP_CONTENT = {
        calculate_space.STATUS_WORK_SPACE_ERR: {
            "detail": "请删除历史分析记录，释放空间",
            "alarm_status": 1,
            "log_event": common.OpEvent.WORKSPACE_WARNING_RED,
            "log_mes": "Workspace reports an emergency!"
        },
        calculate_space.STATUS_DISC_SPACE_ERR: {
            'detail': "请释放磁盘空间",
            'alarm_status': 2,
            "log_event": common.OpEvent.DISCSPACE_WARNING_RED,
            "log_mes": "Disc reports an emergency!"
        },
        calculate_space.STATUS_WORK_SPACE_WARNING: {
            'detail': "请删除历史分析记录，释放空间",
            'alarm_status': 3,
            'log_event': common.OpEvent.WORKSPACE_WARNING_YEL,
            'log_mes': "Workspace is warning!"
        },
        calculate_space.STATUS_DISC_SPACE_WARNING: {
            'detail': "请释放磁盘空间",
            'alarm_status': 4,
            'log_event': common.OpEvent.DISCSPACE_WARNING_YEL,
            'log_mes': "Disc is warning!"
        },
        calculate_space.STATUS_NORMAL: {
            'detail': "无报警",
            'alarm_status': 0,
            'log_event': common.OpEvent.SPACE_NO_WARNING,
            'log_mes': "No warning."
        }
    }
else:
    from resource.utils import insert_operationlog
    PARAM_TO_PORT_CONTENT = {
        calculate_space.STATUS_NORMAL: {
            'detail': '无报警',
            'alarm_status': 0,
            'event': 27,
            'message': [46, '']
        },
        calculate_space.STATUS_WORK_SPACE_ERR: {
            'detail': "请删除历史分析记录，释放空间",
            'alarm_status': 1,
            'event': 28,
            'message': [47, '']
        },
        calculate_space.STATUS_DISC_SPACE_ERR: {
            'detail': '请释放磁盘空间',
            'alarm_status': 2,
            'event': 29,
            'message': [48, '']
        },
        calculate_space.STATUS_WORK_SPACE_WARNING: {
            'detail': '请删除历史分析记录，释放空间',
            'alarm_status': 3,
            'event': 30,
            'message': [49, '']
        },
        calculate_space.STATUS_DISC_SPACE_WARNING: {
            'detail': '请释放磁盘空间',
            'alarm_status': 4,
            'event': 31,
            'message': [50, '']
        }
    }


@api_view(["GET"])
def get_query(request):
    """提供接口 返回使用情况"""
    customize_path = CommonUtil.get_customize_path()
    install_path = customize_path.get_tool_root_dir()
    total, used, free, percent = calculate_space.disk_usage_info()
    space_total, memory, space_free, space_percent = \
        calculate_space.workspace_usageinfo(install_path)
    if not all([total, used, free, percent, space_total, memory, space_free,
                space_percent]):
        return Response({'code': 0}, status=status.HTTP_404_NOT_FOUND)
    content = {
        "partitionTotal": total,
        "softNeeded": space_total,
        "detail": "",
        "softPercent": space_percent,
        "softRemain": space_free,
        "partitionPercent": percent,
        "partRemain": free,
        "alarm_status": 0,
        "suggest_remain": "建议剩余工作空间>20G",
    }
    param = calculate_space.query_status()
    if KitConfig.tool_name == KitConfig.DEPENDENCY:
        user_id = request.user.get_id()
        """
        alarm_status=0是无报警
        alarm_status=1是工作空间红色报警
        alarm_status=2是磁盘分区红色报警
        alarm_status=3是工作空间黄色报警
        alarm_status=4是磁盘空间黄色报警
        """
        content["detail"] = PARAM_TO_DEP_CONTENT[param].get("detail")
        content["alarm_status"] = \
            PARAM_TO_DEP_CONTENT[param].get("alarm_status")
        log_event = PARAM_TO_DEP_CONTENT[param].get("log_event")
        log_mes = PARAM_TO_DEP_CONTENT[param].get("log_mes")
        log_result = common.OpResult.SUCCESS
        common.save_operation_log(user_id, log_event, log_result, log_mes)
        return Response(content, status=status.HTTP_200_OK)

    else:
        username = request.user
        content['detail'] = PARAM_TO_PORT_CONTENT[param].get('detail')
        content['alarm_status'] = PARAM_TO_PORT_CONTENT[param].get(
            'alarm_status')
        event = PARAM_TO_PORT_CONTENT[param].get('event')
        detail = PARAM_TO_PORT_CONTENT[param].get('message')
        insert_operationlog(username, event, 0, datetime.now(), detail)
        return Response(content, status=status.HTTP_200_OK)

```

