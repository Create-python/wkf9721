```python
"""解压文件方法"""

import os
import zipfile
import tarfile
import shutil
import subprocess
import re

from util import common_util

MAX_SIZE = 1024 * 1024 * 1024

SUCCESS_CODE = 0
FAIL_CODE = 1

RESPONSE_INFO = {
    "NOT_VALID_FILE": [FAIL_CODE, "Not a valid compressed file",
                       "不是有效的压缩文件"],
    "FILE_ALREADY_EXIST": [FAIL_CODE,
                           "Decompression failed. The folder in the "
                           "compressed package already exists on the server.",
                           "解压失败，压缩文件中的目录在服务器已存在"],
    "FILE_TOO_LARGE": [FAIL_CODE, "Unzip file too large", "解压后文件超过限值"],
    "UNZIP_SUCCESS": [SUCCESS_CODE, "Unzip success", "解压成功"],
    "UNZIP_EXCEPTION": [FAIL_CODE, "Unzip exception", "解压异常"],
    "CANT_FIND_FILE": [FAIL_CODE, "No folder is found in the package",
                       "压缩包内未找到文件夹"],
    "TOO_MANY_FILES": [FAIL_CODE,
                       "The number of folders after decompression is "
                       "greater than 1",
                       "解压后的文件夹数量大于1"],
    "DECOMPRESSION_FAILED": [FAIL_CODE,
                             "Failed to decompress the package because the "
                             "package contains files in incorrect paths.",
                             "解压失败，压缩包中存在异常路径文件。"]
}
CHOICE = {
    'OVERRIDE': 'override',
    'NORMAL': 'normal',
    'SAVE_AS': 'save_as',
}
TAR_BZ_SUFFIX = (".tar.bz", ".tar.bz2", ".tbz", ".tbz2")


class UnzipTool:
    """解压文件类"""

    def __init__(self, path, file_name):
        self.path = path  # 文件路径
        self.file_name = file_name  # 文件名
        self.file_prefix, self.file_suffix = self.get_file_prefix_suffix()
        self.file_path = os.path.join(self.path, self.file_name)
        self.unzip_path = None

    def unzip_path_func(self):
        """获取包解压后的路径"""
        return self.unzip_path

    def file_path_func(self):
        """获取包的路径"""
        return self.file_path

    def get_file_prefix_suffix(self):
        """获取包名的的前缀和后缀"""
        file_prefix, file_suffix = \
            common_util.get_prefix_suffix(self.file_name)
        return file_prefix, file_suffix

    def clean_unzip_file(self):
        """解压失败时，清理解压出的文件夹"""
        if self.unzip_path and os.path.exists(self.unzip_path):
            shutil.rmtree(self.unzip_path)

    @staticmethod
    def calc_space(path):
        """计算解压空间"""
        size = subprocess.Popen(['du', '-s', path],
                                stdout=subprocess.PIPE).communicate()
        return float(re.findall(r'[.\d]+', str(size))[0])

    def is_normal_file(self, file_path, temp_path):
        """
        判断压缩文件中的文件是否带有相对路径
        :param file_path: 文件路径
        :param temp_path: 临时解压的路径
        :return: 响应
        """
        # /../; ../; /..\; \../; \..\;
        pattern = re.compile(r"\/\.\.\/|^\.\.\/|\/\.\.\\|\\\.\.\/|\\\.\.\\")
        result = pattern.search(file_path)
        if not result:
            return True, ""
        self.clean_unzip_file()
        response = RESPONSE_INFO["DECOMPRESSION_FAILED"].copy()
        response.append({})
        if os.path.exists(temp_path):
            shutil.rmtree(temp_path)
        return False, response

    def unzip(self, file_name=None, choice=None):
        """
        解压zip, rar, war, jar包
        :return: [SUCCESS_CODE, "Unzip success", "解压成功"]
        """
        try:
            is_zip_file = zipfile.is_zipfile(self.file_path)
            current_size = 0
            srcfile = zipfile.ZipFile(self.file_path, "r")
            temp_path = self.make_temp_file()
            if not is_zip_file:
                shutil.rmtree(temp_path)
                return RESPONSE_INFO.get("NOT_VALID_FILE")
            for info in srcfile.infolist():
                flag, result = self.is_normal_file(info.filename, temp_path)
                if not flag:
                    return result
                res, response = self.is_large_file(info.file_size,
                                                   current_size)
                if res:
                    shutil.rmtree(temp_path)
                    return response
                srcfile.extract(info.filename, temp_path)
            return self.check_untar(temp_path, file_name, choice)
        except Exception as exp:
            self.clean_unzip_file()
            RESPONSE_INFO["UNZIP_EXCEPTION"][1] = str(exp)
            response = RESPONSE_INFO["UNZIP_EXCEPTION"].copy()
            response.append({})
            return response

    def make_temp_file(self):
        """
        :return: 临时解压的路径
        """
        os.chdir(self.path)
        temp_path = os.path.join(self.path, "temp_tar")
        if os.path.exists(temp_path):
            shutil.rmtree(temp_path)
        os.mkdir(temp_path)
        return temp_path

    def untar(self, file_name=None, choice=None):
        """
        解压tar, tar.gz, gz, tar.bz2, bz2, bz包
        :return: [SUCCESS_CODE, "Unzip success", "解压成功"]
        """
        try:
            temp_path = self.make_temp_file()
            is_whitelist_migration = False
            is_tar_file = tarfile.is_tarfile(self.file_path)
            if not is_tar_file:
                if self.file_suffix in TAR_BZ_SUFFIX:
                    return self.untar_bz2(self.file_path, temp_path, file_name,
                                          choice)
                else:
                    response = RESPONSE_INFO["NOT_VALID_FILE"].copy()
                    response.append({})
                    shutil.rmtree(temp_path)
                    return response
            current_size = 0
            srcfile = tarfile.open(self.file_path)
            first_dir_path = list(srcfile.getmembers())[0].name
            self.unzip_path = os.path.join(self.path, first_dir_path)
            if (choice and choice != CHOICE['SAVE_AS']) and os.path.exists(
                    self.unzip_path):
                shutil.rmtree(temp_path)
                response = RESPONSE_INFO["FILE_ALREADY_EXIST"].copy()
                response.append({})
                return response
            is_whitelist_migration = self.check_whilelist_migration()
            if is_whitelist_migration:
                temp_path = self.path
            for info in srcfile.getmembers():
                flag, result = self.is_normal_file(info.name, temp_path)
                if not flag:
                    return result
                res, response = self.is_large_file(info.size, current_size)
                if res:
                    return response
                srcfile.extract(info.name, temp_path)
            if is_whitelist_migration:
                response = RESPONSE_INFO["UNZIP_SUCCESS"].copy()
                response.append(file_name)
                return response
            return self.check_untar(temp_path, file_name, choice)
        except Exception as exp:
            self.clean_unzip_file()
            RESPONSE_INFO["UNZIP_EXCEPTION"][1] = str(exp)
            response = RESPONSE_INFO["UNZIP_EXCEPTION"].copy()
            response.append({})
            return response

    def is_large_file(self, size, current_size):
        current_size += size
        if current_size < MAX_SIZE:
            return False, ""
        self.clean_unzip_file()
        response = RESPONSE_INFO["FILE_TOO_LARGE"].copy()
        response.append({})
        return True, response

    def check_whilelist_migration(self):
        if (self.file_name.startswith('Whitelist') and self.file_name.
                endswith('.tar.gz')) or \
                (self.file_name.startswith('Migration-package')
                 and self.file_name.endswith('.tar.gz')):
            return True
        else:
            return False

    def untar_bz2(self, file_path, temp_path, file_name=None, choice=None):
        os.chdir(self.path)
        cmd = ["tar", "-tf", file_path]
        sub_pro = subprocess.Popen(cmd, stdout=subprocess.PIPE,
                                   stderr=subprocess.PIPE, shell=False)
        value, _ = sub_pro.communicate()
        if sub_pro.returncode != 0:
            response = RESPONSE_INFO["UNZIP_EXCEPTION"].copy()
            response.append({})
            return response
        flag, result = self.is_normal_file(value.decode(), temp_path)
        if not flag:
            return result
        sp = subprocess.Popen(["tar", "-jxf", file_path, "-C", temp_path],
                              shell=False)
        while True:
            used = self.calc_space(temp_path) * 1024
            if used > MAX_SIZE:
                sp.terminate()
                if os.path.exists(temp_path):
                    shutil.rmtree(temp_path)
                response = RESPONSE_INFO["FILE_TOO_LARGE"].copy()
                response.append({})
                return response
            if sp.poll() is not None:
                return self.check_untar(temp_path, file_name, choice)

    def check_untar(self, temp_path, file_name=None, choice=None):

        file_list = os.listdir(temp_path)
        if len(file_list) != 1:
            if os.path.exists(temp_path):
                shutil.rmtree(temp_path)
            if not file_list:
                response = RESPONSE_INFO["CANT_FIND_FILE"].copy()
            else:
                response = RESPONSE_INFO["TOO_MANY_FILES"].copy()
            response.append({})
            return response
        self.unzip_path = os.path.join(self.path, file_list[0])
        if os.path.isfile(os.path.join(temp_path, file_list[0])):
            shutil.rmtree(temp_path)
            response = RESPONSE_INFO["CANT_FIND_FILE"].copy()
            response.append({})
            return response
        if choice and choice != CHOICE['SAVE_AS'] and os.path.exists(
                self.unzip_path):
            shutil.rmtree(temp_path)
            response = RESPONSE_INFO["FILE_ALREADY_EXIST"].copy()
            response.append({})
            return response

        if file_name:
            file_name, file_suffix = common_util.get_prefix_suffix(file_name)
            old_name_path = os.path.join(temp_path, file_list[0])
            new_name_path = os.path.join(temp_path, file_name)
            os.rename(old_name_path, new_name_path)
        else:
            file_name = file_list[0]
        new_path = os.path.join(self.path, file_name)
        shutil.copytree(os.path.join(temp_path, file_name), new_path,
                        symlinks=True, ignore_dangling_symlinks=True)
        shutil.rmtree(temp_path)
        response = RESPONSE_INFO.get("UNZIP_SUCCESS").copy()
        response.append(file_name)
        return response

```

