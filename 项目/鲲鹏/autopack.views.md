```python
"""
自动打包接口文档
"""
import os
import re
import shutil
import subprocess
import time
from datetime import datetime
from functools import cmp_to_key

from wsgiref.util import FileWrapper
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework import status
from django.http import FileResponse
from django.db.models import Q
from tool.kit_config import KitConfig

from report_process.report_service import _check_his_report_nums
from resource import utils
from upload_tool.upload_control import upload_file_process
from usermanager.models import User
from taskmanager.models import Task
from autopack.autopack_task import TaskPackage, TaskQueuePackage
from autopack.rpm_list import dict_from_path
from util.check_space import check_space
from util.response import SuccessResponse, FailResponse, CommonResponse, \
    generate_response_body
from util import common_util, task_util

from tool.util.common_util import CommonUtil
from tool.exception.binary_scan_exception import OsTypesDifferException
from tool.refactoring.os_info import CurrentOsInfoGetUtil
from tool.kit_config import OsConfig
from upload_tool import file_upload
from tool.refactoring.refactoring import Refactor

LOGGER = utils.get_logger()
# 请求状态码
START_SUCCESS = 0
START_FAILED = 1
START_WITH_CENTOS = 3
START_WITHOUT_CENTOS = 4
STATUS_SUCCESS = 0
STATUS_FAILED = 1
REFACTORING_PACKAGE_TYPE = (".rpm", ".deb")
# ""兼容老接口， 0: 首次下浮， 1： 用户二次下发
FLAG_TYPES = ("", 0, 1)
# 包的架构有两种
ARCHITECTURE_TYPE = ('aarch64', 'noarch')

AUTO_PACK_TYPE = {
    '3': 'packagerebuild',
    '4': 'data',

}

CHOICE = {
    'OVERRIDE': 'override',
    'NORMAL': 'normal',
    'SAVE_AS': 'save_as',
}
_ROOT_PATH = CommonUtil.get_customize_path().get_tool_root_dir()


def autopack_check(flag, is_download, package_path):
    """
    重打包参数检查
    :param flag: 是否匹配鲲鹏yum仓库
    :param is_download: 是否下载
    :param package_path: 软件包路径
    :return: 响应，是否成功
    """
    info = 'The input parameter is incorrect.'
    info_chinese = "输入参数错误。"
    fail_resp = FailResponse(info, info_chinese)
    if not isinstance(is_download, bool):
        return fail_resp, False
    if flag not in FLAG_TYPES:
        return fail_resp, False
    result = get_package_list(package_path)
    his_report_num_status = _check_his_report_nums(len(result))
    # 达到历史报告最大值，禁止操作
    if his_report_num_status.get_code() == 3:
        info = his_report_num_status.get_msg()
        info_chinese = his_report_num_status.get_cn_msg()
        return FailResponse(info, info_chinese), False
    return None, True


@api_view(['POST'])
@common_util.check_if_user_login()
def start_autopack(request):
    """
    下发打包任务的restful接口
    """
    response = check_space()
    if response:
        return Response(response, status.HTTP_200_OK)
    request.user.update_op_time()
    data = {}
    username = str(request.user)
    LOGGER.debug('Start a re_package task')
    filepath = request.data.get('filepath', '')  # 需要校验
    flag = request.data.get('flag', '')
    is_download = request.data.get('is_download', True)
    package_path = get_package_path(request)
    response, ok = autopack_check(flag, is_download, package_path)
    if not ok:
        return response
    Refactor.is_download = is_download
    status_code, info, info_chinese, package_type \
        = package_path_check(filepath, username)
    if status_code == START_FAILED:
        utils.insert_operationlog(username, 10, 1, datetime.now(), [19, ''])
        return FailResponse(info, info_chinese, data)
    if package_type == 'rpm' and flag == FLAG_TYPES[1]:
        response_body = check_if_package_in_list(filepath)
        if response_body:
            return Response(response_body, status.HTTP_200_OK)
    response, result = check_file_access(filepath, data)
    if not result:
        return response
    task_name = datetime.now().strftime('%Y%m%d%H%M%S')
    task_info = [filepath, package_type]
    status_code = START_FAILED
    try:
        create_success, info, info_chinese = create_autopack_task(
            username, task_info, task_name)
        if create_success:
            data = {"task_id": task_name}
            status_code = STATUS_SUCCESS
    except (IOError, OSError, ValueError, TypeError) as ex:
        LOGGER.error('Except: %s', ex)
        utils.insert_operationlog(username, 10, 1, datetime.now(), [19, ''])
    response_body = generate_response_body(
        status_code, info, info_chinese, data)
    LOGGER.debug('The task_list POST end!')
    return Response(response_body, status.HTTP_200_OK)


def check_file_access(filepath, data):
    ret = os.access(filepath, os.R_OK)
    if not ret:
        info = 'Failed to create the task. The current porting user ' \
               'does not have the read permission on the file. ' \
               'Modify the permission and try again.'
        info_chinese = u"创建任务失败, 当前porting用户没有该文件的读取权限, " \
                       u"请修改权限后重试。"
        status_code = START_FAILED
        response_body = generate_response_body(
            status_code, info, info_chinese, data)
        return Response(response_body, status.HTTP_200_OK), False
    return None, True


def check_if_package_in_list(filepath):
    """
    @param filepath: 路径
    @return: 响应体
    """
    response_body = {}
    data = {}
    package_name = os.path.basename(filepath)
    package_name_list = package_name.split('.')
    # 替换架构 AARCH64
    package_name_aarch64_list = package_name_list[:]
    package_name_aarch64_list[-2] = ARCHITECTURE_TYPE[0]
    package_name_aarch64 = '.'.join(package_name_aarch64_list)
    # 替换架构 noaarch
    package_name_noaarch_list = package_name_list[:]
    package_name_noaarch_list[-2] = ARCHITECTURE_TYPE[1]
    package_name_noaarch = '.'.join(package_name_noaarch_list)
    # centos 7.6
    os_name, os_version = CurrentOsInfoGetUtil.get_rpm_system()
    os_info = os_name + os_version
    rpm_dict = dict_from_path()
    if package_name_aarch64 in rpm_dict.keys():
        data[package_name_aarch64] = rpm_dict[package_name_aarch64]
    elif package_name_noaarch in rpm_dict.keys():
        data[package_name_noaarch] = rpm_dict[package_name_noaarch]
    if data:
        info = 'The RPM package exists on the Kunpeng mirror source. ' \
               'You can download and install it.'
        info_chinese = "rpm包在鲲鹏镜像源上存在，可以下载安装后使用。"
        status_code = START_WITHOUT_CENTOS
        response_body = generate_response_body(
            status_code, info, info_chinese, data)
    if data and os_info == 'centos7.6':
        info = 'The RPM package exists on the Kunpeng mirror source. ' \
               'You can configure the yum source and install it. '
        info_chinese = "rpm包在鲲鹏镜像源上存在，可以配置yum源安装后使用。"
        status_code = START_WITH_CENTOS
        response_body = generate_response_body(
            status_code, info, info_chinese, data)
    return response_body


def create_autopack_task(username, task_info, task_name):
    """
    创建autopack任务，失败返回False，成功返回True
    :param username:
    :param task_info:
    :param task_name:
    :return:
    """
    # 判断当前用户是否有重打包任务
    running_task = Task.objects.filter(
        Q(user_name=username) & Q(task_type__in=[1, 2]) & Q(status=1))
    if running_task.exists():
        utils.insert_operationlog(
            username, 10, 1, datetime.now(), [19, ''])
        LOGGER.info('Other tasks are running.')
        info = "The task is running."
        info_chinese = "任务正在运行。"
        return False, info, info_chinese
    taskrun = TaskPackage(username, task_info, task_name)
    if TaskQueuePackage().add_task(taskrun):
        LOGGER.debug('Create task is successful.')
        init_autopack_task(task_name, username)
        info = 'Task created successfully.'
        info_chinese = u"创建任务成功。"
        return True, info, info_chinese
    else:
        utils.insert_operationlog(
            username, 10, 1, datetime.now(), [19, ''])
        LOGGER.error('Failed to creat task.')
        info = 'Failed to create the task.'
        info_chinese = u"创建任务失败。"
        return False, info, info_chinese


def init_autopack_task(task_name, user_name):
    """ 初始化重打包任务 """
    task = Task(
        task_type=1,
        task_name=task_name,
        user_name=user_name,
        time_created=datetime.now(),
        status=1,
        progress=0,
        source_dir="",
        compiler_type="",
        compiler_version="",
        construct_tool="",
        compile_command="",
        target_os="",
        target_kernel=""
    )
    task.save()


@api_view(['GET'])
@common_util.check_if_user_login()
def autopack_result(request, file_name=None):
    """
    自动打包结果下载
    @param request: 请求
    @param file_name:请求文件
    @return:
    """
    request.user.update_op_time()

    if not file_name:
        file_list = []
        for ext_type in REFACTORING_PACKAGE_TYPE:
            file_list.extend(get_file_list_of_autopack(request.user, ext_type))
        data = {'totalcount': len(file_list), 'filelist': file_list}

        info = "Autopack result query succeeded."
        info_chinese = u"查询重打包文件成功。"

        # 构建响应体数据
        return SuccessResponse(info, info_chinese, data)

    _, ext = os.path.splitext(file_name)
    if re.search(r".*?\.so\.?.*", file_name):
        ext = '.so'
    package_path = os.path.join(
        request.user.workspace, "%ss" % ext.strip("."))
    file = os.path.join(package_path, file_name)

    if not os.path.isfile(file):
        utils.insert_operationlog(
            request.user.username, 26, 1, datetime.now(), [55, ""])
        info = "File not found"
        info_chinese = u"文件不存在"
        return FailResponse(info, info_chinese)

    utils.insert_operationlog(
        request.user.username, 26, 0, datetime.now(), [54, ""])
    wrapper = FileWrapper(open(file, "rb"))
    response = FileResponse(wrapper)
    response['Content-Type'] = 'application/octet-stream'
    response['Content-Length'] = os.path.getsize(file)
    response[
        'Content-Disposition'] = 'attachment;filename="%s"' % file_name
    return response


@api_view(['POST'])
@common_util.check_if_user_login()
def check_upload(request):
    """
    检查文件是否能够上传
    :param request:
    :return: 能否上传文件
    """
    response = check_space()
    if response:
        return Response(response, status.HTTP_200_OK)
    user_name = str(request.user)
    file_name = request.data.get("file_name")
    file_size = request.data.get("file_size")
    scan_type = request.data.get("scan_type")
    if not scan_type or scan_type not in AUTO_PACK_TYPE.keys():
        err_code = STATUS_FAILED
        info = "Input parameter error."
        ch_info = "输入参数错误。"
        data = {}
        return CommonResponse(err_code, info, ch_info, data)
    if common_util.is_name_contain_special_character(file_name)[0]:
        info = "The file or folder name cannot contain spaces or special " \
               "characters such as ^ ` / | ; & $ > < \ !"
        ch_info = r"文件或文件夹名中不能包含空格以及^ ` / | ; & $ > < \ ! " \
                  r"等特殊字符"
        return FailResponse(info, ch_info)
    choice = request.data.get("choice")
    if choice == CHOICE['SAVE_AS']:
        invaild = common_util.check_name(file_name)
        if invaild:
            err_code = STATUS_FAILED
            info = "Name invalid"
            ch_info = "无效名字"
            data = {}
            return CommonResponse(err_code, info, ch_info, data)
    file_path = \
        os.path.join(KitConfig.tool_root_dir, user_name,
                     AUTO_PACK_TYPE[scan_type])
    data_path_file = False
    if scan_type == '4':
        data_path_file = True
        customize_path = CommonUtil.get_customize_path()
        data_path = customize_path.get_data_path(request.user.username)
        file_path = data_path
    err_code, info, ch_info, data = file_upload.UploadFile(
        request).build_file_upload_check(file_name, file_size, file_path,
                                         choice, data_path_file,
                                         need_unzip='false')
    LOGGER.info(info)
    if err_code:
        utils.insert_operationlog(user_name, 22, err_code, datetime.now(),
                                  [46, ""])
    return CommonResponse(err_code, info, ch_info, data)


