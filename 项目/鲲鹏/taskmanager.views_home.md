```python
"""
查询首页系统信息接口
"""
import os
import json
from datetime import datetime
import shutil

from django.db.models import Q
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response

from resource import utils
from util.check_space import check_space
from util.request_check import check_before_upload_whitelist, \
    del_exist_upgrade_package, check_platform, FILE_CHECK_RETURN_INFO
from util.response import SuccessResponse, FailResponse, CommonResponse
from util import task_util, common_util
from util.verify_view import upload_verify, validate_data_type
from usermanager.models import User
from taskmanager.models import Task
from taskmanager.whitelist_task import TaskQueue, TaskWhitelist
from taskmanager.migration_scan_task import MigrationScanTask, \
    MigrationTaskQueue
from upload_tool import file_upload
from upload_tool.upload_control import upload_control
from tool.porting.api.scan_api import ResetTaskStatus
from tool.kit_config import OsConfig, GccConfig, KitConfig, GfortranConfig
from .views import check_path, get_file_content

LOGGER = utils.get_logger()

WEAK_CONSISTENCY_COMPILE_TYPE = '7'
# 请求状态码
STATUS_SUCCESS = 0
STATUS_FAILED = 1
TASK_TYPE = ["whitelist", "solution", "solution_manage", "autopack"]
EVENT = {
    'whitelist_upload': 24,
    'upload_success': 50,
    'upload_failed': 51,
    "delete_file": 43,
}
RESPONSE_INFO = {
    "DELETE_FILE_SUCCESS":
        [STATUS_SUCCESS, "Delete file success", "删除文件成功"],
    "DELETE_FILE_FAIL": [STATUS_FAILED, "Delete file fail", "删除文件失败"],
    "EXCEPTION": [STATUS_FAILED, "Exception", "删除文件异常"],
    "FILE_NOT_EXIST": [STATUS_FAILED, "File not exist",
                       "删除的文件不存在"],
    "NO AUTHORITY": [STATUS_FAILED, "No authority", "权限不足"],
    "MISSING PARAMETER": [STATUS_FAILED, "Missing parameter", "缺少参数"],
    "CONTACT ADMINISTRATOR": [STATUS_FAILED, "Contact administrator to delete",
                              "请联系管理员删除"],
    "PATH DOES NOT EXIST": [STATUS_FAILED, "Path doesnt exist", "路径不存在"],
    "NAME_INVALID": [STATUS_FAILED, "Name invalid", "无效名字"],
    "PARAMETER_INVALID": [STATUS_FAILED, "Parameter invalid", "参数不合法"],
    "FILE_NAME_INVALID": [STATUS_FAILED,
                          "The file or folder name cannot contain spaces or "
                          "special characters such as ^ ` / | ; & $ > < \ !",
                          r"文件或文件夹名中不能包含空格以及^ ` / | ; & $ > "
                          r"< \ ! 等特殊字符"]
}

CHOICE = {
    'OVERRIDE': 'override',
    'NORMAL': 'normal',
    'SAVE_AS': 'save_as',
}


@api_view(['GET'])
@common_util.check_if_user_login()
def system_list_info(request):
    """
    查询首页系统信息接口
    :param request:
    :return: get
    """
    request.user.update_op_time()
    os_system_list = []
    for vaild_os_version in OsConfig.valid_os_version_transfer:
        os_system = dict()
        os_system['os_system'] = vaild_os_version
        vaild_os_version = vaild_os_version.lower().replace(' ', '')
        kernel = OsConfig.target_porting_os.get(
            vaild_os_version, '').get('kernel')
        os_system['kernel_default'] = kernel
        gcc = OsConfig.target_porting_os.get(
            vaild_os_version, '').get('gcc').upper()
        if gcc:
            gcc = list(gcc)
            gcc.insert(3, ' ')
            gcc = ''.join(gcc)
        os_system['gcc_default'] = gcc
        os_system_list.append(os_system)
    data = {
        'os_system_list': os_system_list,
        'gcc_list': GccConfig.valid_compiler_info_upper,
        'fortran_list': GfortranConfig.valid_compiler_info_upper,
        'construct_tools': KitConfig.valid_construct_tools
    }
    return SuccessResponse('success', '成功', data)


