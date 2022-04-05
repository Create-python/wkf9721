```python
#!/usr/bin/env python3
# -*-coding:utf-8-*-
"""
@Usage: whitelist scanning
@Company: 2019, Huawei Technologies Co., Ltd. All rights reserved.
@Created: 08.02.2019
"""
import logging
import os
from collections import defaultdict
from distutils.version import StrictVersion

from tool.logger.logger import Logger
from tool.kit_config import KitConfig, OsConfig, BinaryScanResultType
from .whitelist_loader.whitelist_loader_factory import WhitelistLoaderType
from .whitelist_loader.whitelist_loader_factory import WhitelistLoaderFactory

LOGGER = logging.getLogger(Logger.LOGGER_NAME)


class WhiteList:
    """
    Usage: whitelist scanning
    """
    # whitelist level: 0, 1, 2, 3, 4, 6
    MIGRATION_LEVEL = 5
    def __init__(self):
        self.file_type = False
        self.result_level_index = -2
    def match_whitelist(self, lib_dict, os_name, os_version, file_type=False):
        """
        match lib_list and whitelist
        :param lib_dict: [(libname, libversion),]
        :param os_name:
        :param os_version:
        :param file_type: 扫描结果是否添加文件类型，二进制扫描需要，重打包不需要        :return:
        scan_report[
                    [libname, libversion, release_version, path,
                        os_name, os_version, level, url],
                    ....
                    [total_level, level_0, level_1, level_2, level_3,
                        level_4, level_5，level_6]
                    ]
        """
        LOGGER.info("Start match whitelist.")
        self.init_parameter(file_type)
        scan_report_dict = dict()
        if not self.scan_result_parameter_check(lib_dict, dict) or \
                not self.os_parameter_check(os_name, os_version):
            return scan_report_dict

        os_info = [os_name.lower(), os_version.lower()]

        # 遍历扫描结果字典
        for key in lib_dict.keys():
            so_list, exec_file_list, static_library_list, jar_list = \
                self._separate_list(lib_dict[key])

            # subscripts indicates number of each level and total number
            level_number = [0] * (self.MIGRATION_LEVEL + 2)
            scan_report = []
            if so_list:
                scan_report = self.SoMatch(self.file_type).match_result(
                    so_list, os_info, level_number)
            if exec_file_list:
                scan_report += self.ExecMatch(self.file_type).match_result(
                    exec_file_list, os_info, level_number)
            if static_library_list:
                scan_report += self.StaticMatch(self.file_type).match_result(
                    static_library_list, os_info, level_number)
            if jar_list:
                scan_report += self.JarMatch(self.file_type).match_result(
                    jar_list, os_info, level_number)
            if scan_report:
                scan_report.sort(key=lambda k: k[self.result_level_index])
                scan_report.append(self.level_process(level_number))
                scan_report_dict[key] = scan_report

        LOGGER.info("End matching whitelist, return scan report list.")
        return scan_report_dict

    def init_parameter(self, file_type):
        """
        初始化参数        :param file_type:
        :return:
        """
        self.file_type = file_type
        if file_type:
            self.result_level_index = -3
    @staticmethod
    def scan_result_parameter_check(scan_result, parameter_type):
        """
        检查扫描结果入参        :param scan_result: 扫描结果        :param parameter_type: scan_result类型        :return: True/False
        """
        if not scan_result:
            LOGGER.warning("Scan result is empty.")
            return False
        if not isinstance(scan_result, parameter_type):
            LOGGER.warning("Scan result is not correct type.")
            return False
        return True
    @staticmethod
    def os_parameter_check(os_name, os_version):
        """
        检查os入参        :param os_name: os名称        :param os_version: os版本        :return: True/False
        """
        if not os_name.strip() or not os_version.strip():
            LOGGER.warning("OS_name or OS_version is null.")
            return False
        return True
    @staticmethod
    def get_whitelist_name(res_list, input_os):
        """
        获取白名单名称        :param res_list: 扫描结果        :param input_os: os名称        :return:
        """
        is_jar_list = (res_list[0][-1] == BinaryScanResultType.JAR)
        # 加载jar包白名单
        if is_jar_list:
            if input_os not in OsConfig.target_porting_os:
                return ""
            # gcc版本>=4.8.5都可以用centos7.6的列表
            gcc_version = OsConfig.target_porting_os.get(input_os)["gcc"][3:]
            try:
                strict_gcc_version = StrictVersion(gcc_version)
            except ValueError as err:
                LOGGER.error("Invalid gcc version:%s. Except:%s.", gcc_version,
                             err)
                return ""
            if strict_gcc_version < StrictVersion("4.8.5"):
                return ""
            whitelist_file = 'maven_centos7.6.csv.cipher'
            return os.path.join(KitConfig.whitelist_dir, whitelist_file)
        # 加载os白名单
        whitelist_file = \
            OsConfig.target_porting_os.get(input_os).get('whitelist')
        return [os.path.join(KitConfig.whitelist_dir, item)
                for item in whitelist_file]

    @staticmethod
    def deal_so_url(url):
        """
        处理扫描结果内url列的数据        :param url: 扫描结果内url列的数据        :return: 处理后的url
        """
        if not url.startswith("http") and "/" in url:
            return url.split("/")[-1]
        return url

    @staticmethod
    def level_process(level_number):
        """
        扫描结果等级统计列表处理        :param level_number: 扫描结果等级统计列表        :return: 处理后的扫描结果等级统计列表        """
        total_number = sum(level_number)
        level_number.insert(0, total_number)
        return level_number

    @staticmethod
    def add_file_type(file_type_flag, match_result, file_type):
        """
        为匹配结果添加文件类型        :param file_type_flag: 是否添加文件类型标志        :param match_result: 匹配结果        :param file_type: 文件类型        :return:
        """
        if file_type_flag:
            match_result.append(file_type)
        return match_result

    @staticmethod
    def _separate_list(scan_report_list):
        """
        将扫描结果细分为动态库列表、二进制文件列表、静态库列表、jar包列表        :param scan_report_list: 扫描结果        :return: 动态库列表、二进制文件列表、静态库列表、jar包列表        """
        exec_file_list = []
        lib_list = []
        jar_list = []
        static_library_list = []
        for item in scan_report_list:
            if item[-1] == BinaryScanResultType.JAR:
                jar_list.append(item)
            elif item[-1] == BinaryScanResultType.EXEC:
                exec_file_list.append(item)
            elif item[-1] == BinaryScanResultType.STATIC_LIBRARY:
                static_library_list.append(item)
            else:
                lib_list.append(item)
        return lib_list, exec_file_list, static_library_list, jar_list

    class SoMatch:
        """ so匹配内嵌类 """
        def __init__(self, file_type):
            self._file_type = file_type
            self._os_info = list()
            self._so_whitelist_dict = dict()
            self._so_name_dict = dict()

        def match_result(self, so_list, os_info, level_number):
            """
            get so match result
            so_list: [(so_name, so_version, release_version,
                        path, architecture)]
            return: [(so_name, so_version, release_version, path,
                    os_name, os_version, level, url)]
            """
            scan_report_list = []
            self._os_info = os_info
            self._so_whitelist_dict, self._so_name_dict = \
                self._generate_so_whitelist_dict(so_list)
            for lib_list_item in so_list:
                if not lib_list_item[0]:
                    continue
                if lib_list_item[1]:
                    lib_temp = self._so_fuzzy_match_result(lib_list_item)
                else:
                    so_version = 'd'
                    lib_temp = \
                        self._so_full_match_result(lib_list_item, so_version)
                level_number[int(lib_temp[-2])] += 1
                if lib_temp[-2] in ('0', '1', '2'):
                    lib_temp = WhiteList.add_file_type(
                        self._file_type, lib_temp,
                        BinaryScanResultType.DYNAMIC_LIBRARY)
                else:
                    lib_temp = WhiteList.add_file_type(
                        self._file_type, lib_temp,
                        BinaryScanResultType.SOFTWARE)
                scan_report_list.append(lib_temp)
            LOGGER.info("End matching so_whitelist, return scan report list.")
            return scan_report_list

        def _so_full_match_result(self, so_list_item, so_version):
            """
            so libraries full match result
            """
            lib_temp = [so_list_item[0], so_version]
            lib_temp_tuple = (so_list_item[0], so_version)
            if lib_temp_tuple in self._so_whitelist_dict:
                level_temp = self._so_whitelist_dict.get(lib_temp_tuple)
            else:
                # 如果so文件架构为AArch64，不在白名单也认为其无需迁移
                if so_list_item[-1] == 'AArch64':
                    level_temp = ['0', '']
                else:
                    level_temp = [str(WhiteList.MIGRATION_LEVEL), '']
                    if lib_temp[1] == 'd':
                        # 匹配不到，报告呈现仍为原本版本号
                        lib_temp[1] = so_list_item[1]
            if level_temp[1]:
                level_temp[1] = WhiteList.deal_so_url(level_temp[1])
            lib_temp += \
                [so_list_item[2], so_list_item[3]] + self._os_info + level_temp
            return lib_temp

        def _so_fuzzy_match_result(self, so_list_item):
            """
            if so_version is not None, need to fuzzy match whitelist
            """
            so_item = (so_list_item[0], so_list_item[1])
            if so_item in self._so_whitelist_dict:
                so_version = so_list_item[1]
                lib_temp = self._so_full_match_result(
                    so_list_item, so_version)
            else:
                if so_list_item[0] in self._so_name_dict:
                    lib_temp = self._so_fuzzy_match_rules(so_list_item)
                    if not lib_temp:
                        so_version = 'd'
                        lib_temp = self._so_full_match_result(
                            so_list_item, so_version)
                else:
                    so_version = 'd'
                    lib_temp = self._so_full_match_result(
                        so_list_item, so_version)
            return lib_temp

        def _so_fuzzy_match_rules(self, so_list_item):
            """
            so library fuzzy match rules
            """
            so_result = self._so_name_dict.get(so_list_item[0])
            so_result.sort()
            for item in so_result:
                if item[0] > so_list_item[1]:
                    level_temp = item
                    break
            else:
                level_temp = []
            lib_temp = []
            if level_temp:
                lib_temp = [so_list_item[0], level_temp[0], so_list_item[2],
                            so_list_item[3], self._os_info[0],
                            self._os_info[1], level_temp[1], level_temp[2]]
            return lib_temp

        def _generate_so_whitelist_dict(self, so_list):
            """
            生成so列表所需白名单字典            :param so_list: so列表            :return:
            """
            so_whitelist_dict = dict()
            so_name_dict = defaultdict(list)

            os_name = "".join(self._os_info)
            whitelist_file = WhiteList.get_whitelist_name(so_list, os_name)
            whitelist_loader = \
                WhitelistLoaderFactory.generate_loader(WhitelistLoaderType.SO)
            for item in whitelist_file:
                so_dict1, so_dict2 = whitelist_loader.load_whitelist(item)
                so_whitelist_dict.update(so_dict1)
                for so_name in so_dict2:
                    so_name_dict[so_name].extend(so_dict2[so_name])
            return so_whitelist_dict, so_name_dict

    class JarMatch:
        """ jar匹配内嵌类 """
        def __init__(self, file_type):
            self._file_type = file_type
            self._os_info = list()

        def match_result(self, jar_list, os_info, level_number):
            """
            jar匹配结果            :param jar_list: jar列表            :param os_info: os信息            :param level_number: 等级列表            :return:
            """
            LOGGER.info("Start match jar_whitelist.")
            self._os_info = os_info
            scan_report_list = []
            jar_version = ''
            whitelist_dict = self._generate_jar_whitelist_dict(jar_list)
            for jar_item in jar_list:
                if jar_item[0] in whitelist_dict:
                    level_temp = whitelist_dict.get(jar_item[0])
                else:
                    level_temp = [str(WhiteList.MIGRATION_LEVEL), '']
                level_number[int(level_temp[0])] += 1
                jar_temp = [jar_item[0], jar_version, jar_item[1],
                            jar_item[2]] + os_info + level_temp
                jar_temp = WhiteList.add_file_type(
                    self._file_type, jar_temp, BinaryScanResultType.JAR)
                scan_report_list.append(jar_temp)
            LOGGER.info("End matching jar_whitelist, return scan report list.")
            return scan_report_list

        def _generate_jar_whitelist_dict(self, jar_list):
            """
            生成jar包所需白名单字典            :param jar_list: jar列表            :return:
            """
            os_name = "".join(self._os_info)
            whitelist_file = WhiteList.get_whitelist_name(jar_list, os_name)
            whitelist_loader = \
                WhitelistLoaderFactory.generate_loader(WhitelistLoaderType.JAR)
            whitelist_dict = whitelist_loader.load_whitelist(whitelist_file)
            return whitelist_dict

    class ExecMatch:
        """ 二进制文件匹配内嵌类 """
        def __init__(self, file_type):
            self._file_type = file_type

        def match_result(self, exec_file_list, os_info, level_number):
            """
            二进制文件匹配结果            :param exec_file_list: 二进制文件            :param os_info: os信息            :param level_number: 等级列表            :return:
            """
            exec_version = ''
            exec_url = ''
            scan_report_list = list()
            for exec_list_item in exec_file_list:
                exec_temp = [exec_list_item[0], exec_version,
                             exec_list_item[1], os_info[0], os_info[1],
                             str(WhiteList.MIGRATION_LEVEL), exec_url]
                exec_temp = WhiteList.add_file_type(
                    self._file_type, exec_temp, BinaryScanResultType.EXEC)
                scan_report_list.append(exec_temp)
                level_number[WhiteList.MIGRATION_LEVEL] += 1
            return scan_report_list

    class StaticMatch:
        """ 静态库文件匹配内嵌类 """
        def __init__(self, file_type):
            self._file_type = file_type

        def match_result(self, static_library_list, os_info, level_number):
            """
            静态库文件匹配结果            :param static_library_list: 静态库文件            :param os_info: os信息            :param level_number: 等级列表            :return:
            """
            static_library_version = ''
            static_library_url = ''
            scan_report_list = list()
            for static_library in static_library_list:
                temp = [static_library[0], static_library_version,
                        static_library[1], os_info[0], os_info[1],
                        str(WhiteList.MIGRATION_LEVEL), static_library_url]
                temp = WhiteList.add_file_type(
                    self._file_type, temp, BinaryScanResultType.STATIC_LIBRARY)
                scan_report_list.append(temp)
                level_number[WhiteList.MIGRATION_LEVEL] += 1
            return scan_report_list
```

