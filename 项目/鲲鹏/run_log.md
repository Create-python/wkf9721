```python
#!/usr/local/python3
# -*- encoding: utf-8 -*-
import configparser
import logging
import os
import re
import threading
import time
import shutil
import zipfile
from wsgiref.util import FileWrapper

from rest_framework.decorators import api_view
from django.http import FileResponse, StreamingHttpResponse, HttpResponse

from tool.logger.logger import Logger
from tool.util.common_util import CommonUtil
from util.response import SuccessResponse, FailResponse
from util import common_util


LOGGER = common_util.get_logger()

STATUS_SUCCESS = 0
STATUS_FAILED = 1
ADMIN_USER_ID = 1
MAX_COUNT = 99
LOG_DIR, LOG_FILE = os.path.split(Logger.log_output_path)

run_log_level = ""


def get_backup_count():
    cp = configparser.ConfigParser(
        defaults={'filename': '%s' % Logger.log_output_path})
    cp.read(Logger.log_config_path)
    section = cp["handler_fileHandler"]
    args = section.get("args")
    backup_count = min(int(args.split(",")[3].strip()), MAX_COUNT)
    return backup_count


def get_name_pattern_and_count(backup_count):
    try:
        if backup_count > 0:
            ret = r"^%s($|\.\d{1,%s}\.zip$)" \
                  % (LOG_FILE.replace(".", r"\."), len(str(backup_count)))
        else:
            ret = r"^%s$" % LOG_FILE.replace(".", r"\.")
        return ret, backup_count
    except Exception as e:
        raise ValueError("run log configuration error: %s" % e)


def file_condition(pattern, count, file_name):
    if re.match(pattern, file_name):
        match = re.search(r"\d+", file_name)
        if not match or int(match.group()) <= count:
            return True

    return False


@api_view(["GET"])
@common_util.check_if_user_login()
@common_util.check_is_admin_user()
def get_log_infos(_):
    """

    @param request:
    @return:
    """
    # 文件查询
    try:
        backup_count = get_backup_count()
        pattern, count = get_name_pattern_and_count(backup_count)
        file_list = [
            f for f in os.listdir(LOG_DIR) if file_condition(pattern, count, f)
        ]
    except (ValueError or FileNotFoundError):
        info = "Run log configuration error."
        info_chinese = "工作日志配置异常。"
        return FailResponse(info, info_chinese)

    # 构建响应体日志数据
    data = {'totalcount': len(file_list), 'loglist': file_list}

    info = "Run log query succeeded."
    info_chinese = u"查询运行日志成功。"

    return SuccessResponse(info, info_chinese, data)


@api_view(["GET"])
@common_util.check_if_user_login()
@common_util.check_is_admin_user()
def download(request):
    """

    @param request:
    @return:
    """
    file_name = request.GET.get("filename")
    if not file_name:
        file_name = LOG_FILE

    file_path = os.path.join(LOG_DIR, file_name)

    try:
        backup_count = get_backup_count()
        pattern, count = get_name_pattern_and_count(backup_count)
        if not file_condition(pattern, count, file_name) \
                or not os.path.isfile(file_path):
            info = "File not found"
            info_chinese = u"文件不存在"
            return FailResponse(info, info_chinese)
    except (ValueError or FileNotFoundError):
        info = "run log configuration error"
        info_chinese = "工作日志配置异常。"
        return FailResponse(info, info_chinese)

    wrapper = FileWrapper(open(file_path, "rb"))
    response = FileResponse(wrapper)
    response['Content-Type'] = 'application/octet-stream'
    response['Content-Length'] = os.path.getsize(file_path)
    response['Content-Disposition'] = 'attachment;filename="%s"' % file_name

    return response


def trans_log_level(level):
    level_map = {"ERROR": "err", "WARNING": "warn"}
    return level_map[level] if level in level_map else level.lower()


def run_log_level_sync_fun():
    global run_log_level
    while True:
        # 修改log配置文件
        log_config_path = Logger.log_config_path
        config = configparser.ConfigParser()
        config.read(log_config_path)
        level = config.get("logger_" + Logger.LOGGER_NAME, "level")
        if level != run_log_level:
            run_log_level = level
            Logger.reset_log_level(logging.getLogger(Logger.LOGGER_NAME),
                                   trans_log_level(run_log_level))

        time.sleep(1)


def start_log_level_sync():
    try:
        sync_thread = threading.Thread(target=run_log_level_sync_fun, args=())
        sync_thread.setDaemon(True)
        sync_thread.start()
    except SystemError:
        LOGGER.error("start log level sync thread failed")


@api_view(["GET"])
@common_util.check_if_user_login()
@common_util.check_is_admin_user()
def download_all_logs(_):
    """
    日志一键下载
    @param:
    @return:
    """
    log_level, log_dir = CommonUtil.get_log_config_from_file()
    file_name = "log.zip"
    install_path = CommonUtil.get_customize_path().get_tool_root_dir()
    file_path = os.path.join(install_path, file_name)
    # 删除上一次的压缩包
    if os.path.exists(file_path):
        os.remove(file_path)
    # 拷贝全汇编的日志文件到depadv/log或portadv/log目录下
    source_path = os.path.join(install_path, "tools/all_asm/logs")
    target_path = os.path.join(install_path, "log")
    os.mkdir(target_path)
    assembly_path = os.path.join(target_path, "assembly_logs")
    try:
        shutil.copytree(source_path, assembly_path)
    except FileNotFoundError:
        LOGGER.warning("Folder does not exist:%s" % source_path)
    # 拷贝make、cmake、预处理、服务器运行日志到log目录下
    copy_logs_files(log_dir, target_path)
    # 打包make、cmake、预处理、全汇编、服务器运行日志
    shutil.make_archive(target_path, "zip", target_path)
    # wrapper = FileWrapper(open(file_path, "rb"))
    # response = FileResponse(wrapper)
    # response['Content-Type'] = 'application/octet-stream'
    # response['Content-Length'] = os.path.getsize(file_path)
    # response['Content-Disposition'] = 'attachment;filename="%s"' % file_name
    # # 移除log目录  防止和下次内容有冲突
    # shutil.rmtree(target_path)
    # return response

    response = HttpResponse()
    response['Content_Type'] = 'application/octet-stream'
    response["Content-Disposition"] = 'attachment;filename="%s"' % file_name
    # response['Content-Length'] = os.path.getsize(file_path)
    response['X-Accel-Redirect'] = "/protected/%s" % file_name
    # 移除log目录  防止和下次内容有冲突
    shutil.rmtree(target_path)
    return response


def copy_logs_files(path, out):
    if os.listdir(path):
        for file in os.listdir(path):
            front_name = os.path.join(path, file)
            back_name = os.path.join(out, file)
            backup_count = get_backup_count()
            pattern, count = get_name_pattern_and_count(backup_count)
            file_name = os.path.basename(front_name)
            if os.path.isfile(front_name) and \
                    file_condition(pattern, count, file_name):
                shutil.copy(front_name, back_name)
    else:
        LOGGER.warning("The directory is empty: %s" % path)


def match_pattern(file_name):
    # 匹配makefile_parser.log或CmakePortingAdvisor.log或prevision.log
    pattern = r"(makefile_parser(|_bak)\.log)|" \
              r"(CmakePortingAdvisor\.log($|\.[12]\.zip$))|" \
              r"(prevision\.log($|\.[12]\.zip$))"
    if re.match(pattern, file_name):
        return True


def file_iterator(file_path, chunk_size=512):
    """
    文件生成器,防止文件过大，导致内存溢出
    :param file_path: 文件绝对路径
    :param chunk_size: 块大小
    :return: 生成器
    """
    with open(file_path, mode='rb') as f:
        while True:
            c = f.read(chunk_size)
            if c:
                yield c
            else:
                break


def send_zipfile(file_path, target_path, file_name):
    """
    Create a ZIP file on disk and transmit it in chunks of 8KB,
    without loading the whole file into memory. A similar approach can
    be used for large dynamic PDF files.
    """
    archive = zipfile.ZipFile(file_path, "w", zipfile.ZIP_DEFLATED)
    filelists = []
    for file in os.listdir(target_path):
        if os.path.isdir(target_path + '/' + file):
            filelists.append(target_path + '/' + filelists)
        else:
            filelists.append(target_path + '/' + file)

    for file in filelists:
        archive.write(file)
    archive.close()
    wrapper = FileWrapper(file_path)
    response = StreamingHttpResponse(wrapper, content_type='application/zip')
    response['Content-Disposition'] = 'attachment;filename="%s"' % file_name
    response['Content-Length'] = os.path.getsize(file_path)
    return response


def compress(get_files_path, set_files_path):
    f = zipfile.ZipFile(set_files_path , 'w', zipfile.ZIP_DEFLATED )
    for dirpath, dirnames, filenames in os.walk( get_files_path ):
        fpath = dirpath.replace(get_files_path, '')
        fpath = fpath and fpath + os.sep or ''
        for filename in filenames:
            f.write(os.path.join(dirpath,filename), fpath+filename)
    f.close()

```

