```
#! /usr/bin/env python
# -*- coding: utf-8 -*-
'''
Copyright: Copyright (c) Huawei Technologies Co., Ltd. 2020-2020.
All rights reserved.
Create: 2020-06-03
Content: 证书查看
'''
import os
import time
import logging
import threading
import subprocess
from datetime import datetime
from datetime import timedelta

from django.db import IntegrityError
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response

from cert.models import Cert
from tool.logger.logger import Logger
from tool.util.common_util import CommonUtil
from timeout_configuration.models import SystemConfiguration
from util.response import SuccessResponse, FailResponse, generate_response_body
from util import common_util as utils
from tool.kit_config import KitConfig

if KitConfig.tool_name == KitConfig.DEPENDENCY:
    from log_manager import common
else:
    from resource.utils import insert_operationlog


class RangeException(Exception):
    def __init__(self, errorvalue):
        self.value = errorvalue

    def __str__(self):
        return self.value


class Range:
    def __init__(self, value):
        if not (_MIN_ALARM_CERT_TIME <= value <= _MAX_ALARM_CERT_TIME):
            raise RangeException("Out of range")


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


def start_cert_reader():
    """
    起线程检查证书有效性并记录
    """
    try:
        cert_thread = threading.Thread(target=update_cert_info, args=())
        cert_thread.setDaemon(True)
        cert_thread.start()
    except SystemError:
        LOGGER.error("start cert thread failed.")


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
    except Cert.DoesNotExist:
        cert = Cert(cert_name=name)
    cert.cert_expired = exp_time
    cert.cert_flag = state
    try:
        cert.save()
    except IntegrityError:
        LOGGER.error("cert exists")


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
    except Exception:
        err = "Parse the certificate file failed by command"
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
    except Exception:
        err = "Parse certificate expire time failed"
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
    except Cert.DoesNotExist:
        info = "Failed to read cert information."
        info_chinese = u"读取证书信息失败。"
        LOGGER.error(info)
        return FailResponse(info, info_chinese)

    info = "Read cert successfully. cert_flag:{}, cert_expired" \
           ":{}".format(cert.cert_flag,
                        cert.cert_expired.strftime('%Y-%m-%d %H:%M:%S %f'))
    info_chinese = u"读取证书信息成功。"
    data = {"cert_flag": cert.cert_flag, "cert_expired": cert.cert_expired}
    LOGGER.info(info)
    return SuccessResponse(info, info_chinese, data)


def operation_log(user, state):
    if state == "SUCCESS":
        if KitConfig.tool_name == KitConfig.DEPENDENCY:
            user_id = user.get_id()
            log_event = common.OpEvent.CONFIGURATE_ALARM_TIME
            log_mes = "The certificate expiration alarm time" \
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
            log_mes = "The certificate expiration alarm time" \
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
            LOGGER.error(error)
            config_value = _DEFAULT_ALARM_CERT_TIME
            system_configure.config_value = _DEFAULT_ALARM_CERT_TIME
        except RangeException as err:
            LOGGER.error(err)
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
    info = "The certificate expiration alarm time is changed successfully.  "
    info_chinese = "修改证书到期告警时间成功。"
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
        info = "The certificate expiration alarm time " \
               "has been obtained successfully."
        info_chinese = u"获取证书到期告警超时时间成功。"
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
            info = "Failed to set the certificate expiration alarm time."
            info_chinese = u"设置证书到期告警时间失败。"
            response_body = generate_response_body(status_code,
                                                   info,
                                                   info_chinese,
                                                   data)
            operation_log(user=user, state="FAIL")
        return Response(response_body, status=status.HTTP_200_OK)
"""证书Model"""
from django.db import models

# Create your models here.
class Cert(models.Model):
    STATE_VALID = 1  # 证书有效
    STATE_EXPIRING = 0  # 证书即将过期
    STATE_EXPIRED = -1  # 证书已过期
    cert_name = models.CharField(max_length=255, unique=True)  # 证书名称
    cert_expired = models.DateTimeField()  # 证书过期时间
    cert_flag = models.CharField(max_length=2)  # 证书状态: 1未到期,0即将到期,-1过期












#! /usr/bin/env python

# -*- coding: utf-8 -*-
'''
Copyright: Copyright (c) Huawei Technologies Co., Ltd. 2020-2020.
All rights reserved.
Create: 2020-07-15
Content: 超时配置表
'''
from django.db import models


class SystemConfiguration(models.Model):
    """
    功能描述：系统配置表:timeout超时时间，cert_time证书到期告警时间，version工具引入的版本
    接口：
    修改记录：
    """
    CONFIGURE_CHOICES = ((0, "timeout"),
                         (1, "cert_time"))

    config_name = models.CharField(max_length=255, choices=CONFIGURE_CHOICES,
                                   unique=True)
    config_value = models.CharField(max_length=255)
    # 2.2.1版本引入该字段
    version = models.fields.CharField(max_length=10, default="2.2.1")
```

