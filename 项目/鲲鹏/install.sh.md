```shell
#!/bin/bash
# Copyright Huawei Technologies Co., Ltd. 2019-2019. All rights reserved.

current_dir=$(cd $(dirname $0); pwd)
current_space=0
source $current_dir/install_webfunc.sh
source $current_dir/install_func.sh
source $current_dir/common_path.sh
source $current_dir/common_func.sh

check_avail_architecture(){
    architecture=$(uname -m)
    package_architecture=$(cat $current_dir/architecture)
    if [ "$architecture" == "x86_64" ]; then
        if [ "$architecture" != "$package_architecture" ]; then
            echo -e "\e[1;31mThe installation package(Kunpeng) is not compatible with the environment. Replace it with X86_64 one.\e[0m"
            exit 1
        fi
    else
        if [ "$package_architecture" == "x86_64" ];then
            echo -e "\e[1;31mThe installation package(x86_64) is not compatible with the environment. Replace it with Kunpeng one.\e[0m"
            exit 1
        fi
    fi

}

# 加密
generate_encrypt_password()
{
    local install_path="$1"
    local web_path="${install_path}/tools/webui"
    cd "${web_path}"
    mkdir -m 700 -p common
    cd "${web_path}/common/"
    mkdir -m 700 -p nginx nordic greece rome new kronos

    local resource_path="${install_path}/tools/portal/resource"
    cd "${resource_path}"
    source "${install_path}/tools/venv/bin/activate"
    local password="$2"
    local encrypt_py
    if [ -f "${resource_path}/install_encrypting.pyc" ];then
        encrypt_py="${resource_path}/install_encrypting.pyc"
    else
        encrypt_py="${resource_path}/install_encrypting.py"
    fi
    python3 ${encrypt_py} ${install_path} ${password}
    if [ "$?" -ne "0" ]; then
        rm -rf "${web_path}/common/"
        echo -e "\e[1;31mEncrypting passphase failed.\e[0m"
        exit 1
    fi

    cd "${web_path}/common/"
    mv -f ./kronos/* ./nginx
    chown -R porting:porting ./
    chown porting:porting ../zeus
    echo -e "\e[1;32mEncrypting passphase successfully.\e[0m"
}

weakconsistency_lib_install(){
    local customize_path=$1
    # 提示用户是否安装对应依赖文件
    while true
    do
        read -p "The GCC/library version is earlier than expected(7.3.0/6.0.24). Do you want to install the dependencies for weakconsistency? (y/n default: y)" Y_N
        if [[ -z "${Y_N}" ]]; then
            Y_N="y"
        fi
        if [[ "$Y_N" == "n" ]] || [[ "$Y_N" == "N" ]]; then
            echo -e "\e[1;33mThe GCC/library version is earlier than expected(7.3.0/6.0.24). Upgrade the GCC or install the dependencies for weakconsistency by following the procedure described in section 2.1 \"Installing Dependencies for WeakConsistency\" in the User Guide. Otherwise, the automatic translation of weakconsistency files cannot be used.\e[0m"
            break
        elif [[ "$Y_N" == "y" ]] || [[ "$Y_N" == "Y" ]]; then
            bash $customize_path/portadv/tools/weakconsistency/staticcodeanalyzer/add_libraries.sh
            break
        else
            echo -e "\e[1;31mIncorrect entered information.\e[0m"
        fi
    done
}

# 总的调用函数
install_main()
{
    # 执行ldconfig命令刷新Id.so.cache文件
    ldconfig >/dev/null 2>&1
    #过滤空参数
    check_blank_para "$1"

    # 查询安装包与环境架构是否匹配
    check_avail_architecture

    # 兼容OS检查
    check_os_verison

    check_selinux_status "$os_name"

    if [[ "$1" == "web" ]];
    then
        # 判断安装途径是否为升级安装
        if [[ "$INSTALL_SOURCE_FLG" == "0" ]]; then
            # 检查用户安装是否有权限
            check_user_auth "web" "port"
            # 判断glibc版本没有2.17 退出安装
            check_glibc_version "$os_type"
            if [[ "$?" != "0" ]]; then
                exit 1
            fi
            # 检查用户是否已经安装了web
            check_web_is_install "port" "$os_name"
            # 获取用户输入的路径 默认为/opt
            # 用户交互输入路径判断
            get_customize_path "web" "port"
            # 根据用户输如返回路径
            customize_path=$(get_user_path)

            #判断创建文件是否正确
            check_para_tar "portadv" $customize_path
            # config ip and port,输入前端服务ip与端口号，并检验前端服务ip，端口号,porting Nginx-8084,porting Django-7998
            ip_port_config 8084 7998
            if [ "0" != "$?" ]; then
                file_rm $current_dir/portal $current_dir/webui $current_dir/cmd
                echo -e "\e[1;31mInstallation failed.\e[0m"
                exit 1
            fi
        else
            # 用户通过升级调用
            sed -i "s/INSTALL_SOURCE_FLG='1'/INSTALL_SOURCE_FLG='0'/" $current_dir/common_path.sh
            customize_path=$2
            ip_port_config 8084 7998
            if [ "0" != "$?" ]; then
                file_rm $current_dir/portal $current_dir/webui $current_dir/cmd
                echo -e "\e[1;31mInstallation failed.\e[0m"
                exit 1
            fi
        fi
        NGINX_DIR=$customize_path/portadv/tools/nginx-install
        # create logs directory
        logs_path=$customize_path/portadv/logs
        mkdir -p $logs_path
        #tar
        tar_check $logs_path
        # 创建portal目录
        mkdir -p $customize_path/portadv/tools/portal
        # 修改log.conf
        sed -i "s#/opt#$customize_path#g" $current_dir/portal/resource/conf/log_conf
        #portal
        cp -rf $current_dir/portal/resource $customize_path/portadv/tools/portal/resource
        cp -rf $current_dir/uninstall.sh $current_dir/check_port_conflict.sh $current_dir/common_func.sh $current_dir/change_ip_port.sh $customize_path/portadv/tools/
        # webui and cmd
        cp -rf $current_dir/webui $current_dir/cmd $customize_path/portadv/tools/
        cp -rf $current_dir/cmd/config $customize_path/portadv/
        cp -rf $current_dir/delete_file.sh $customize_path/portadv/tools/webui
        # cmake 安装cp
        cp -rf $current_dir/portal/cmake $customize_path/portadv/tools/
        # prevision 安装cp
        cp -rf $current_dir/portal/prevision $customize_path/portadv/tools/
        # inline-asm安装cp
        cp -rf $current_dir/portal/inline_asm $customize_path/portadv/tools/
        # all_asm全汇编二进制文件安装cp
        cp -rf $current_dir/portal/all_asm $customize_path/portadv/tools/
        # 日志logcp
        cp -rf $current_dir/portal/log $customize_path/portadv/tools/
        platform=$(uname -a | awk -F " " '{print $(NF-1)}')
        if [[ ${platform} == "aarch64" ]]; then
            # WeakConsistency弱内存cp
            cp -rf $current_dir/portal/weakconsistency $customize_path/portadv/tools/
        fi
        # 二进制文件验证的cms文件copy到tools下面
        cp -rf $current_dir/../cms  $customize_path/portadv/tools/
        # 解决方案移植模板放到resource/migration/目录下
        mkdir -p $customize_path/portadv/resource/migration
        cp -rf $customize_path/portadv/config/solution_migration_template/* $customize_path/portadv/resource/migration
        if [[ "$os_type" == "redhat" ]]; then
            rm -rf $customize_path/portadv/config/dbdecode
            rm -rf $customize_path/portadv/config/ssdecode
        elif [[ "$os_type" == "debian" ]]; then
            rm -rf $customize_path/portadv/config/dbdecode
            rm -rf $customize_path/portadv/config/ssdecode
        else
            rm -rf $customize_path/portadv/config/ssdecode
            rm -rf $customize_path/portadv/config/dbdecode
        fi
        # 创建porting用户和组
        create_user
        # Django启动前添加sqlite3临时文件锁
        mkdir -p $customize_path/portadv/config/lock_file
        touch $customize_path/portadv/config/lock_file/db.sqlite3.lock
        chmod -R 700 $customize_path/portadv/config/lock_file
        chown porting:porting $customize_path/portadv -R

        # install django
        djg_install $current_dir "port" $customize_path
        chown porting:porting $customize_path/portadv -R
        # 启动django，设置开机启动服务
        if [[ $os_name =~ (CentOS .* 6\.) || $os_name =~ (Red Hat .* 6\.) ]]; then
            sed -i "s#/opt#$customize_path#g" $current_dir/portal/script_port/gunicorn_port
            cp -rf $current_dir/portal/script_port/gunicorn_port /etc/init.d/
            chmod +x /etc/init.d/gunicorn_port
        else
            sed -i "s#/opt#$customize_path#g" $current_dir/portal/script_port/gunicorn_port.service
            cp -rf $current_dir/portal/script_port/gunicorn_port.service /etc/systemd/system/
        fi

        #############################
        cd $current_dir

        ###################################
        # python3安装
        py3_sqlite "$customize_path"
        #################################
        if [ "0" != "$?" ]; then
            file_rm $current_dir/portal $current_dir/webui $current_dir/cmd
            echo -e "\e[1;31mInstallation failed.\e[0m"
            exit 1
        fi
        cd $current_dir/webui/script_port/
        bash install_plugin.sh
        if [[ "$?" != "0" ]]; then
            file_rm $current_dir/portal $current_dir/webui $current_dir/cmd
            echo -e "\e[1;31mInstallation failed.\e[0m"
            echo -e "\e[1;31mNo image source is available. Please configure the network or mount an available image.\e[0m"
            exit 1
        fi
        old_mask=`umask`
        umask 077
        # install nginx
        ngx_install $current_dir "port" $customize_path
        local openssl_dir=$(openssl version -a | grep OPENSSLDIR | grep -v gcc | awk '{print $2}' | sed 's/\"//g')
        if [[ "$?" != "0" ]]; then
            file_rm $current_dir/portal $current_dir/webui $current_dir/cmd
            echo -e "\e[1;31mInstallation failed.\e[0m"
            echo -e "\e[1;31mNo found openssl\e[0m"
            exit 1
        fi
        local openssl_path="${openssl_dir}/openssl.cnf"
        local cert_path=$customize_path/portadv/tools/webui
        local python_path=$customize_path/portadv/tools/python-install/bin/python3
        local rand_script_path=$customize_path/portadv/tools/portal/resource/util/openssl_rand.pyc
        # 长度为15的字节数组，base64编码后为20个可见字符
        local password=`$python_path $rand_script_path -password 15`
        if [[ "$?" != "0" ]]; then
            file_rm $current_dir/portal $current_dir/webui $current_dir/cmd
            echo -e "\e[1;31mInstallation failed.\e[0m"
            exit 1
        fi
        local ca_password=`$python_path $rand_script_path -password 15`
        if [[ "$?" != "0" ]]; then
            file_rm $current_dir/portal $current_dir/webui $current_dir/cmd
            echo -e "\e[1;31mInstallation failed.\e[0m"
            exit 1
        fi
        generate_encrypt_password $customize_path/portadv $password
        if [[ "$?" != "0" ]]; then
            file_rm $current_dir/portal $current_dir/webui $current_dir/cmd
            echo -e "\e[1;31mInstallation failed.\e[0m"
            exit 1
        fi
        echo "It may take a few minutes to generate the certificate, please wait."
        cd $cert_path
        # 生成秘钥交换文件
        openssl dhparam -out dhparam.pem 2048 > /dev/null 2>&1
        # 生成ca.key
        openssl genrsa -aes256  -passout "pass:$ca_password" -out ca.key 4096 > /dev/null 2>&1
        # 生成ca.crt
        openssl req -new -x509 -sha256 -days 3650 -subj "/C=CN/CN=DevKit" -passin "pass:$ca_password" -key ca.key -out ca.crt > /dev/null 2>&1

        # 生成cert.key
        openssl genrsa -aes256 -passout "pass:$password" -out cert.key 4096 > /dev/null 2>&1
        # 生成cert.csr
        openssl req -new -sha256 -key cert.key -subj "/C=CN/CN=Porting-advisor" -passin "pass:$password" \
            -out cert.csr
        # 生成cert.crt
        openssl x509 -req -extfile <(cat $openssl_path <(printf "[SAN]\nsubjectAltName=DNS:Porting-advisor"))\
            -extensions SAN -CA ca.crt -CAkey ca.key \
            -CAcreateserial -days 3650 -passin "pass:$ca_password" -in cert.csr -out cert.pem > /dev/null 2>&1
        unset ca_password
        unset password
        bash delete_file.sh $customize_path ca.key
        bash delete_file.sh $customize_path ca.srl
        bash delete_file.sh $customize_path cert.csr
        echo "Certificate generated successfully. You can import the root certificate to the browser to mask security alarms when you access the tool. The root certificate is stored in $customize_path/portadv/tools/webui/ca.crt."
        umask $old_mask
        # 启动nginx，设置开机启动服务
        sed -i "s#/opt#$customize_path#g" $current_dir/webui/script_port/nginx_port*
        # =================================================================
        if [[ $os_name =~ (CentOS .* 6\.) || $os_name =~ (Red Hat .* 6\.) ]]; then
            sed -i "s#/opt#$customize_path#g" $customize_path/portadv/tools/webui/script_port/service_daemon.sh
            cp -rf $current_dir/webui/script_port/nginx_port /etc/init.d/
            chmod +x /etc/init.d/nginx_port
            service_set_nginx "start" "port" $customize_path
            if [ -f "/var/spool/cron/root" ]; then
                if grep "$customize_path/portadv/tools/webui/script_port/service_daemon.sh" /var/spool/cron/root
                then
                    echo "service_daemon already exists."
                else
                    echo "*/1 * * * * /bin/bash $customize_path/portadv/tools/webui/script_port/service_daemon.sh >/dev/null 2>&1" >> /var/spool/cron/root
                fi
            else
                touch /var/spool/cron/root
                echo "*/1 * * * * /bin/bash $customize_path/portadv/tools/webui/script_port/service_daemon.sh >/dev/null 2>&1" >> /var/spool/cron/root
                crontab /var/spool/cron/root
                service crond restart
            fi
        else
            if [[ "$os_type" == "redhat" ]]; then
                cp -rf $current_dir/webui/script_port/nginx_port.service /usr/lib/systemd/system/
            else
                cp -rf $current_dir/webui/script_port/nginx_port.service /etc/systemd/system/
            fi
            systemctl_set_nginx "start" "port" $customize_path
        fi
        chmod -R 600 $NGINX_DIR/logs/error.log
        chmod -R 600 $NGINX_DIR/logs/access.log
        chmod -R 600 $NGINX_DIR/logs/nginx.pid
        cd $customize_path/portadv
        file_rm $current_dir/portal $current_dir/webui $current_dir/cmd
        file_is_exist $current_dir/tmp_piplist
        chown porting:porting $customize_path/portadv -R
        chmod -R 700 $customize_path/portadv/config/ $customize_path/portadv/resource/ $customize_path/portadv/tools/ $customize_path/portadv/logs/
        tools_path=$customize_path/portadv/tools
        setting_path=$tools_path/portal/resource/resource
        chmod -R 500 $tools_path/portal/resource/
        chmod 700 $tools_path/portal/resource/ $tools_path/portal/resource/resource $tools_path/portal/resource/log $tools_path/webui
        chmod 500 $tools_path/check_port_conflict.sh $tools_path/uninstall.sh $tools_path/change_ip_port.sh
        chmod 600 $setting_path/settings.pyc $setting_path/gunicorn-conf.py $setting_path/wsgi.pyc $setting_path/secret_key.pyc
        chmod 600 $tools_path/portal/resource/log/porting.log $customize_path/portadv/logs/porting.log
        chmod 600 $tools_path/webui/zeus $tools_path/webui/common/rome/* $tools_path/webui/common/nordic/* $tools_path/webui/common/nginx/* $tools_path/webui/common/greece/*
        chmod 600 $tools_path/webui/cert.key $tools_path/webui/cert.pem $tools_path/webui/dhparam.pem
        chmod 600 $tools_path/portal/resource/db.sqlite3
        chmod 400 $customize_path/portadv/config/decode_gcm.so
        if [[ $os_name =~ (CentOS .* 6\.) || $os_name =~ (Red Hat .* 6\.) ]]; then
            add_porting_sudo  "service" "/sbin/service nginx_port stop"
            add_porting_sudo  "service" "/sbin/service nginx_port start"
        else
            systemctl_path=$(which systemctl)
            add_porting_sudo  "systemctl" "$systemctl_path restart nginx_port"
        fi
        # 暂时只支持Kunpeng, CentOS7.6上进行解决方案移植
        if [[ "$os_type" == "redhat" ]]; then
            add_porting_sudo_list
        fi
        rm -rf $customize_path/portadv/tools/webui/nginx-install
        passwd -l porting
        if [ -f /.dockerenv ]; then
            cd $customize_path/portadv/tools/portal/resource
            nohup $customize_path/portadv/tools/venv/bin/python3 $customize_path/portadv/tools/venv/bin/gunicorn -c resource/gunicorn-conf.py  resource.wsgi >/dev/null &
        else
            if [[ $os_name =~ (CentOS .* 6\.) || $os_name =~ (Red Hat .* 6\.) ]]; then
                chkconfig --add gunicorn_port
                chkconfig gunicorn_port on
                service gunicorn_port start
            else
                systemctl daemon-reload
                systemctl start gunicorn_port
                systemctl enable gunicorn_port
            fi
        fi
        cat /dev/null > $customize_path/portadv/logs/porting.log
        install_path_=$(cd $customize_path;pwd)
        # 汇编工具化迁移运行条件检查
        cp -rf $current_dir/check_asm_env.sh $customize_path/portadv/tools/
        chown porting:porting $customize_path/portadv/tools/check_asm_env.sh
        chmod 500 $customize_path/portadv/tools/check_asm_env.sh
        chmod 600 $customize_path/portadv/logs/install.log
        bash $customize_path/portadv/tools/check_asm_env.sh install
        if [[ ${platform} == "aarch64" ]];
        then
            weakconsistency_lib_install $customize_path
        fi
        ldconfig >/dev/null 2>&1
        echo -e "\e[1;32mPorting Web console is now running, go to:https://$IP:$HTTPSPORT.\e[0m"
        echo -e "\e[1;32mSuccessfully installed the Kunpeng Porting Advisor in $install_path_/portadv/.\e[0m"
        return 0
    fi

    if [[ "$1" == "cmd" ]]; then
        #########install#####
        if [[ "$INSTALL_SOURCE_FLG" == "0" ]]; then
            check_user_auth "cmd" "port"
            # 获取用户输入的路径 默认为/opt
            # 用户交互输入路径判断
            get_customize_path "cmd" "port"
            # 根据用户输如返回路径
            customize_path=$(get_user_path)
        else
            sed -i "s/INSTALL_SOURCE_FLG='1'/INSTALL_SOURCE_FLG='0'/" $current_dir/common_path.sh
            customize_path=$2
        fi
        check_para_tar "portadv" $customize_path
        # create logs directory
        logs_path=$customize_path/portadv/tools/cmd/logs
        mkdir -p $logs_path
        tar_check $logs_path
        chmod 600 $customize_path/portadv/tools/cmd/logs/install.log
        cmd_install "portadv" $current_dir $customize_path
        return 0
    fi
    echo -e "\e[1;31mIncorrect parameter.\e[0m"
    echo -e "\e[1;31mbash install.sh cmd\e[0m"
    echo -e "\e[1;31mbash install.sh web\e[0m"
    exit 1
}
```