@api_view(['POST'])
@common_util.check_if_user_login()
def upload_package(request):
    """
    上传包
    @param request:请求
    @return: 成功或失败响应
    """
    user_name = request.user
    response = check_space()
    if response:
        return Response(response, status.HTTP_200_OK)
    scan_type = request.META.get("HTTP_SCAN_TYPE")
    if not scan_type or scan_type not in AUTO_PACK_TYPE.keys():
        err_code = STATUS_FAILED
        info = "Input parameter error."
        ch_info = "输入参数错误。"
        data = {}
        return CommonResponse(err_code, info, ch_info, data)

    file_path = \
        os.path.join(KitConfig.tool_root_dir, str(user_name),
                     AUTO_PACK_TYPE[scan_type])

    return autopack_upload(request, file_path, [])


@api_view(['POST'])
@common_util.check_if_user_login()
def upload_data(request):
    """
    上传依赖
    @param request:请求
    @return: 成功或失败结果
    """
    response = check_space()
    if response:
        return Response(response, status.HTTP_200_OK)
    # data路径暂时先写死，安装路径可配置功能实现后修改
    customize_path = CommonUtil.get_customize_path()
    data_root = customize_path.get_data_path(request.user.username)
    if not os.path.isdir(data_root):
        os.mkdir(data_root)
    file_path = data_root
    return autopack_upload(request, file_path, [])