@api_view(['POST'])
@common_util.check_if_user_login()
def manage_whitelist(request):
    """
    创建白名单管理任务
    """
    request.user.update_op_time()
    response = check_space()
    response_body = {}
    if response:
        return Response(response, status.HTTP_200_OK)

    try:
        task_info = int(request.data.get("option", ""))
    except ValueError as ex:
        LOGGER.info("Failed to convert the data type. Expect: %s" % str(ex))
        response_body["status"] = STATUS_FAILED
        response_body['info'] = "Invalid parameter"
        response_body['infochinese'] = u"非法参数"
        response_body['data'] = ""
        return Response(response_body, status=status.HTTP_400_BAD_REQUEST)

    # 判断是否为有效的管理员用户和有效操作
    status_code, info, info_chinese, data = admin_user_check(request)
    if status_code == STATUS_FAILED:
        return FailResponse(info, info_chinese, data)

    LOGGER.debug('whitelist POST begin!')
    status_code, info, info_chinese, data = \
        create_whitelist_task(request, task_info)
    LOGGER.debug('whitelist POST end!')
    return CommonResponse(status_code, info, info_chinese, data)


def create_whitelist_task(request, task_info):
    """ 创建白名单任务，判断当前是否有任务在运行 """
    username = request.user.username
    status_code = STATUS_FAILED
    info = 'Operation failed.'
    info_chinese = u"操作失败。"
    data = {"task_name": ""}
    try:
        current = datetime.now().strftime('%Y%m%d%H%M%S')
        # 判断是否有: 1.同一时刻任务, 2.非白名单任务, 3.扫描和重打包任务
        task_judge = Task.objects.filter(
            (Q(user_name=username) & Q(task_type=2) &
             (Q(task_name=current) | Q(progress__lt=100))) |
            (Q(task_type__in=[0, 1]) & Q(status=1)) |
            (Q(task_type=7) & Q(status=2))
        ).exists()

        if not task_judge:
            taskrun = TaskWhitelist(username, task_info)
            if TaskQueue().add_task(taskrun):
                data = {"task_name": taskrun.task_name}
                utils.insert_operationlog(
                    username, 13 + task_info, 0, datetime.now(),
                    [24 + (task_info * 2), ''])
                status_code = STATUS_SUCCESS
                LOGGER.info('Success to add whitelist task. Option '
                            'code: %d', task_info)
                info = 'Successful operation.'
                info_chinese = u"操作成功。"
            else:
                # 添加任务失败，删除数据库无用任务
                Task.objects.filter(id=taskrun.task_id).delete()
                utils.insert_operationlog(
                    username, 13 + task_info, 1, datetime.now(),
                    [25 + (task_info * 2), ''])
                LOGGER.info('Failed to add whitelist task. Option '
                            'code: %d', task_info)
        else:
            utils.insert_operationlog(
                username, 13 + task_info, 1, datetime.now(),
                [25 + (task_info * 2), ''])
            LOGGER.info('Failed to add whitelist task. Option code: %d',
                        task_info)
            info = 'Failed to perform the whitelist operation ' \
                   'because a user is performing another task.'
            info_chinese = u"有用户正在执行其它任务，无法执行白名单相关操作。"
    except (IOError, OSError, ValueError, TypeError) as error:
        utils.insert_operationlog(
            username, 13 + task_info, 1, datetime.now(),
            [25 + (task_info * 2), ''])
        LOGGER.error('Failed to add whitelist task. '
                     'Option code: %d. Except:%s.', task_info, error)
    return status_code, info, info_chinese, data


@api_view(['POST'])
@common_util.check_if_user_login()
@common_util.check_is_admin_user()
def delete_whitelist_package(request):
    """
    功能:删除升级包
    参数:无
    返回值: 删除状态
    """
    return del_exist_upgrade_package(request)


