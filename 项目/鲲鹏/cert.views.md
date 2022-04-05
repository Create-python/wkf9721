```python
#! /usr/bin/env python
# -*- coding: utf-8 -*-

"""
Copyright: Copyright (c) Huawei Technologies Co., Ltd. 2020-2020.
All rights reserved.
Create: 2020-06-03
Content: 证书查看
"""
import os
import re
import stat
import time
import logging
import threading
import subprocess
from datetime import datetime
from datetime import timedelta
from ctypes import CDLL

from django.db import IntegrityError
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response

from cert.models import Cert
from tool.logger.logger import Logger
from tool.util.common_util import CommonUtil
from tool.kit_config import KitConfig
from timeout_configuration.models import SystemConfiguration
from util.openssl_rand import generate_password
from util.response import SuccessResponse, FailResponse, generate_response_body
from cert.validator import CsrForm
from util import common_util as utils
from util.operation_log_util import save_operation_log


# 请求状态码
STATUS_SUCCESS = 0
STATUS_FAILED = 1

# datetime format
CTIME_FT = "%b %d %H:%M:%S %Y %Z"
ROOT_PATH = CommonUtil.get_customize_path().get_tool_root_dir()
CERT_PATH = os.path.join(ROOT_PATH, 'tools', 'webui', 'cert.pem')

_MIN_ALARM_CERT_TIME = 7
_MAX_ALARM_CERT_TIME = 180
_DEFAULT_ALARM_CERT_TIME = 90

LOGGER = logging.getLogger(Logger.LOGGER_NAME)
PARENT_FILE_PERMISSION = '500'
FIZESIZE = 1000
PUBLIC_KEY_MIN_LEN = 2048
RESTART_NGINX_WAIT_TIME = 6
ROOT_PATH = CommonUtil.get_customize_path().get_tool_root_dir()
INSTALL_PATH = CommonUtil.get_customize_path().get_customize_path()
# 传给dep_port_crypto的webui路径后需要带/
WEBUI_PATH = os.path.join(ROOT_PATH, 'tools', 'webui', '')
DELETE_FILE_SCRIPT = os.path.join(WEBUI_PATH, "delete_file.sh")
NEW_PATH = os.path.join(WEBUI_PATH, 'common', 'new')
BACKUP_PATH = os.path.join(WEBUI_PATH, 'common', 'backup')
NGINX_PATH = os.path.join(WEBUI_PATH, 'common', 'nginx')
NGINX_BAK_PATH = os.path.join(WEBUI_PATH, 'common', 'nginx_bak')
KRONOS_PATH = os.path.join(WEBUI_PATH, 'common', 'kronos')
WEBUI_KEY_PATH = os.path.join(WEBUI_PATH, "cert.key")
WEBUI_PEM_PATH = os.path.join(WEBUI_PATH, "cert.pem")
WEBUI_ROOT_CERT_PATH = os.path.join(WEBUI_PATH, "ca.crt")
NEW_KEY_PATH = os.path.join(NEW_PATH, "cert.key")
NEW_PEM_PATH = os.path.join(NEW_PATH, "cert.pem")
BACKUP_KEY_PATH = os.path.join(BACKUP_PATH, "cert.key")
BACKUP_PEM_PATH = os.path.join(BACKUP_PATH, "cert.pem")
CRYPTO_SO_PATH = os.path.join(WEBUI_PATH, 'dep_port_crypto.so')
SUBJ_FIELD_MAP = {
    "country": "/C",
    "common_name": "/CN",
    "state": "/ST",
    "locality": "/L",
    "organization": "/O",
    "organizational_unit": "/OU",
}
NGINX_SHELL = "service_nginx.sh"
if KitConfig.tool_name == KitConfig.DEPENDENCY:
    from log_manager import common
    NGINX_SERVICE = "nginx_dep"
    SERVICE_NGINX_PATH = os.path.join(WEBUI_PATH, "script_dep", NGINX_SHELL)
    GENERATE_CSR_EVENT = 32
    EXPORT_CERT_EVENT = 33
    RESTART_NGINX_EVENT = 34
else:
    from resource.utils import insert_operationlog
    NGINX_SERVICE = "nginx_port"
    SERVICE_NGINX_PATH = os.path.join(WEBUI_PATH, "script_port", NGINX_SHELL)
    GENERATE_CSR_EVENT = 40
    EXPORT_CERT_EVENT = 41
    RESTART_NGINX_EVENT = 42
SYSTEMD_RESTART_CMD = ["sudo", "systemctl", "restart", NGINX_SERVICE]
SERVICE_STOP_CMD = ["sudo", "service", NGINX_SERVICE, "stop"]
SERVICE_START_CMD = ["sudo", "service", NGINX_SERVICE, "start"]
INSTALL_PATH = CommonUtil.get_customize_path().get_customize_path()
NGINX_STATUS_CMD = [SERVICE_NGINX_PATH, "status", INSTALL_PATH]
DOCKER_STOP_NGINX_CMD = ["sudo", SERVICE_NGINX_PATH, "stop", INSTALL_PATH]
DOCKER_START_NGINX_CMD = ["sudo", SERVICE_NGINX_PATH, "start", INSTALL_PATH]
PASSWORD_BYTE_LEN = 15


class RangeException(Exception):
    def __init__(self, errorvalue):
        self.value = errorvalue

    def __str__(self):
        return self.value


class Range:
    def __init__(self, value):
        if not (_MIN_ALARM_CERT_TIME <= value <= _MAX_ALARM_CERT_TIME):
            raise RangeException("Out of range")


def start_cert_reader():
    """
    起线程检查证书有效性并记录
    """
    try:
        cert_thread = threading.Thread(target=update_cert_info, args=())
        cert_thread.setDaemon(True)
        cert_thread.start()
    except SystemError as system_error:
        LOGGER.error("Failed to start cert thread. Except:%s.", system_error)


def update_cert_info():
    """
    每天2:00更新人机证书信息
    """
    while True:
        time.sleep(300)
        now = datetime.now()
        if now.hour == 2:
            read_cert_info('cert', CERT_PATH)


def read_cert_info(name, path):
    """
    读取证书信息
    """
    state = Cert.STATE_EXPIRED
    exp_time = check_cert_exp_date(path)  # 过期时间
    if exp_time:
        now = datetime.now()
        cert_time = query_cert_time_configuration()
        if exp_time > now + timedelta(days=cert_time):
            state = Cert.STATE_VALID
        elif exp_time > now:
            state = Cert.STATE_EXPIRING
        else:
            state = Cert.STATE_EXPIRED
    try:
        cert = Cert.objects.get(cert_name=name)
    except Cert.DoesNotExist as not_exist:
        LOGGER.warning("The certificate does not exist. Except:%s.", not_exist)
        cert = Cert(cert_name=name)
    cert.cert_expired = exp_time
    cert.cert_flag = state
    try:
        cert.save()
    except IntegrityError as integrity_error:
        LOGGER.error("The cert is already exist. Except:%s.", integrity_error)


def check_cert_exp_date(cert_file):
    """
    使用OPENSSL命令获取证书的失效日期
    """
    try:
        cmd = ['openssl', 'x509', '-in', cert_file, '-noout', '-enddate']
        cmd_rst = subprocess.Popen(cmd, stdout=subprocess.PIPE,
                                   stderr=subprocess.PIPE, shell=False)
        orig_date = cmd_rst.stdout.read()
        tran_date = parse_datetime(orig_date)
        return tran_date
    except Exception as error:
        err = "Parse the certificate file failed by command. Except:%s" % error
        LOGGER.error(err)


def parse_datetime(orig_date):
    """
    解析OpenSSL命令获取到的时间，返回datetime格式
    """
    try:
        not_after_date, _ = orig_date.split('\n'.encode())
        _, date_time = not_after_date.split('='.encode())
        tran_date = datetime.strptime(date_time.decode(), CTIME_FT)
        return tran_date
    except Exception as error:
        err = "Parse certificate expire time failed. Except:%s." % error
        LOGGER.error(err)


@api_view(['GET'])
@utils.check_if_user_login()
def check_cert_state(_):
    """
    证书有效性查询接口
    """
    try:
        cert = Cert.objects.get(cert_name='cert')
    # 证书信息不存在
    except Cert.DoesNotExist as not_exist:
        LOGGER.warning("The certificate does not exist. Except:%s.", not_exist)
        info = "Failed to read cert information."
        info_chinese = u"读取证书信息失败。"
        LOGGER.error(info)
        return FailResponse(info, info_chinese)

    # UTC时间转成服务器本地时间
    time_diff = datetime.now() - datetime.utcnow()
    local_expired_time = cert.cert_expired + time_diff
    log_info = "Read cert successfully. cert_flag:{}, cert_expired" \
               ":{}".format(cert.cert_flag,
                            local_expired_time.strftime('%Y-%m-%d %H:%M:%S %f'))
    info = "Read cert successfully."
    info_chinese = u"读取证书信息成功。"
    data = {"cert_flag": cert.cert_flag,
            "cert_expired": local_expired_time.strftime('%Y-%m-%dT%H:%M:%S')}
    LOGGER.info(log_info)
    return SuccessResponse(info, info_chinese, data)
def operation_log(user, state):
    if state == "SUCCESS":
        if KitConfig.tool_name == KitConfig.DEPENDENCY:
            user_id = user.get_id()
            log_event = common.OpEvent.CONFIGURATE_ALARM_TIME
            log_mes = "The Certificate Expiry Alarm Threshold" \
                      " is changed successfully. "
            log_result = common.OpResult.SUCCESS
            common.save_operation_log(user_id, log_event, log_result, log_mes)
        else:
            username = user
            event = 38
            detail = [90, '']
            insert_operationlog(username, event, 0, datetime.now(), detail)
    else:
        if KitConfig.tool_name == KitConfig.DEPENDENCY:
            user_id = user.get_id()
            log_event = common.OpEvent.CONFIGURATE_ALARM_TIME
            log_mes = "The Certificate Expiry Alarm Threshold" \
                      " is changed unsuccessfully. "
            log_result = common.OpResult.FAIL
            common.save_operation_log(user_id, log_event, log_result,
                                      log_mes)
        else:
            username = user
            event = 38
            detail = [91, '']
            insert_operationlog(username, event, 0, datetime.now(), detail)


def query_cert_time_configuration():
    system_configure = SystemConfiguration.objects.filter(
        config_name="cert_time").first()
    if not system_configure:
        system_configure = SystemConfiguration(config_name="cert_time")
        system_configure.config_value = _DEFAULT_ALARM_CERT_TIME
        config_value = _DEFAULT_ALARM_CERT_TIME
    else:
        try:
            config_value = int(system_configure.config_value)
            Range(config_value)
        except ValueError as error:
            LOGGER.error("Failed to get cert time configuration. "
                         "Except: %s" % error)
            config_value = _DEFAULT_ALARM_CERT_TIME
            system_configure.config_value = _DEFAULT_ALARM_CERT_TIME
        except RangeException as err:
            LOGGER.error("Failed to get cert time configuration. "
                         "Except: %s" % err)
            config_value = _DEFAULT_ALARM_CERT_TIME
            system_configure.config_value = _DEFAULT_ALARM_CERT_TIME
    system_configure.save()
    return config_value


def check_cert_time(cert_time):
    if isinstance(cert_time, bool):
        # 避免是布尔值
        return False
    return isinstance(cert_time, int) and \
        _MIN_ALARM_CERT_TIME <= cert_time <= _MAX_ALARM_CERT_TIME


def update_cert_time_configuration(cert_time):
    """
    功能描述：处理证书到期时间告警配置逻辑
    参数：用户名
    返回值：响应体
    异常描述：
    修改记录：首次开发
    """
    system_configure = SystemConfiguration.objects.filter(
        config_name="cert_time").first()
    if not system_configure:
        system_configure = SystemConfiguration(config_name="cert_time")
        system_configure.config_value = cert_time
        system_configure.save()
    else:
        system_configure.config_value = cert_time
        system_configure.save()
    # 构建响应体
    status_code = STATUS_SUCCESS
    data = {"cert_time": cert_time}
    info = "The Certificate Expiry Alarm Threshold is changed successfully.  "
    info_chinese = "修改证书到期告警阈值成功。"
    response_body = generate_response_body(status_code, info, info_chinese,
                                           data)
    return response_body


@api_view(['GET', 'POST'])
@utils.check_if_user_login()
@utils.check_is_admin_user()
def cert_time_configuration(request):
    """
    功能描述：处理超时配置查询和配置请求
    参数：request
    返回值：Response
    异常描述：
    修改记录：首次开发
    """
    user = request.user
    if request.method == 'GET':
        cert_time = query_cert_time_configuration()
        # 构建响应体
        data = {'cert_time': cert_time}
        status_code = STATUS_SUCCESS
        info = "The Certificate Expiry Alarm Threshold " \
               "has been obtained successfully."
        info_chinese = u"获取证书到期告警阈值成功。"
        response_body = generate_response_body(status_code, info, info_chinese,
                                               data)
        return Response(response_body, status=status.HTTP_200_OK)
    # 配置分支
    if request.method == 'POST':
        cert_time = request.data.get("cert_time", "")
        if check_cert_time(cert_time=cert_time):
            response_body = update_cert_time_configuration(cert_time)
            read_cert_info('cert', CERT_PATH)
            operation_log(user=user, state="SUCCESS")
        else:
            data = {"cert_time": cert_time}
            status_code = STATUS_FAILED
            info = "Failed to set the Certificate Expiry Alarm Threshold."
            info_chinese = u"设置证书到期告警阈值失败。"
            response_body = generate_response_body(status_code,
                                                   info,
                                                   info_chinese,
                                                   data)
            operation_log(user=user, state="FAIL")
        return Response(response_body, status=status.HTTP_200_OK)
def encrypt_password(password):
    """
    加密保存私钥密码
    :param password: 密码
    :return: 是否成功
    """
    cso = CDLL(CRYPTO_SO_PATH)
    if not os.path.exists(NEW_PATH):
        os.mkdir(NEW_PATH)
    result = cso.EncryptKronos(password.encode(),
                               len(password),
                               WEBUI_PATH.encode())
    if result != 0:
        return False
    return True


def build_subj(data):
    """
    生成subj参数
    :param data: 请求数据
    :return: subj的字符串格式
    """
    item_list = []
    for field in SUBJ_FIELD_MAP:
        value = data.get(field)
        if value:
            item_list.append("{}={}".format(SUBJ_FIELD_MAP[field], value))
    return "".join(item_list)


@api_view(["POST"])
@utils.check_if_user_login()
@utils.check_is_admin_user()
def generate_csr(request):
    """
    生成CSR文件
    :param request: 请求
    :return: 响应
    """
    form = CsrForm(request.data)
    if not form.is_valid():
        info = 'The input parameter is incorrect.'
        info_chinese = "输入参数错误。"
        save_operation_log(request.user, GENERATE_CSR_EVENT, STATUS_FAILED,
                           ("Failed to generate the CSR file.", 95))
        return FailResponse(info, info_chinese)
    password = generate_password(PASSWORD_BYTE_LEN)
    if not encrypt_password(password):
        LOGGER.warning("Failed to encrypt the password.")
        save_operation_log(request.user, GENERATE_CSR_EVENT, STATUS_FAILED,
                           ("Failed to generate the CSR file.", 95))
        return FailResponse("Failed to generate the password.", "密码生成失败。")
    proc = subprocess.run(["openssl", "genrsa", "-aes256", "-passout",
                           "pass:{}".format(password), "-out", "cert.key",
                           "4096"], capture_output=True, cwd=NEW_PATH)
    if proc.returncode:
        save_operation_log(request.user, GENERATE_CSR_EVENT, STATUS_FAILED,
                           ("Failed to generate the CSR file.", 95))
        LOGGER.error("Failed to generate the private key. Expect: %s",
                     proc.stderr.decode())
        return FailResponse("Failed to generate the private key.",
                            "生成私钥失败。")
    os.chmod(os.path.join(NEW_PATH, "cert.key"), stat.S_IWUSR | stat.S_IRUSR)
    proc = subprocess.run(["openssl", "req", "-new", "-key", "cert.key",
                           "-passin", "pass:{}".format(password), "-subj",
                           build_subj(request.data)], capture_output=True,
                          cwd=NEW_PATH)
    if proc.returncode:
        save_operation_log(request.user, GENERATE_CSR_EVENT, STATUS_FAILED,
                           ("Failed to generate the CSR file.", 95))
        LOGGER.error("Generate CSR file fail. Expect: %s", proc.stderr.decode())
        return FailResponse("Failed to generate the CSR file.",
                            "生成CSR文件失败。")
    del password
    data = {
        "content": proc.stdout
    }
    info = "The CSR file is generated successfully."
    info_chinese = "生成CSR文件成功。"
    save_operation_log(request.user, GENERATE_CSR_EVENT, STATUS_SUCCESS,
                       ("The CSR file is generated successfully.", 94))
    return SuccessResponse(info, info_chinese, data)


def validate_cert_file(request):
    """
    读取并验证证书文件
    :param request: 请求
    :return: 证书文件
    """
    tmp_path = os.path.join(KitConfig.tool_root_dir, "tmp")
    if not os.path.exists(tmp_path):
        os.mkdir(tmp_path)
    type_limit = (".pem", ".crt", ".cer")
    request.session["type_limit"] = type_limit
    request.session["file_path"] = NEW_PATH
    cert_path = os.path.join(NEW_PATH, "cert.pem")
    if os.path.exists(cert_path):
        os.remove(cert_path)
    file = request.FILES.get("file")
    if not file:
        raise Exception("File require")
    request.session.pop("type_limit")
    request.session.pop("file_path")
    if file.size > 1024 * 1024:
        raise Exception("File size error")
    if os.path.splitext(file.name)[-1] not in type_limit:
        raise Exception("File type error")
    return file


def check_cert_privatekey_match(key_file_path, cert_file_path):
    """
    证书与私钥匹配检查
    :param key_file_path: 私钥路径
    :param cert_file_path: 证书文件路径
    :return: 是否匹配
    """
    try:
        cso = CDLL(CRYPTO_SO_PATH)
        match_result = cso.VerifyKronos(WEBUI_PATH.encode(),
                                        key_file_path.encode(),
                                        cert_file_path.encode())
        if match_result != 0:
            LOGGER.error("The private key does not match the certificate.")
            return False
    except Exception as error:
        LOGGER.error("The private key does not match the certificate. "
                     "Except:%s.", error)
        raise OSError("The private key does not match the certificate.")
    return True


def check_cert_enddate(cert_file_name):
    """
    检测证书有效期
    :param cert_file_name: 证书文件路径
    :return: 是否有效
    """
    proc = subprocess.run(["openssl", "x509", "-in", cert_file_name,
                           "-noout", "-enddate"], capture_output=True)
    out = proc.stdout.decode()
    end_date_str = out.strip().split("=")[-1]
    end_date = datetime.strptime(end_date_str, CTIME_FT)
    if end_date <= datetime.utcnow():
        return False
    return True


def check_cert_key_len(cert_file_path):
    """
    密钥长度检查
    :param cert_file_path: 证书文件路径
    :return: 公钥长度是否大于2048
    """
    try:
        cmd_pubkey_before = ['openssl', 'x509', '-in', cert_file_path,
                             '-noout', '-text']
        cmd_pubkey_after = ['grep', 'Public-Key']

        cmd_pubkey_before_result = subprocess.Popen(cmd_pubkey_before,
                                                    stdout=subprocess.PIPE,
                                                    stderr=subprocess.PIPE,
                                                    shell=False)
        cmd_pubkey_result = \
            subprocess.Popen(cmd_pubkey_after,
                             stdin=cmd_pubkey_before_result.stdout,
                             stdout=subprocess.PIPE,
                             stderr=subprocess.PIPE,
                             shell=False)
        public_key = cmd_pubkey_result.stdout.read().strip()

        _, key_size_str = public_key.split('('.encode())
        key_size_str, _ = key_size_str.split(' '.encode())
        key_size = int(key_size_str)

        if key_size < PUBLIC_KEY_MIN_LEN:
            return False
    except Exception as error:
        LOGGER.error("Failed to check the key length. Except:%s.", error)
        raise OSError("Failed to check the key length.")
    return True


def validate_cert(cert_file_name, key_file_name):
    """
    验证证书信息
    :param cert_file_name: 证书文件路径
    :param key_file_name: 私钥文件路径
    :return: （响应，是否成功）
    """
    if not os.path.exists(os.path.join(NEW_PATH, "cert.key")):
        LOGGER.info("The private key does not exist.")
        info = "The private key does not exist."
        info_chinese = "私钥不存在。"
        return FailResponse(info, info_chinese), False
    try:
        ret = check_cert_enddate(cert_file_name)
    except (IndexError, ValueError) as error:
        info = "Failed to obtain the certificate information."
        info_chinese = "获取证书信息失败。"
        LOGGER.info(info + " Except:%s.", error)
        return FailResponse(info, info_chinese), False
    if not ret:
        info = "The certificate has expired."
        info_chinese = "证书已过期。"
        LOGGER.info(info)
        return FailResponse(info, info_chinese), False
    if not check_cert_privatekey_match(key_file_name, cert_file_name):
        info = "The private key does not match the certificate."
        info_chinese = "证书和私钥不匹配。"
        LOGGER.info(info)
        return FailResponse(info, info_chinese), False
    try:
        ret = check_cert_key_len(cert_file_name)
    except OSError as os_error:
        LOGGER.info("Failed to obtain the certificate information. Except:%s",
                    os_error)
        return FailResponse("Failed to obtain the certificate information.",
                            "获取证书信息失败。"), False
    if not ret:
        info = "The certificate private key is less than 2048 bits."
        info_chinese = "证书密钥长度小于2048 bits。"
        LOGGER.info(info)
        return FailResponse(info, info_chinese), False
    return None, True


@api_view(["POST"])
@utils.check_if_user_login()
@utils.check_is_admin_user()
def upload_cert(request):
    """
    证书导入
    :param request: 请求
    :return: 响应
    """
    try:
        cert_file = validate_cert_file(request)
    except Exception as exc:
        LOGGER.info("The file verification failed. Expect: %s", exc)
        save_operation_log(request.user, EXPORT_CERT_EVENT, STATUS_FAILED,
                           ("Failed to import the web server certificate.", 97))
        info = "Failed to import the web server certificate. Please try again."
        info_chinese = "web服务端证书导入失败，请重新导入。"
        return FailResponse(info, info_chinese)
    cert_file_name = os.path.join(NEW_PATH, "cert.pem")
    key_file_name = os.path.join(NEW_PATH, "cert.key")
    with open(cert_file_name, "wb+") as fp:
        fp.write(cert_file.read())
    os.chmod(cert_file_name, stat.S_IWUSR | stat.S_IRUSR)
    resp, ok = validate_cert(cert_file_name, key_file_name)
    if not ok:
        save_operation_log(request.user, EXPORT_CERT_EVENT, STATUS_FAILED,
                           ("Failed to import the web server certificate.", 97))
        return resp
    info = ("The web server certificate is successfully imported "
            "and will take effect after the software is reset.")
    info_chinese = "web服务端证书导入成功，复位软件后即可生效。"
    save_operation_log(request.user, EXPORT_CERT_EVENT, STATUS_SUCCESS,
                       ("The web server certificate is successfully imported",
                        96))
    return SuccessResponse(info, info_chinese)


def is_redhat6():
    """
    是否redhat6系列系统
    """
    release_file = "/etc/redhat-release"
    if os.path.exists(release_file):
        with open(release_file, "r") as fp:
            redhat_release = fp.read().strip()
        if re.match(r"CentOS.*release 6\.", redhat_release) or \
                re.match(r"Red Hat.*release 6\.", redhat_release):
            return True
    return False


def is_docker():
    """
    判断是否docker环境
    """
    docker_env = "/.dockerenv"
    return os.path.exists(docker_env)


def stop_and_start_nginx(docker=False):
    """
    停止并启动Nginx
    redhat6和docker使用reload证书不生效
    """
    if docker:
        start_cmd = DOCKER_START_NGINX_CMD
        stop_cmd = DOCKER_STOP_NGINX_CMD
    else:
        start_cmd = SERVICE_START_CMD
        stop_cmd = SERVICE_STOP_CMD
    stop_proc = subprocess.run(stop_cmd, capture_output=True)
    if stop_proc.returncode:
        return stop_proc
    start_proc = subprocess.run(start_cmd, capture_output=True)
    return start_proc


def restart_nginx():
    """
    重启nginx
    :return: 是否成功
    """
    try:
        if is_docker():
            proc = stop_and_start_nginx(docker=True)
        elif is_redhat6():
            proc = stop_and_start_nginx()
        else:
            proc = subprocess.run(SYSTEMD_RESTART_CMD, capture_output=True)
        if proc.returncode:
            LOGGER.error(
                "Failed to restart Nginx. Expect: %s", proc.stderr.decode())
        return proc
    except Exception as err:
        LOGGER.error("Failed to restart Nginx. Expect: %s", err)
        return None


def nginx_status():
    """
    查看nginx状态
    :return: Nginx是否正常运行
    """
    status_rst = subprocess.run(NGINX_STATUS_CMD, capture_output=True)
    if "is running" in status_rst.stdout.decode():
        return True
    return False


def backup_cert():
    """
    备份证书和工作密钥
    将证书备份到BACKUP_PATH，将工作密钥备份到NGINX_BAK_PATH
    """
    if not os.path.exists(WEBUI_KEY_PATH) or not os.path.exists(WEBUI_PEM_PATH):
        return
    if os.path.exists(NGINX_BAK_PATH):
        safe_delete_folder(NGINX_BAK_PATH)
    if os.path.exists(NGINX_PATH):
        os.rename(NGINX_PATH, NGINX_BAK_PATH)
    if not os.path.exists(BACKUP_PATH):
        os.mkdir(BACKUP_PATH, 0o700)
    os.rename(WEBUI_KEY_PATH, BACKUP_KEY_PATH)
    os.rename(WEBUI_PEM_PATH, BACKUP_PEM_PATH)
    return True


def rollback_cert():
    """
    恢复证书和工作密钥
    将工作密钥NGINX_PATH改为原来的KRONOS_PATH
    将备份的工作密钥恢复NGINX_BAK_PATH > NGINX_PATH
    将私钥和证书文件改为原来的NEW_PATH
    将备份的私钥证书文件恢复BACKUP_PATH > WEBUI_PATH
    """
    if not os.path.exists(KRONOS_PATH):
        safe_delete_folder(KRONOS_PATH)
    if not os.path.exists(NGINX_PATH) or not os.path.exists(NGINX_BAK_PATH):
        return
    os.rename(NGINX_PATH, KRONOS_PATH)
    os.rename(NGINX_BAK_PATH, NGINX_PATH)
    if os.path.exists(WEBUI_KEY_PATH):
        os.rename(WEBUI_KEY_PATH, NEW_KEY_PATH)
    if os.path.exists(WEBUI_PEM_PATH):
        os.rename(WEBUI_PEM_PATH, NEW_PEM_PATH)
    if os.path.exists(BACKUP_KEY_PATH):
        os.rename(BACKUP_KEY_PATH, WEBUI_KEY_PATH)
    if os.path.exists(BACKUP_PEM_PATH):
        os.rename(BACKUP_PEM_PATH, WEBUI_PEM_PATH)


def replace_cert():
    """
    替换证书和工作密钥
    新的工作密钥在KRONOS_PATH中，新的key和pem文件在NEW_PATH中
    将新的工作密钥改为NGINX_PATH，将新的证书替换到WEBUI_PATH
    """
    if not os.path.exists(NEW_PATH) or not os.path.exists(KRONOS_PATH):
        return
    if not os.path.exists(NEW_KEY_PATH) or not os.path.exists(NEW_PEM_PATH):
        return
    if not backup_cert():
        return
    if os.path.exists(NGINX_PATH):
        safe_delete_folder(NGINX_PATH)
    os.rename(KRONOS_PATH, NGINX_PATH)
    os.mkdir(KRONOS_PATH, 0o700)
    os.rename(NEW_KEY_PATH, WEBUI_KEY_PATH)
    os.rename(NEW_PEM_PATH, WEBUI_PEM_PATH)
    return True


@api_view(["POST"])
@utils.check_if_user_login()
@utils.check_is_admin_user()
def reload_nginx(request):
    """
    重启Nginx
    :param request: 请求
    :return: 响应
    """
    # 证书不存在，不用重启
    if not replace_cert():
        info = "The web server certificate does not exist."
        info_chinese = "web服务端证书不存在。"
        return FailResponse(info, info_chinese)
    proc = restart_nginx()
    if proc and not proc.returncode and nginx_status():
        safe_delete_folder(BACKUP_PATH)
        safe_delete_folder(NGINX_BAK_PATH)
        read_cert_info('cert', CERT_PATH)
        info = "Nginx is restarted successfully."
        info_chinese = "Nginx重启成功"
        save_operation_log(request.user, RESTART_NGINX_EVENT, STATUS_SUCCESS,
                           ("Nginx is restarted successfully.", 98))
        return SuccessResponse(info, info_chinese)
    info = "Failed to restart Nginx."
    info_chinese = "Nginx重启失败"
    no_tty_err = "you must have a tty to run sudo"
    if proc and proc.returncode and no_tty_err in proc.stderr.decode():
        info = "Failed to restart Nginx. Please comment out the Defaults "\
               "requiretty configuration in the /etc/sudoers "\
               "file and try again."
        info_chinese = "重启Nginx失败，请注释/etc/sudoers文件中"\
                       '的"Defaults requiretty"配置后重试。'
    rollback_cert()
    restart_nginx()
    save_operation_log(request.user, RESTART_NGINX_EVENT, STATUS_FAILED,
                       (info, 99))
    return FailResponse(info, info_chinese)


def get_all_file_path(dirname):
    """
    获取文件夹下所有文件路径
    :param dirname: 文件夹路径
    :return: 文件列表
    """
    result = []
    for maindir, _, file_name_list in os.walk(dirname):
        for filename in file_name_list:
            apath = os.path.join(maindir, filename)
            result.append(apath)
    return result


def safe_delete_folder(folder_path):
    """
    安全删除文件夹
    :param folder_path: 文件夹路径
    :return:
    """
    if not os.path.exists(folder_path):
        return

    if not os.listdir(folder_path):
        os.rmdir(folder_path)
        return

    result = get_all_file_path(folder_path)
    for file_path in result:
        try:
            safely_delete_file(file_path)
        except Exception as error:
            LOGGER.error("Failed to safely delete the file. Except:%s", error)

    if not os.listdir(folder_path):
        os.rmdir(folder_path)


def safely_delete_file(file_path):
    """git
    :param file_path: file path
    0, 1, secure random number cover three times
    """
    if not os.path.exists(file_path):
        return
    delete_cmd = [DELETE_FILE_SCRIPT, INSTALL_PATH, file_path]
    subprocess.run(delete_cmd, capture_output=True)


@api_view(["GET"])
@utils.check_if_user_login()
@utils.check_is_admin_user()
def ca_cert(request):
    info = "Failed to obtain the root certificate."
    info_chinese = "获取根证书失败。"
    if not os.path.exists(WEBUI_ROOT_CERT_PATH):
        LOGGER.info("Failed to obtain the root certificate. Except: %s "
                    "does not exist.",
                    WEBUI_ROOT_CERT_PATH)
        return FailResponse(info, info_chinese)
    if not os.access(WEBUI_ROOT_CERT_PATH, os.R_OK):
        LOGGER.info("Failed to obtain the root certificate. Except: %s "
                    "Permission denied", WEBUI_ROOT_CERT_PATH)
        return FailResponse(info, info_chinese)
    info = "Root certificate obtained successfully."
    info_chinese = "获取根证书成功。"
    data = {"content": open(WEBUI_ROOT_CERT_PATH).read()}
    return SuccessResponse(info, info_chinese, data)

```