@api_view(['DELETE'])
@common_util.check_if_user_login()
def abort_task(request, task_id):
    """
    退出任务
    @param request: 请求
    @param task_id: 任务ID
    @return: 任务停止结果
    """
    return task_util.stop(request, task_id, task_util.PORT_AUTOPACK)


def autopack_upload(request, file_path,
                    type_limit=REFACTORING_PACKAGE_TYPE):
    """
    重打包上传函数
    @param request:请求
    @param file_path:文件路径
    @param type_limit: 上传限制
    @return: 请求
    """
    file_field = "file"
    file_name, upload_result = upload_file_process(file_field, file_path,
                                                   request, list(type_limit))

    return auto_Response(upload_result, file_name, file_path, request)


def auto_Response(upload_result, file_name, file_path, request):
    """
    返回Response内容
    :param file_name:文件名
    :param path:文件路径
    :param upload_result: 返回的内容
    :return: Response
    """
    code = upload_result[0]
    info = upload_result[1]
    info_chinese = upload_result[2]
    data = file_name
    op_log(code, file_path, request)
    return CommonResponse(code, info, info_chinese, data)


def op_log(code, path, request):
    """
    日志记录
    @param code: 操作日志事件
    @param path: 用户路径
    @param request:请求
    @return:
    """
    customize_path = CommonUtil.get_customize_path()
    data_root = customize_path.get_data_path(request.user.username)
    time = datetime.now()
    if code:
        if path != data_root:
            LOGGER.info("Failed to upload the refactoring package.")
            log_detail_code = 57
        else:
            LOGGER.info("Failed to upload the refactoring data.")
            log_detail_code = 59
        log_result_code = 1
        utils.insert_operationlog(request.user.username, 22, log_result_code,
                                  time, [log_detail_code, ""])
    else:
        if path != data_root:
            file_path = path
            LOGGER.info("The refactoring package has been uploaded "
                        "successfully. File uploaded to:%s", file_path)
            log_detail_code = 56
        else:
            file_path = data_root
            LOGGER.info("The refactoring data has been uploaded "
                        "successfully. File uploaded to:%s", file_path)
            log_detail_code = 58
        log_result_code = 0
        utils.insert_operationlog(request.user.username, 22, log_result_code,
                                  time, [log_detail_code, file_path])