@api_view(['POST', 'GET'])
@common_util.check_if_user_login()
@common_util.check_is_admin_user()
def whitelist_upload(request):
    """
    白名单上传
    """
    response = check_space()
    if response:
        return Response(response, status.HTTP_200_OK)
    # 判断是否为有效的管理员用户和有效操作
    username = str(request.user)
    if request.method == 'GET':
        return check_before_upload_whitelist(request)
    # 验证上传文件是否指定包
    filename = request.META.get("HTTP_FILENAME")
    if not filename.startswith('Whitelist-package') \
            or not filename.endswith('.tar.gz'):
        utils.insert_operationlog(
            username,
            EVENT['whitelist_upload'], STATUS_FAILED, datetime.now(),
            [EVENT['upload_failed'], '']
        )
        res_info = FILE_CHECK_RETURN_INFO.get('FILE_ERROR')
        return FailResponse(res_info.get('INFO'), res_info.get('CH_INFO'))
    # 上传文件，并根据上传情况返回失败还是成功
    if request.method == 'POST':
        filepath = os.path.normpath(
            os.path.join(KitConfig.tool_root_dir, username))
        upload_response = upload_verify(request, filepath)
        if upload_response.status_code != status.HTTP_200_OK:
            utils.insert_operationlog(username, EVENT['whitelist_upload'],
                                      STATUS_FAILED, datetime.now(),
                                      [EVENT['upload_failed'], ''])
            LOGGER.info("Failed to upload whitelist package.")
        else:
            utils.insert_operationlog(username, EVENT['whitelist_upload'],
                                      STATUS_SUCCESS, datetime.now(),
                                      [EVENT['upload_success'], filepath])
            LOGGER.info("Successfully uploaded the whitelist package to "
                        "%s", filepath)
        return upload_response


@api_view(['POST'])
@common_util.check_if_user_login()
def reset_status(request):
    """
    清除任务状态
    """
    request.user.update_op_time()
    response = check_space()
    if response:
        return Response(response, status.HTTP_200_OK)

    try:
        task_type = request.data.get("tasktype", "")
        LOGGER.info("start reset %s task status!", task_type)
        if task_type not in TASK_TYPE:
            status_code = STATUS_FAILED
            info = "Wrong task type"
            info_chinese = "错误的任务类型"
        else:
            reset = ResetTaskStatus(task_type)
            result = reset.clear_status()
            if result:
                status_code = STATUS_SUCCESS
                info = "clear task status success"
                info_chinese = "任务状态清除成功"
            else:
                status_code = STATUS_FAILED
                info = "task status error"
                info_chinese = "任务状态异常"
    except Exception as err:
        LOGGER.error(err)
        status_code = STATUS_FAILED
        info = "unkown error"
        info_chinese = "未知错误"
    return CommonResponse(status_code, info, info_chinese)


def admin_user_check(request):
    """
    对输入参数进行统一的校验
    """
    user = User.objects.filter(username=request.user)
    status_code = STATUS_SUCCESS
    info = ""
    info_chinese = u""
    data = {}
    # 判断是否为管理员用户
    if user[0].id != 1:
        LOGGER.error('User permission error.')
        status_code = STATUS_FAILED
        info = "User permission error."
        info_chinese = u"用户权限错误。"
    else:
        # 二次密码校验
        whitelist_option = int(request.data.get("option", ""))
        if whitelist_option not in [0, 1, 2]:
            LOGGER.error('the whitelist option is wrong')
            status_code = STATUS_FAILED
            info = "Option parameter is wrong"
            info_chinese = u"输入的参数错误。"
        else:
            admin_password = request.data.get("password", "")
            admin_username = User.objects.get(id=1)
            if not admin_username.second_password_validate(admin_password):
                LOGGER.error('The administrator password is incorrect.')
                operation_number = [25, 27, 29]
                operation_event = [13, 14, 15]
                status_code = STATUS_FAILED
                info = "The password of the current user is incorrect or " \
                       "the user has been locked. "
                info_chinese = "当前用户密码错误或用户已锁定"
                # 写入日志
                utils.insert_operationlog(
                    username=admin_username,
                    event=operation_event[whitelist_option],
                    result=1,
                    times=datetime.now(),
                    detail=[operation_number[whitelist_option], ''])
            del admin_password
    return status_code, info, info_chinese, data


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
    file_name = request.data.get("file_name", "")
    file_size = request.data.get("file_size", "")
    need_unzip = request.data.get("need_unzip", "")
    scan_type = request.data.get("scan_type", "")
    code_path = request.data.get("code_path", "")
    if not isinstance(file_name, str) or not isinstance(code_path, str):
        err_code = STATUS_FAILED
        info = 'Incorrect parameter type.'
        ch_info = u'参数类型错误。'
        data = {}
        return CommonResponse(err_code, info, ch_info, data)
    if not scan_type or scan_type not in task_util.SCAN_TYPE.keys():
        err_code = STATUS_FAILED
        info = "Input parameter error."
        ch_info = "输入参数错误。"
        data = {}
        return CommonResponse(err_code, info, ch_info, data)
    if common_util.is_name_contain_special_character(file_name)[0]:
        return response_content('FILE_NAME_INVALID')

    choice = request.data.get("choice")
    if choice == CHOICE['SAVE_AS']:
        invaild = common_util.check_name(file_name)
        if invaild:
            return response_content('NAME_INVALID')
    file_path = \
        os.path.join(KitConfig.tool_root_dir, user_name,
                     task_util.SCAN_TYPE[scan_type], code_path)
    if scan_type == WEAK_CONSISTENCY_COMPILE_TYPE:
        err_code, info, ch_info, data = file_upload.UploadFile(request). \
            compile_file_upload_check(file_name, file_size, need_unzip,
                                      file_path, choice)
    else:
        err_code, info, ch_info, data = file_upload.UploadFile(request). \
            file_upload_check(file_name, file_size, need_unzip, file_path,
                              choice)
    LOGGER.info(info)
    if err_code:
        utils.insert_operationlog(user_name, 22, err_code, datetime.now(),
                                  [46, ""])
    return CommonResponse(err_code, info, ch_info, data)


