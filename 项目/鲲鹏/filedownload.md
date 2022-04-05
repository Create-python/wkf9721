```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

"""
Copyright: Copyright (c) Huawei Technologies Co., Ltd. 2020-2020. All rights reserved.
Create: 2020-11-09
Content: filecontent
"""
import os

from django.http import HttpResponse, \
    HttpResponseNotFound, HttpResponseBadRequest, FileResponse
from wsgiref.util import FileWrapper
from rest_framework import status
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import AllowAny
from rest_framework.response import Response

from jwt_aut.authentication import JSONWebAuthentication
from util.common_util import is_admin_user, if_user_login
from util.verify_view import validate_data_type
from resource import utils
from tool.util.common_util import CommonUtil


LOGGER = utils.get_logger()
# 请求状态码
STATUS_SUCCESS = 0
STATUS_FAILED = 1
_ROOT_PATH = CommonUtil.get_customize_path().get_tool_root_dir()

# 获取文件auth_key路径映射，key为能下载的路径，value为是否需要在前边增加用户名参数
DOWNLOAD_PATH_MAP = {
    "report/packagerebuild": True,
    "migration": True,
    "sos": True,
    "jars": True,
    "downloadlog": False,
}


@api_view(['POST'])
@permission_classes([AllowAny, ])
def download(request):
    """
    文件下载
    支持将jwt放到headers和body中，body支持application/x-www-form-urlencoded
    和application/json格式
    :param request: 请求
    """
    sub_path = request.POST.get("sub_path")
    file_path = request.POST.get("file_path")
    if not sub_path or not file_path:
        # 兼容IDE，body使用json格式
        sub_path = request.data.get("sub_path")
        file_path = request.data.get("file_path")
    LOGGER.debug("file_path: %s, sub_path: %s", file_path, sub_path)
    file_path_check_is_failed = validate_data_type(file_path)
    if not sub_path or not file_path or file_path_check_is_failed:
        return HttpResponseBadRequest("The input parameter is incorrect.")
    jwt = request.POST.get("jwt")
    if jwt:
        user, _ = JSONWebAuthentication().jwt_auth(jwt)
    else:
        user, _ = JSONWebAuthentication().authenticate(request)
    response_body = if_user_login(user)
    if response_body['status'] == STATUS_FAILED:
        return Response(response_body,
                        status=status.HTTP_401_UNAUTHORIZED)
    user_home = DOWNLOAD_PATH_MAP.get(sub_path, None)
    if user_home is None:
        LOGGER.info("Path not found. sub_path: %s.", sub_path)
        return HttpResponseNotFound("File not found")
    username = ""
    if user_home:
        username = user.username
    else:
        # 不在用户空间下限制只有管理员才能下载
        response_body = is_admin_user(user)
        if response_body['status'] == STATUS_FAILED:
            return Response(response_body,
                            status=status.HTTP_401_UNAUTHORIZED)
    path = os.path.join(_ROOT_PATH, username, sub_path)
    full_path = os.path.join(path, file_path)
    real_path = os.path.realpath(full_path)
    # 路径校验，防止文件名有..导致下载任意文件
    if not real_path.startswith(path) or not os.path.exists(real_path):
        LOGGER.info("Path not found. real_path: %s", real_path)
        return HttpResponseNotFound("File not found")
    relative_path = os.path.join(username, sub_path, file_path)
    new_uri = "/protected/%s" % relative_path
    filename = file_path.split('/')[-1]
    response = HttpResponse()
    response['Content-Type'] = 'application/octet-stream'
    response['Content-Disposition'] = 'attachment;filename="%s"' % filename
    response['X-Accel-Redirect'] = new_uri  # 使用NGINX重定向到对应的url下
    return response


def common_download(file_path):
    """
    普通文件下载
    :param file_path: 文件路径
    :return: 文件是否下载成功, response
    """
    try:
        wrapper = FileWrapper(open(file_path, "rb"))
        response = FileResponse(wrapper)
        response['Content-Type'] = '*/*'
        response['Content-Length'] = os.path.getsize(file_path)
        response[
            'Content-Disposition'] = 'attachment;filename="%s"' % file_path
        LOGGER.info("Success to download the file.")
        return True, response
    except FileNotFoundError as not_found:
        LOGGER.info("Download file failed. Except:%s.", not_found)
        return False, None
```

