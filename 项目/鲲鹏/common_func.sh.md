```shell
#!/bin/bash

X86_SUPPORTED_CENTOS_VERSION="6.5 6.9 6.10 7.2 7.3 7.4 7.5 7.6 7.7 8.0 8.1"
X86_SUPPORTED_DEBIAN_VERSION="10"
X86_SUPPORTED_REDHAT_VERSION="6.5 6.9 6.10 7.2 7.3 7.4 7.5 7.6"
X86_SUPPORTED_SUSE_VERSION="15.1"
X86_SUPPORTED_UBUNTU_VERSION="18.04.1"
X86_SUPPORTED_OPENEULER_VERSION="20.3"
X86_SUPPORTED_NEOKYLIN_VERSION="V7Update6"
X86_SUPPORTED_LINX_VERSION="6.0.80"
X86_SUPPORTED_DEEPIN_VERSION="15.2 15.1.1"
X86_SUPPORTED_KYLIN_VERSION="V10"

ARM_SUPPORTED_CENTOS_VERSION="7.4 7.5 7.6 7.7 8.0 8.1"
ARM_SUPPORTED_DEBIAN_VERSION="10"
ARM_SUPPORTED_EULEROS_VERSION="2.0(SP8)"
ARM_SUPPORTED_SUSE_VERSION="15.1"
ARM_SUPPORTED_UBUNTU_VERSION="18.04.1"
ARM_SUPPORTED_OPENEULER_VERSION="20.03"
ARM_SUPPORTED_NEOKYLIN_VERSION="V7Update6 V7Update5"
ARM_SUPPORTED_LINX_VERSION="6.0.90"
ARM_SUPPORTED_DEEPIN_VERSION="15.2"
ARM_SUPPORTED_KYLIN_VERSION="V10"
ARM_SUPPORTED_UOS_VERSION="20 SP1"

# 获取cpu核数
get_cpu_core_num()
{
    cpu_core_nums=$(lscpu |grep "^CPU(s):" | awk '{print$2}')
    if [ "$cpu_core_nums" -le 4 ]; then
        cpu_core_nums=4
    elif [ "$cpu_core_nums" -ge 8 ] ; then
        cpu_core_nums=8
    fi
    echo $cpu_core_nums
    return 0
}


# 修改python配置文件gunicorn-conf.py
sed_workers_path()
{
    if [ "$2" == "port" ]; then
        path="$1/portal/resource/resource"
        num=7998
    else
        path="$1/portal/resource/dependencydjango"
        num=7996
    fi
    core_num=$(cat /proc/cpuinfo |grep "processor"|wc -l)
    sed -i "s/$num/$tool_port/" $path/gunicorn-conf.py
    sed -i "s#/opt#$3#g" $path/gunicorn-conf.py
    if [ "$core_num" -lt 4 ]; then
        sed -i "s/workers = multiprocessing.cpu_count()/workers = 4/" $path/gunicorn-conf.py
    elif [ "$core_num" -ge 4 ] && [ "$core_num" -le 8 ]; then
        sed -i "s/workers = multiprocessing.cpu_count()/workers = $core_num/" $path/gunicorn-conf.py
    else
        sed -i "s/workers = multiprocessing.cpu_count()/workers = 8/" $path/gunicorn-conf.py
    fi
}

# 检查服务器类型属于ARM或者X86
check_server_type()
{
    server_type=$(uname -a | grep "x86_64")
    if [[ -z $server_type ]]; then
        server_type=$(uname -a | grep "aarch64")
        if [[ -z $server_type ]]; then
            echo -e "\e[1;31mThe OS type is not supported.\e[0m"
            echo -e "\e[1;31mCheck the compatibility at:\e[0m"
            echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
            exit 1
        else
            server_type="arm"
        fi
    else
        server_type="x86"
    fi
}

# 检查centOS、Redhat、中标麒麟或者是欧拉OS
check_redhat()
{
    os_version=$(cat /etc/system-release)
    if [[ $os_version =~ Red ]]; then
        os_version="redhat"
    elif [[ $os_version =~ CentOS ]]; then
        os_version="centos"
    elif [[ $os_version =~ NeoKylin ]]; then
        os_version="NeoKylin"
    elif [[ $os_version =~ EulerOS ]]; then
        os_version="EulerOS"
    else
        echo -e "\e[1;31mThe OS type is not supported.\e[0m"
        echo -e "\e[1;31mCheck the compatibility at:\e[0m"
        echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
        exit 1
    fi
}

# 检查debian系统
check_debian()
{
    os_version=$(cat /etc/os-release)
    if [[ $os_version =~ Debian ]]; then
        os_version="debian"
    elif [[ $os_version =~ Linx ]]; then
        os_version="linx"
    elif [[ $os_version =~ Deepin ]]; then
        os_version="deepin"
    elif [[ $os_version =~ Ubuntu ]]; then
        os_version="ubuntu"
    elif [[ $os_version =~ uos ]]; then
        os_version="uos"
    else
        echo -e "\e[1;31mThe OS type is not supported.\e[0m"
        echo -e "\e[1;31mCheck the compatibility at:\e[0m"
        echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
        exit 1
    fi
}

# 兼容OS检查
check_os_verison()
{
    #判断OS属于哪个系列
    os_name="other_os"
    if [ -f /etc/redhat-release ]; then
        os_type=redhat
        os_name=$(cat /etc/redhat-release)
        os_verison_number=$(cat /etc/redhat-release | grep -Po "\d+\.\d+")
        check_server_type
        if [[ $server_type == "x86" ]]; then
            check_redhat
            if [[ $os_version == "centos" ]]; then
                if [[ ! $X86_SUPPORTED_CENTOS_VERSION =~ $os_verison_number ]]; then
                    echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                    echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                    echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                    exit 1
                fi
            elif [[ $os_version == "redhat" ]]; then
                if [[ ! $X86_SUPPORTED_REDHAT_VERSION =~ $os_verison_number ]]; then
                    echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                    echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                    echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                    exit 1
                fi
            elif [[ $os_version == "NeoKylin" ]]; then
                os_verison_number=$(cat /etc/system-release | awk '{print$6}')
                if [[ ! $X86_SUPPORTED_NEOKYLIN_VERSION =~ $os_verison_number ]]; then
                    echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                    echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                    echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                    exit 1
                fi
            else
                echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                exit 1
            fi
        else
            check_redhat
            if [[ $os_version == "centos" ]]; then
                if [[ ! $ARM_SUPPORTED_CENTOS_VERSION =~ $os_verison_number ]]; then
                    echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                    echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                    echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                    exit 1
                fi
            elif [[ $os_version == "NeoKylin" ]]; then
                os_verison_number=$(cat /etc/system-release | awk '{print$6}')
                if [[ ! $ARM_SUPPORTED_NEOKYLIN_VERSION =~ $os_verison_number ]]; then
                    echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                    echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                    echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                    exit 1
                fi
            elif [[ $os_version == "EulerOS" ]]; then
                os_verison_number=$(cat /etc/system-release | awk '{print$3$4}')
                if [[ $ARM_SUPPORTED_EULEROS_VERSION != $os_verison_number ]]; then
                    echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                    echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                    echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                    exit 1
                fi
            else
                echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                exit 1
            fi
        fi
    elif [ -f /etc/openEuler-release ]; then
        os_type=redhat
        os_name=$(cat /etc/openEuler-release)
        if [[ ! $os_name =~ $ARM_SUPPORTED_OPENEULER_VERSION ]]; then
            echo -e "\e[1;31mThe OS type is not supported.\e[0m"
            echo -e "\e[1;31mCheck the compatibility at:\e[0m"
            echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
            exit 1
        fi
    elif [ -f /etc/kylin-release ]; then
        os_type=redhat
        os_name=$(cat /etc/kylin-release)
        check_server_type
        if [[ $server_type == "x86" ]]; then
            if [[ ! $os_name =~ $X86_SUPPORTED_KYLIN_VERSION ]]; then
                echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                exit 1
            fi
        else
            if [[ ! $os_name =~ $ARM_SUPPORTED_KYLIN_VERSION ]]; then
                echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                exit 1
            fi
        fi
    elif [ -f /etc/debian_version ]; then
        os_type=debian
        os_verison_number=$(cat /etc/os-release)
        check_server_type
        if [[ $server_type == "x86" ]]; then
            check_debian
            if [[ $os_version == "debian" ]]; then
                if [[ ! $os_verison_number =~ $X86_SUPPORTED_DEBIAN_VERSION ]]; then
                    echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                    echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                    echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                    exit 1
                fi
            elif [[ $os_version == "linx" ]]; then
                if [[ ! $os_verison_number =~ $X86_SUPPORTED_LINX_VERSION ]]; then
                    echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                    echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                    echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                    exit 1
                fi
            elif [[ $os_version == "deepin" ]]; then
                os_verison_number=$(cat /etc/os-release |grep VERSION= |awk -F '"' '{print$2}')
                if [[ ! $X86_SUPPORTED_DEEPIN_VERSION =~ $os_verison_number ]]; then
                    echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                    echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                    echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                    exit 1
                fi
            elif [[ $os_version == "ubuntu" ]]; then
                if [[ ! $os_verison_number =~ $X86_SUPPORTED_UBUNTU_VERSION ]]; then
                    echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                    echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                    echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                    exit 1
                fi
            else
                echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                exit 1
            fi
        else
            check_debian
            if [[ $os_version == "debian" ]]; then
                if [[ ! $os_verison_number =~ $ARM_SUPPORTED_DEBIAN_VERSION ]]; then
                    echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                    echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                    echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                    exit 1
                fi
            elif [[ $os_version == "linx" ]]; then
                if [[ ! $os_verison_number =~ $ARM_SUPPORTED_LINX_VERSION ]]; then
                    echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                    echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                    echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                    exit 1
                fi
            elif [[ $os_version == "deepin" ]]; then
                if [[ ! $os_verison_number =~ $ARM_SUPPORTED_DEEPIN_VERSION ]]; then
                    echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                    echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                    echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                    exit 1
                fi
            elif [[ $os_version == "ubuntu" ]]; then
                if [[ ! $os_verison_number =~ $ARM_SUPPORTED_UBUNTU_VERSION ]]; then
                    echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                    echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                    echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                    exit 1
                fi
            elif [[ $os_version == "uos" ]]; then
                if [[ ! $os_verison_number =~ $ARM_SUPPORTED_UOS_VERSION ]]; then
                    echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                    echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                    echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                    exit 1
                fi
            else
                echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                exit 1
            fi
        fi
    elif [ -f /etc/SUSE-brand ]; then
        os_type=suse
        check_server_type
        os_verison_number=$(cat /etc/os-release)
        if [[ $server_type == "x86" ]]; then
            if [[ ! $os_verison_number =~ $X86_SUPPORTED_SUSE_VERSION ]]; then
                echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                exit 1
            fi
        else
            if [[ ! $os_verison_number =~ $ARM_SUPPORTED_SUSE_VERSION ]]; then
                echo -e "\e[1;31mThe OS type is not supported.\e[0m"
                echo -e "\e[1;31mCheck the compatibility at:\e[0m"
                echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
                exit 1
            fi
        fi
    else
        echo -e "\e[1;31mThe OS type is not supported.\e[0m"
        echo -e "\e[1;31mCheck the compatibility at:\e[0m"
        echo -e "\e[1;31mhttps://support-it.huawei.com/kunpeng-software-support/#/index/search\e[0m"
        exit 1
    fi
}

#检查升级后是否存在sourcecode报告目录
check_cmd_sourcecode()
{
    report_dir="/tools/cmd/report/"
    report_dir_path=$1${report_dir}
    if [ -d ${report_dir_path} ];
    then
        sourcecode_dir="sourcecode"
        sourcecode_dir_path=${report_dir_path}${sourcecode_dir}
        if [ ! -d ${sourcecode_dir_path} ];
        then
            report_list=$(ls ${report_dir_path})
            mkdir ${sourcecode_dir_path}
            if [ "0" != "$?" ];
            then
                echo -e "\e[1;33mFailed to create the source code report directory.\e[0m"
            fi
            for report in ${report_list};
            do
                if [[ ${report} == "package" ]];
                then
                    continue
                else
                    mv ${report_dir_path}${report} ${sourcecode_dir_path}
                    if [ "0" != "$?" ];
                    then
                        echo -e "\e[1;33mFailed to move the report.\e[0m"
                    fi
                fi
            done
        fi
    fi
}
```