@api_view(['POST'])
@common_util.check_if_user_login()
def upload(request):
    """
    上传文件接口
    :param request:
    :return: 上传结果
    """
    response = check_space()
    if response:
        return Response(response, status.HTTP_200_OK)
    user_name = request.user
    scan_type = request.META.get("HTTP_SCAN_TYPE")
    code_path = request.META.get("HTTP_CODE_PATH", "")
    if not scan_type or scan_type not in task_util.SCAN_TYPE.keys():
        err_code = STATUS_FAILED
        info = "Input parameter error."
        ch_info = "输入参数错误。"
        data = {}
        return CommonResponse(err_code, info, ch_info, data)
    file_path = \
        os.path.join(KitConfig.tool_root_dir, str(user_name),
                     task_util.SCAN_TYPE[scan_type], code_path)
    file_field = "file"
    # 接收文件，验证，解压

    choice = request.META.get("HTTP_CHOICE", '')

    if scan_type == WEAK_CONSISTENCY_COMPILE_TYPE:
        result = upload_control(file_field, file_path, request,
                                choice, [".out"])
    else:
        result = upload_control(file_field, file_path, request, choice)
    if len(result) == 3:
        result = result.copy()
        result.append("")
    err_code, info, ch_info, data = result
    LOGGER.info(info)
    if err_code:
        utils.insert_operationlog(user_name, 22, err_code, datetime.now(),
                                  [46, ""])
    else:
        LOGGER.info("The file uploaded to:%s", file_path)
        utils.insert_operationlog(user_name, 22, err_code, datetime.now(),
                                  [47, file_path])
    return CommonResponse(err_code, info, ch_info, data)


@api_view(['DELETE'])
@common_util.check_if_user_login()
def delete_file(request):
    """
    删除文件接口
    :param request: 无
    :return: 删除文件的结果
    """
    user = request.user
    file_path, workspace_path, response_info = check_param(request)
    if response_info:
        return response_info
    file_path = os.path.realpath(file_path)
    if not file_path.startswith(workspace_path):
        return response_content("PARAMETER_INVALID")
    if not os.path.exists(file_path):
        utils.insert_operationlog(user.username, EVENT.get("delete_file"), 1,
                                  datetime.now(), [101, ''])
        LOGGER.error("The file to be deleted does not exist.")
        return response_content("FILE_NOT_EXIST")
    try:
        if os.path.isfile(file_path):
            os.remove(file_path)
        else:
            shutil.rmtree(file_path)
        msg = "Successfully deleted the file in %s" % file_path
        LOGGER.info(msg)
        utils.insert_operationlog(request.user, EVENT.get("delete_file"), 0,
                                  datetime.now(), [100, file_path])
        return response_content("DELETE_FILE_SUCCESS")
    except PermissionError as err:
        LOGGER.warning("Failed to delete file. Except: %s", err)
        utils.insert_operationlog(user.username, EVENT.get("delete_file"), 1,
                                  datetime.now(), [102, ''])
        return response_content("NO AUTHORITY")
    except OSError as e:
        LOGGER.warning(str(e))
        utils.insert_operationlog(request.user, EVENT.get("delete_file"), 1,
                                  datetime.now(), [112, ""])
        return response_content("DELETE_FILE_FAIL")