def get_file_list_of_autopack(user, ext_type):
    """
    获取依赖信息
    @param user:用户
    @param ext_type:包扩展名
    @return:
    """

    def file_condition(file):
        _, ext = os.path.splitext(file)
        return ext.strip(".") == ext_type

    path = os.path.join(user.workspace, "%ss" % ext_type)
    if not os.path.isdir(path):
        return []
    return [f for f in os.listdir(path) if file_condition(f)]


def package_path_check(filepath, user_name):
    """
    检查传输的软件包路径正确性
    """
    user = User.objects.get(username=user_name)
    work_space = os.path.join(user.workspace, task_util.PACKAGE_REBUILD)
    package_type = ""
    if not isinstance(filepath, str):
        status_code = START_FAILED
        LOGGER.error("Incorrect parameter type.")
        info = 'Incorrect parameter type.'
        info_chinese = u'参数类型错误。'
        return status_code, info, info_chinese, package_type
    if os.path.exists(filepath):
        abspath = os.path.realpath(filepath)
        # msg为路径不合法时的失败响应体信息
        result, msg = common_util. \
            check_path_under_workspace(work_space, abspath)
        if result:
            ret, info, info_chinese = \
                common_util.is_name_contain_special_character(
                    abspath[abspath.find(work_space) + len(work_space):])
            if ret:
                return STATUS_FAILED, info, info_chinese, package_type
            temp_type = abspath.split('.')
            package_type = temp_type[-1]
            # 当前不支持deb格式的软件包处理，后续支持时补充判断条件
            if package_type in ['rpm', 'deb']:
                # 校验当前os是否支持package_type类型软件包构建
                OsConfig.target_os_name = \
                    CurrentOsInfoGetUtil.get_current_osinfo(package_type)[0]
                try:
                    CommonUtil.check_file_os_type(filepath)
                except OsTypesDifferException as err:
                    LOGGER.error('Except: %s', err)
                    status_code = START_FAILED
                    info = "The current OS does not support" \
                           " this software package type. "
                    info_chinese = u"当前操作系统不支持此类型软件包。"
                    return status_code, info, info_chinese, package_type
                status_code, info, info_chinese = pack_dep_tools(package_type)
            else:
                status_code = START_FAILED
                LOGGER.error("The file type is not supported.")
                info = "The file type is not supported."
                info_chinese = u"输入的文件类型不支持。"
        else:
            status_code = START_FAILED
            info, info_chinese = msg
    else:
        status_code = START_FAILED
        LOGGER.error("Failed to find the path or file.")
        info = "Failed to find the path or file."
        info_chinese = u"未找到路径或文件。"
    return status_code, info, info_chinese, package_type


