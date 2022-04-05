```python
#!/usr/bin/env python3
# -*-coding:utf-8-*-
"""
Usage：Decode and parse of encoding data
Copyright：2019, Huawei Technologies Co., Ltd. All rights reserved.
Modified：2019-11-18 Created
"""
from ctypes import cdll, c_char_p, c_int, c_void_p, c_ulonglong
import os


class DataParse:
    """
    brief: to parse single line
    """
    BINARY_FILE = 'config/decode_gcm.so'
    # 加密的配置文件，报错工作密钥等组件
    ROOT_IV = 'config/766848.txt'
    CONFIG_FILE = 'config/sign/2663448728466.txt.cipher'
    ENDE_SIZE = 16
    CONTENT_SIZE = 1024
    GCM_IV_LEN = 12
    @staticmethod
    def set_path(filepath: str):
        try:
            DataParse.config_path = os.path.normpath(
                os.path.join(filepath, DataParse.BINARY_FILE))
            DataParse.config_file = os.path.normpath(
                os.path.join(filepath, DataParse.CONFIG_FILE))
            root_iv_file = os.path.normpath(
                os.path.join(filepath, DataParse.ROOT_IV))
            with open(root_iv_file, 'r') as iv:
                DataParse.gcm_iv = iv.read()
        except OSError as os_err:
            print(os_err)
        except Exception as err:
            print(err)

    @staticmethod
    def decode_config(filepath: str):
        """
        brief: filepath config
        :param filepath:
        :return:
        """
        try:
            DataParse.decode_config_path = os.path.normpath(
                os.path.join(filepath, DataParse.BINARY_FILE))
            DataParse.config_file = os.path.normpath(
                os.path.join(filepath, DataParse.CONFIG_FILE))
            root_iv_file = os.path.normpath(
                os.path.join(filepath, DataParse.ROOT_IV))
            with open(root_iv_file, 'r') as iv:
                DataParse.gcm_iv = iv.read()
        except OSError as os_err:
            print(os_err)
        except Exception as err:
            print(err)

    @staticmethod
    def parse_binary(file):
        """
        parse so file and get return data
        if failed to malloc address, malloc_addr and ret_data  will be None.
        The caller needs to make a non-None judgment on the obtained value
        :param file
        :return:string data,filename
        """
        path_normal = os.path.dirname(file)
        file_size = os.path.getsize(file)
        filename = file[len(path_normal) + 1:].split(".")[0]
        rt_bf = cdll.LoadLibrary(DataParse.decode_config_path)
        file = file.encode()
        config_file = DataParse.config_file.encode()
        # Set the function's input and return value types
        rt_bf.MallocAddr.argtypes = [c_int]
        rt_bf.MallocAddr.restype = c_void_p
        rt_bf.GetDecode.argtypes = [c_char_p, c_char_p, c_int,
                                    c_char_p, c_void_p]
        rt_bf.GetDecode.restype = c_char_p
        rt_bf.FreeSpace.argtypes = [c_void_p]
        rt_bf.FreeSpace.restype = None
        # Apply for memory based on file size
        malloc_addr = rt_bf.MallocAddr(file_size)
        # Return the decrypted data
        ret_data = rt_bf.GetDecode(
            config_file, DataParse.gcm_iv.encode(), DataParse.GCM_IV_LEN,
            file, malloc_addr).decode()
        # Free memory
        rt_bf.FreeSpace(malloc_addr)
        return ret_data, filename

    @staticmethod
    def encode_str(int_content1, int_content2):
        '''
        加密非文件数据
        返回加密后的数据        '''
        config_file = DataParse.config_file.encode()
        rt_bf = cdll.LoadLibrary(DataParse.config_path)
        # Set the function's input and return value types
        rt_bf.MallocAddr.argtypes = [c_int]
        rt_bf.MallocAddr.restype = c_void_p
        rt_bf.GetEncodeInt.argtypes = [c_char_p, c_char_p, c_int, c_ulonglong,
                                       c_ulonglong, c_void_p]
        rt_bf.GetEncodeInt.restype = c_char_p
        rt_bf.FreeSpace.argtypes = [c_void_p]
        rt_bf.FreeSpace.restype = None
        # Apply for memory based on file size
        malloc_addr = rt_bf.MallocAddr(DataParse.CONTENT_SIZE)
        # Return the decrypted data
        ret_data = rt_bf.GetEncodeInt(config_file, DataParse.gcm_iv.encode(),
                                      DataParse.GCM_IV_LEN,
                                      int_content1,
                                      int_content2,
                                      malloc_addr).decode()
        # Free memory
        rt_bf.FreeSpace(malloc_addr)
        return ret_data

    @staticmethod
    def decode_str(cipher):
        '''
        解密密文数据        :param cipher: 密文数据        :return: 返回明文数据        '''
        config_file = DataParse.config_file.encode()

        num = cipher.split()
        cipher1 = int(num[0], 16)
        cipher2 = int(num[1], 16)
        rt_bf = cdll.LoadLibrary(DataParse.config_path)
        rt_bf.MallocAddr.argtypes = [c_int]
        rt_bf.MallocAddr.restype = c_void_p
        rt_bf.GetDecodeInt.argtypes = [c_char_p, c_char_p, c_int, c_ulonglong,
                                       c_ulonglong, c_void_p]
        rt_bf.GetDecodeInt.restype = c_char_p
        rt_bf.FreeSpace.argtypes = [c_void_p]
        rt_bf.FreeSpace.restype = None
        malloc_addr = rt_bf.MallocAddr(DataParse.CONTENT_SIZE)
        result = rt_bf.GetDecodeInt(config_file, DataParse.gcm_iv.encode(),
                                    DataParse.GCM_IV_LEN,
                                    cipher1, cipher2, malloc_addr).decode()
        rt_bf.FreeSpace(malloc_addr)
        dec_result = result.split()
        return int(dec_result[0]), int(dec_result[1])

    @staticmethod
    def str_to_list(whitelist):
        """
        parse str data to whitelist
        :param whitelist
        :return: whitelist
        """
        white_result = []
        temp_whitelist = whitelist.split('\n')
        if temp_whitelist[-1] == '':
            temp_whitelist.pop()
        for whitelist_line in temp_whitelist:
            try:
                line_item = whitelist_line.split(',')
                white_result.append(line_item)
            except Exception as ex:
                print(ex)
        return white_result

    @staticmethod
    def str_to_dict(dict_data):
        """
        parse str data to dict data
        :param dict_data
        :return:
        """
        res_dict = {}
        temp = dict_data.split('\n')
        if temp[-1] == '':
            temp.pop()
        for line in temp:
            line = line.strip()
            try:
                dict_index = line.split(':')[0]
                dict_value = line.split(':')[1]
                res_dict[dict_index] = dict_value
            except Exception as ex:
                print(ex)
        return res_dict

```