def check_param(request):
    """
    下拉框删除文件时的参数检查
    :param request:
    :return:
    """
    file_name = request.data.get('file_name', '')
    path = request.data.get('path', '')
    user = request.user
    # 类型校验
    for data in [file_name, path]:
        if data:
            failed_response = validate_data_type(data)
            if failed_response:
                return "", "", failed_response
    workspace = user.workspace
    if not all((file_name, path)):
        utils.insert_operationlog(request.user, EVENT.get("delete_file"), 1,
                                  datetime.now(), [103, ''])
        LOGGER.error("The path or file name is missing.")
        return "", "", response_content("MISSING PARAMETER")
    if path in task_util.RULE_PATH:
        file_path = os.path.join(workspace, path, file_name)
        workspace_path = workspace + path + '/'
        return file_path, workspace_path, ""
    else:
        utils.insert_operationlog(request.user, EVENT.get("delete_file"), 1,
                                  datetime.now(), [105, ''])
        LOGGER.error("The path does not exist.")
        return "", "", response_content("PATH DOES NOT EXIST")


def response_content(content):
    """
   返回结果信息
   :param : content
   :return: 结果信息
    """
    err_code = RESPONSE_INFO[content][0]
    info = RESPONSE_INFO[content][1]
    ch_info = RESPONSE_INFO[content][2]
    data = ''
    return CommonResponse(err_code, info, ch_info, data)


@api_view(['POST'])
@common_util.check_if_user_login()
def build_migration_scan(request):
    """
    x86平台32位-64位迁移
    :param request:
    :return: 迁移任务建立结果
    """
    response = check_space()
    if response:
        return Response(response, status.HTTP_200_OK)
    user_name = request.user.username
    scan_file = request.data.get("scan_file", "")  # 扫描文件名
    os_mapping_dir = request.data.get("os_mapping_dir", "")
    status_code, info, info_chinese = check_platform()
    if status_code == STATUS_FAILED:
        utils.insert_operationlog(user_name, 47, 1, datetime.now(),
                                  [5, '64-bit porting precheck'])
        return FailResponse(info, info_chinese)
    if not scan_file:
        info = "missing scan parameters"
        info_chinese = "缺少扫描参数"
        return FailResponse(info, info_chinese)
    # 类型校验
    failed_response = validate_data_type(scan_file)
    if failed_response:
        return failed_response
    user = User.objects.get(username=user_name)
    work_space = os.path.join(user.workspace, task_util.PRE_CHECK)
    file_path = os.path.join(
        KitConfig.tool_root_dir, str(user_name), 'precheck', scan_file)
    file_path = os.path.realpath(file_path)
    success, info, info_chinese = check_path(work_space, [file_path])
    if not success:
        return FailResponse(info, info_chinese)

    status_code, info, info_chinese, data = \
        create_migration_scan_task(user_name, file_path, os_mapping_dir)
    return CommonResponse(status_code, info, info_chinese, data)


def create_migration_scan_task(user_name, file_path, os_mapping_dir):
    """ 创建x86平台32位-64位迁移任务 """
    task_name = datetime.now().strftime('%Y%m%d%H%M%S')
    scan_param = {"task_type": 5, "source_path": file_path}
    task = init_migration_scan_task(task_name, user_name, file_path,
                                    os_mapping_dir)
    status_code, data = STATUS_FAILED, {}
    task_others = Task.objects.filter(user_name=user_name,
                                      task_type=5, status=1).exists()
    if task_others:
        status_code = STATUS_FAILED
        LOGGER.error('Failed to create task.')
        info = 'A porting pre-check task is being executed. ' \
               'Please try again later.'
        info_chinese = u"当前用户正在执行迁移预检任务，请稍后再试"
        utils.insert_operationlog(user_name, 47, 1, datetime.now(),
                                  [5, '64-bit porting precheck'])
        return status_code, info, info_chinese, data
    task_others = Task.objects.filter(user_name=user_name,
                                      task_type=6, status=1).exists()
    if task_others:
        status_code = STATUS_FAILED
        LOGGER.error('Failed to create task.')
        info = 'A structure type definition alignment check task is ' \
               'being executed. Please try again later.'
        info_chinese = u"当前用户正在执行结构类型定义对齐检查任务，请稍后再试"
        utils.insert_operationlog(user_name, 47, 1, datetime.now(),
                                  [5, '64-bit porting precheck'])
        return status_code, info, info_chinese, data
    try:
        taskrun = MigrationScanTask(user_name, scan_param, task_name)
        task.save()
        if MigrationTaskQueue().add_task(taskrun):
            status_code = STATUS_SUCCESS
            info, info_chinese = 'Task creation successful.', u"创建任务成功。"
            data = {'id': task_name}
            utils.insert_operationlog(user_name, 47, 0, datetime.now(),
                                      [4, '64-bit porting precheck'])
        else:
            task.delete()
            LOGGER.error('Task creation failed.')
            info, info_chinese = 'Task creation failed.', u"创建任务失败。"
            utils.insert_operationlog(user_name, 47, 1, datetime.now(),
                                      [5, '64-bit porting precheck'])
    except (IOError, OSError, ValueError, TypeError) as exp:
        LOGGER.error('Task creation failed: %s' % str(exp))
        status_code = STATUS_FAILED
        info = '%s' % str(exp)
        info_chinese = u"创建任务异常"
        utils.insert_operationlog(user_name, 47, 1, datetime.now(),
                                  [5, '64-bit porting precheck'])
    return status_code, info, info_chinese, data