def pack_dep_tools(package_type):
    """
    检查依赖包的安装情况
    """
    rpm_dep = ["rpmrebuild", "rpmbuild", "rpm2cpio"]
    deb_dep = ["ar", "dpkg-deb"]

    need_list = []

    if package_type == "rpm":
        dep_list = rpm_dep
    else:
        dep_list = deb_dep

    for tool in dep_list:
        received = subprocess.call(["which", tool], stdout=subprocess.DEVNULL)
        if received != 0:
            need_list.append(tool)

    if not need_list:
        status_code = START_SUCCESS
        info = ""
        info_chinese = ""
    else:
        status_code = START_FAILED
        info = "Please download and install dependency tool: " + " ".join(
            need_list)
        info_chinese = "以下依赖组件未安装，请自行下载安装：" + " ".join(need_list)
        LOGGER.error(info)

    return status_code, info, info_chinese


def get_package_path(request):
    """
    获取重打包结果路径
    :param request: 请求
    :return: 路径
    """
    path = os.path.join(_ROOT_PATH, request.user.username, "report",
                        "packagerebuild")
    if not os.path.exists(path):
        os.makedirs(path, 0o700)
    return path


def rpm_list_sort(pkg_info1, pkg_info2):
    """
    软件包列表排序函数，按照name升序，时间降序
    :param pkg_info1: 包信息1
    :param pkg_info2: 包信息2
    :return:
    """
    if pkg_info1["name"] > pkg_info2["name"]:
        return 1
    if pkg_info1["name"] < pkg_info2["name"]:
        return -1
    if pkg_info1["create_time"] > pkg_info2["create_time"]:
        return -1
    if pkg_info1["create_time"] < pkg_info2["create_time"]:
        return 1
    return 0


