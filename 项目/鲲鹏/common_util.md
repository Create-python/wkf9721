```python
"""
模块说明：通用工具处理
版权所有：华为技术有限公司 版本所有(C)2019~2020
修改信息：2020-5-14  新建文件
"""
import logging
import re
import os
from rest_framework.response import Response
from rest_framework import status

from util.enum_constants import RESULTS
from util.first_login import FIRST_LOGIN_INFO
from util.response import generate_response_body
from tool.kit_config import KitConfig
from tool.logger.logger import Logger

if KitConfig.tool_name == KitConfig.DEPENDENCY:
    from user_manager.models import User
else:
    from usermanager.models import User

# 响应码
STATUS_SUCCESS = 0
STATUS_FAILED = 1

TAR_ZIP_TYPE = ("xz", "bz", "gz", "bz2")
TAR_ZIP_PATTERN = r"(.+)(\.tar(?:$|\.(?:%s)$))" % "|".join(TAR_ZIP_TYPE)


def get_logger():
    return logging.getLogger(Logger.LOGGER_NAME)


LOGGER = get_logger()


def if_user_login(username):
    """
    :param username: 用户名
    :return:   json格式响应体
    """
    try:
        user = User.objects.get(username=username)
    except User.DoesNotExist as error:
        status_code = RESULTS.FAILED.get_code()
        info = "The user does not exist."
        LOGGER.warning(info + " Except:%s.", error)
        info_chinese = u"用户不存在。"
        response_body = generate_response_body(
            status_code, info, info_chinese)
        return response_body

    if not user.login_status:
        status_code = RESULTS.FAILED.get_code()
        info = "The user does not login."
        info_chinese = u"用户未登陆。"
        response_body = generate_response_body(
            status_code, info, info_chinese)
        return response_body
    if user.is_firstlogin:
        status_code = RESULTS.FAILED.get_code()
        info = FIRST_LOGIN_INFO
        info_chinese = u"用户为首次登陆，请先修改密码。"
        response_body = generate_response_body(
            status_code, info, info_chinese)
        return response_body
    status_code = RESULTS.SUCCESSFUL.get_code()
    info = ""
    info_chinese = ""
    response_body = generate_response_body(
        status_code, info, info_chinese)
    return response_body


def is_admin_user(username):
    """
    功能描述：判断该用户是否为管理员用户
    参数：用户名
    返回值：响应体
    异常描述：
    修改记录：首次开发
    """
    try:
        user = User.objects.get(username=username)
    except User.DoesNotExist as error:
        status_code = RESULTS.FAILED.get_code()
        info = "The user does not exist."
        info_chinese = u"用户不存在。"
        LOGGER.warning(info + " Except:%s.", error)
        response_body = generate_response_body(
            status_code, info, info_chinese)
        return response_body
    # 工具目前只有管理员用户和普通用户，Dep&Port的普通用户都为User
    if user.get_role() == 'User':
        status_code = RESULTS.FAILED.get_code()
        info = "User does not have permission."
        info_chinese = u"用户权限不足。"
        response_body = generate_response_body(
            status_code, info, info_chinese)
        return response_body
    status_code = RESULTS.SUCCESSFUL.get_code()
    info = ""
    info_chinese = ""
    response_body = generate_response_body(
        status_code, info, info_chinese)
    return response_body


def check_if_user_login():
    """
    功能描述：自定义用户是否登录检查装饰器
    参数：用户名
    返回值：响应体
    异常描述：
    修改记录：首次开发
    """
    def wrapper(func):
        def decorated(request, *args, **kwargs):
            response_body = if_user_login(request.user)
            if response_body['status'] == STATUS_FAILED:
                return Response(response_body,
                                status=status.HTTP_401_UNAUTHORIZED)
            return func(request, *args, **kwargs)
        return decorated
    return wrapper


def check_is_admin_user():
    """
    功能描述：管理员用户是否登录检查装饰器
    参数：用户名
    返回值：响应体
    异常描述：
    修改记录：首次开发
    """
    def wrapper(func):
        def decorated(request, *args, **kwargs):
            response_body = is_admin_user(request.user)
            if response_body['status'] == STATUS_FAILED:
                return Response(response_body,
                                status=status.HTTP_401_UNAUTHORIZED)
            return func(request, *args, **kwargs)
        return decorated
    return wrapper


def is_x86_arch():
    return "x86" in os.uname().machine.lower()


def check_path_under_workspace(workspace, path):
    """
    :param workspace: 工作路径
    :param path: 判断的路径
    :return: 路径是否在工作路径下, 响应信息

    rule：1.必须以workspace开头  2.必须以workspace结尾或者后跟 /+任意字符
    example：
            workspace = /opt/portadv/portadmin
            path = /opt/portadv/portadmin       return True
            path = /opt/portadv/portadmin/      return True
            path = /opt/portadv/portadmin/xxxx  return True
            path = /opt/portadv/portadminxxxxx  return False
            path = /home/opt/portadv/portadmin  return False

    """
    real_path = os.path.realpath(path)
    pattern = "^%s($|/.*)" % os.path.realpath(workspace)  # 去掉workspace最后的/
    is_valid_path = re.match(pattern, real_path)
    if is_valid_path:
        info, info_chinese = "", ""
    else:
        info = "The specified path is not in the user workspace."
        info_chinese = u"指定路径错误，未在用户工作空间下"
        LOGGER.error(
            "Cannot access the file or directory: {}".format(path))
    return (is_valid_path, (info, info_chinese))


def trans_path_to_abspath(paths):
    if not isinstance(paths, str):
        return ""
    return ",".join(
        (os.path.realpath(x) for x in paths.strip().split(",") if x)
    )


def is_name_contain_special_character(name):
    """
    校验文件或目录名是否包含特殊字符
    :param name: 文件或目录名
    :return: bool, response
    """
    special_chars = (' ', '^', '`', '|', ';',
                     '&', '$', '>', '<', '\\', '!')
    if any(char in name for char in special_chars):
        info = "The file or folder name cannot contain spaces or special " \
               "characters such as ^ ` / | ; & $ > < \ !. Modify the " \
               "file or folder name and try again."
        info_chinese = r"文件或文件夹名中不能包含空格以及^ ` / | ; & $ > < \ ! " \
                       r"等特殊字符，请修改后重试。"
        return True, info, info_chinese
    return False, '', ''


# 密码过期时间映射
map_pw_expired = {
    0: ("today", "将于今天"),
    1: ("one day", "还有一天"),
    2: ("two days", "还有两天"),
    3: ("three days", "还有三天"),
    4: ("four days", "还有四天"),
    5: ("five days", "还有五天"),
    6: ("six days", "还有六天"),
    7: ("seven days", "还有七天")
}


def get_prefix_suffix(file_name):
    """获取文件的前缀和后缀"""
    match = re.search(TAR_ZIP_PATTERN, file_name)
    if match:
        return match.group(1), match.group(2)
    return os.path.splitext(file_name)


def check_name(file_name):
    if file_name.startswith("."):
        return True
    file_name, _ = get_prefix_suffix(file_name)
    check = re.compile(r'[\w\.\-\(\)\+]{1,64}')
    return check.search(file_name).group() != file_name
```