def init_migration_scan_task(task_name, user_name, file_path, os_mapping_dir):
    """ 初始化x86平台32位-64位迁移任务 """
    task = Task(
        task_name=task_name,
        user_name=user_name,
        status=1,
        source_dir=file_path,
        task_type=5,
        os_mapping_dir=os_mapping_dir)
    return task


@api_view(['DELETE'])
@common_util.check_if_user_login()
def migration_scan_process(request, task_id):
    """
    32-64位迁移任务终止
    :param request:
    :param task_id:
    :return: del get
    """
    return task_util.stop(request, task_id, task_util.PORT_MIGRATION)


@api_view(['POST'])
@common_util.check_if_user_login()
def content_check(request):
    """
    获取迁移文件内容和迁移建议
    :param request:
    :return: task_id
    """
    file_path = request.data.get("file_path", "")
    task_name = request.data.get("task_name", "")
    if file_path:
        failed_response = validate_data_type(file_path)
        if failed_response:
            return failed_response
    pre_check_workspace = os.path.join(request.user.workspace, 'precheck')
    result, msg = common_util.check_path_under_workspace(pre_check_workspace,
                                                         file_path)
    # 若用户访问路径不在其工作路径下，则返回相应失败信息
    if not result:
        return FailResponse(*msg)
    try:
        content = get_file_content(file_path)
    except IOError as error:
        info = "The path does not exist."
        info_chinese = u"路径不存在。"
        LOGGER.error(info + " Except:%s.", error)
        return FailResponse(info, info_chinese)

    task = Task.objects.get(
        task_name=task_name, user_name=request.user, task_type=5)
    if not task:
        info = "Failed to query the task list because the task or " \
               "user does not exist."
        info_chinese = u"查询任务列表失败，任务或用户不存在。"
        return FailResponse(info, info_chinese)
    scan_result_json = json.loads(task.scan_result)
    info = "Porting suggestion query succeeded."
    info_chinese = u"查询迁移建议成功。"
    data = {
        "content": content,
        "suggestioncontent": "",
        "line": scan_result_json.get(file_path)
    }
    LOGGER.debug(info)
    return SuccessResponse(info, info_chinese, data)


@api_view(['GET'])
@common_util.check_if_user_login()
def migration_task_query(request):
    """
    查询当前用户是否有正在执行的32-64位迁移任务
    :param request:
    :return:
    """
    User.objects.filter(
        username=request.user).update(last_operating=datetime.now())
    LOGGER.debug('Query task undone')
    info = "Task execution complete."
    info_chinese = u"任务已全部执行完成。"
    data = {}
    LOGGER.debug('task_list check if there is running task')
    task_dict = Task.objects.filter(
        status=1, user_name=request.user, task_type=5)
    if task_dict:
        working_task = task_dict[0]
        if str(working_task.user_name) == str(request.user):
            info = "A task is running."
            info_chinese = u"有任务正在运行。"
            LOGGER.error('%s is running', working_task.task_name)
            data = {"id": working_task.task_name}
    return SuccessResponse(info, info_chinese, data)
```