def get_package_list(path):
    """
    获取软件包结果列表
    :param path: 结果列表路径
    :return: 软件包结果列表
    """
    dirs = [each for each in os.listdir(path) if
            os.path.isdir(os.path.join(path, each))]
    result = []
    for time_path in dirs:
        files = os.listdir(os.path.join(path, time_path))
        files = [f for f in files if
                 os.path.isfile(os.path.join(path, time_path, f))]
        if files:
            name = files[0]
            pkg_info = dict()
            pkg_info["name"] = name
            pkg_info["path"] = time_path
            filepath = os.path.join(path, time_path, name)
            ts = os.stat(filepath).st_mtime
            dt = time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(ts))
            pkg_info["create_time"] = dt
            result.append(pkg_info)
    result.sort(key=cmp_to_key(rpm_list_sort))
    return result


@api_view(['GET', 'DELETE'])
@common_util.check_if_user_login()
def history(request):
    """
    历史文件列表和删除接口
    :param request: 请求
    """
    package_path = get_package_path(request)
    if request.method == "GET":
        # 获取文件列表
        result = get_package_list(package_path)
        his_report_num_status = _check_his_report_nums(len(result))
        info = his_report_num_status.get_msg()
        info_chinese = his_report_num_status.get_cn_msg()
        data = {"histasknumstatus": his_report_num_status.get_code(),
                "tasklist": result}
        return SuccessResponse(info, info_chinese, data)

    name = request.GET.get("name")
    path = request.GET.get("path")
    if not name and not path:
        # 清空所有文件
        try:
            shutil.rmtree(os.path.dirname(package_path))
            LOGGER.info("Successfully deleted all software rebuild history "
                        "in %s", package_path)
            utils.insert_operationlog(request.user, 43, 0, datetime.now(),
                                      [113, package_path])
            return SuccessResponse("Delete file success.", "删除文件成功。")
        except Exception as err:
            LOGGER.error("Failed to delete file. Except: %s", err)
        return FailResponse("Failed to delete file.", "删除文件失败。")

    # 删除单个文件
    realpath = os.path.realpath(os.path.join(package_path, path, name))
    if not os.path.dirname(os.path.dirname(realpath)) == package_path:
        return FailResponse("Failed to delete file.", "删除文件失败。")
    if not os.path.exists(realpath):
        LOGGER.info("File: %s not found.", realpath)
        info = "File not found"
        info_chinese = u"文件不存在。"
        return FailResponse(info, info_chinese)
    try:
        shutil.rmtree(os.path.dirname(realpath))
        LOGGER.info("Successfully deleted the software rebuild history in"
                    " %s" % realpath)
        utils.insert_operationlog(request.user, 43, 0, datetime.now(),
                                  [114, realpath])
    except Exception as err:
        LOGGER.error("Failed to delete file. Except: %s", err)
        return FailResponse("Failed to delete file.", "删除文件失败。")
    return SuccessResponse("Delete file success.", "删除文件成功。")
```

