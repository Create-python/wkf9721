```shell
#!/bin/bash
# Copyright Huawei Technologies Co., Ltd. 2019-2020. All rights reserved.
current_dir=$(cd $(dirname $0); pwd)
source $current_dir/common_func.sh
source $current_dir/install_func.sh
source $current_dir/common_path.sh
avail_space=2097152
current_space=0

# 升级主函数
upgrade_main()
{
    # 兼容OS检查
    check_os_verison
    get_upgrade_path 'port'
    install_path=$UPGRADE_PATH/portadv
    backup_path=$UPGRADE_PATH/portadvisor/upgrade_backup
    backup_dir=$UPGRADE_PATH/portadvisor/upgrade_backup/portadv/
    backup_files=$UPGRADE_PATH/portadvisor/upgrade_backup/*_port.service
    #升级开始
    echo -e "\e[1;32mStart upgrade.\e[0m"
    echo -e "\e[1;33mPlease do not press Ctrl+Z or Ctrl+C or restart the system during the upgrade. \e[0m"
    #执行升级脚本
    upgrade
}


set_port_service_time(){
    if [ -f "/var/spool/cron/root" ]; then
        sed -i "s#/opt#$UPGRADE_PATH#g" $UPGRADE_PATH/portadv/tools/webui/script_port/service_daemon.sh
        if grep "$UPGRADE_PATH/portadv/tools/webui/script_port/service_daemon.sh" /var/spool/cron/root
        then
            echo "service_daemon is exist"
        else
            echo "*/1 * * * * /bin/bash $UPGRADE_PATH/portadv/tools/webui/script_port/service_daemon.sh >/dev/null 2>&1" >> /var/spool/cron/root
        fi
    else
        touch /var/spool/cron/root
        echo "*/1 * * * * /bin/bash $UPGRADE_PATH/portadv/tools/webui/script_port/service_daemon.sh >/dev/null 2>&1" >> /var/spool/cron/root
        crontab /var/spool/cron/root
        service crond restart
    fi
}

check_avail_space(){
    #得到当前/opt录下剩余可用的大小
    UPGRADE_PATH=$1
    opt_space=$(space_ $UPGRADE_PATH 0)
    if [ ! $opt_space ]; then
        opt_space=$(space_ $UPGRADE_PATH 1)
    fi
    if [ $opt_space ]; then
        if [ "$opt_space" -lt "$avail_space" ]; then
            echo -e "\e[1;31mThe $UPGRADE_PATH directory does not have enough space for backup.\e[0m"
            echo -e "\e[1;31mUpgrade failed.\e[0m"
            exit 1
        fi
    else
        opt_space=$(df -lk |sed '1d;/ /!N;s/\n//;s/ \+/ /;' | grep /$ | awk '{print int($4)}')

        if test -z "$opt_space"
        then 
            echo -e "\e[1;31mFailed to obtain information about the $UPGRADE_PATH directory during backup.\e[0m"
            echo -e "\e[1;31mUpgrade failed.\e[0m"
            exit 1
        fi

        if [ "$opt_space" -lt "$avail_space" ]; then 
            echo -e "\e[1;31mThe $UPGRADE_PATH directory does not have enough space for backup.\e[0m"
            echo -e "\e[1;31mUpgrade failed.\e[0m"
            exit 1
        fi
    fi
}

check_no_running_task(){
    check_flag='1'
    while [ "$check_flag" == "1" ]
    do
        read -p "Please confirm there are no running tasks(yes/no): " choice
        if [ "$choice" == "yes" ]; then
            check_flag='0'
        elif [ "$choice" == "no" ]; then
            echo -e "\e[1;31mPlease stop running tasks first.\e[0m"
            exit 1
        fi
    done
}


# 检查cmd模式当前是否有任务进行
function check_cmd_task()
{
    task_count=$(ps -ef | grep porting-advisor | grep $USER | wc -l)
    if [[ $task_count > "1" ]]; then
        read -p "A task is running.Are you sure you want to upgrade porting advisor?(y/n)" UPGRADE
        if [ "$UPGRADE" == "n" ]; then
            exit 0
        elif [ "$UPGRADE" == "y" ]; then
            echo -e "\e[1;32mThe Kunpeng Porting Advisor is upgrading.\e[0m"
        else
            echo -e "\e[1;31mIncorrect entered information.\e[0m"
            exit 1
        fi
    else
        echo -e "\e[1;32mThe Kunpeng Porting Advisor is upgrading\e[0m"
    fi
}


# 检查web模式当前是否有任务进行
function check_web_task()
{
    cpu_core_nums=$(get_cpu_core_num)
    process_nums=(`ps -ef | grep gunicorn | grep porting | wc -l`)
    if [[ $(($cpu_core_nums+1)) < $process_nums ]]; then
        read -p "A task is running.Are you sure you want to upgrade porting advisor?(y/n)" UPGRADE_WEB
        if [ "$UPGRADE_WEB" == "n" ]; then
            exit 0
        elif [ "$UPGRADE_WEB" == "y" ]; then
            echo -e "\e[1;32mThe Kunpeng Porting Advisor is upgrading\e[0m"
        else
            echo -e "\e[1;31mIncorrect entered information.\e[0m"
            exit 1
        fi
    fi
}


#upgrade
upgrade(){
    check_no_running_task

    if [ -d "$install_path" ]; then
        if [ -d "$install_path/tools/webui" ]; then
            upgrade_web 
        else
            upgrade_cmd 
        fi
    else
        echo -e "\e[1;31mNo installation tool is detected. Please excute install.sh.\e[0m"
        exit 1
    fi
}

#web_upgrade
upgrade_web(){
    #清理目录中的上次备份信息
    if [ -d $backup_dir ]; then
        rm -rf $backup_dir
    fi

    #检查剩余空间大小
    check_avail_space $UPGRADE_PATH
    echo "Checking..."
    #解压cmd.tar.gz，查看版本号
    tar -zxvf cmd.tar.gz >/dev/null 2>&1
    cd $install_path/tools/cmd/bin/
    all_old_version=$(./porting-advisor --version | grep [0-9] -o | awk '{printf$1}')
    old_version=${all_old_version:0:3}
    old_spc_version=${all_old_version:3:6}
    if [ ! $old_spc_version ]; then
        old_spc_version=0
    fi
    cd $current_dir
    all_new_version=$(./cmd/bin/porting-advisor --version | grep [0-9] -o | awk '{printf$1}')
    new_version=${all_new_version:0:3}
    new_spc_version=${all_new_version:3:6}
    if [ ! $new_spc_version ]; then
        new_spc_version=0
    fi
    rm -rf cmd

    #支持多用户版本号
    multi_user_version="220"
    if [ "$old_version" -ge "$multi_user_version" ]; then
        check_web_task
    fi


    if [ "$old_version" -gt "$new_version" ]; then
        echo -e "\e[1;31mThe installed tool version is later than the upgrade target version. Please check.\e[0m"
        exit 1
    elif [ "$old_version" -lt "$new_version" ]; then
        echo -e "\e[1;32mChecking the tool version succeeded.\e[0m"
    else
        if [ "$old_spc_version" -lt "$new_spc_version" ]; then
            echo -e "\e[1;32mChecking the tool spc_version succeeded.\e[0m"
        else
            echo -e "\e[1;31mThe installed tool version is the same as or later than the upgrade target version. Please check.\e[0m"
            exit 1
        fi
    fi

    #创建备份目录
    if [ ! -d "$backup_path" ]; then
        mkdir -p $backup_path
    fi 
    #停止服务，备份用户nginx,django配置
    if [ -f /.dockerenv ]; then
        kill -9 $(pidof $install_path/tools/venv/bin/python3)
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to stop gunicorn_port command during backup.\e[0m"
            roll_back_before_install web
            exit 1
        fi
        sh $install_path/tools/webui/script_port/service_nginx.sh stop "$UPGRADE_PATH"
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to stop nginx_port command during backup.\e[0m"
            roll_back_before_install web
            exit 1
        fi

        if [ -e /etc/systemd/system/gunicorn_port.service ]; then
            cp -prf /etc/systemd/system/gunicorn_port.service $backup_path
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to back up gunicorn_port.service.\e[0m"
                roll_back_before_install web
                exit 1
            fi
        else
            echo -e "\e[1;31m/etc/systemd/system/gunicorn_port.service does not exist. Please check the system.\e[0m"
            exit 1
        fi

        if [[ "$os_type" == "redhat" ]]; then
            if [ -e /usr/lib/systemd/system/nginx_port.service ];then
                cp -prf /usr/lib/systemd/system/nginx_port.service $backup_path
                if [ "0" != "$?" ]; then
                    echo -e "\e[1;31mFailed to back up nginx_port.service.\e[0m"
                    roll_back_before_install web
                    exit 1
                fi
            else
                echo -e "\e[1;31m/usr/lib/systemd/system/nginx_port.service does not exist. Please check the system.\e[0m"
                exit 1
            fi
        else
            if [ -e /etc/systemd/system/nginx_port.service ]; then
                cp -prf /etc/systemd/system/nginx_port.service $backup_path
                if [ "0" != "$?" ]; then
                    echo -e "\e[1;31mbackup nginx_port.service failed\e[0m"
                    roll_back_before_install web
                    exit 1
                fi
            else
                echo -e "\e[1;31m/etc/systemd/system/nginx_port.service does not exist. Please check the system.\e[0m"
                exit 1
            fi
        fi
    else
        if [[ $os_name =~ (CentOS .* 6\.)  || $os_name =~ (Red Hat .* 6\.) ]]; then
            if [ -f "/var/spool/cron/root" ]; then
                if grep "$UPGRADE_PATH/portadv/tools/webui/script_port/service_daemon.sh" /var/spool/cron/root
                then
                   sed -i '/script_port\/service_daemon.sh/d' /var/spool/cron/root
                fi
            fi
            service gunicorn_port stop
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the service stop gunicorn_port command during backup.\e[0m"
                roll_back_before_install web
                exit 1
            fi
            chkconfig gunicorn_port off
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the service chkconfig off gunicorn_port command during backup.\e[0m"
                roll_back_before_install web
                exit 1
            fi
            chkconfig --del gunicorn_port
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the service chkconfig --del gunicorn_port command during backup.\e[0m"
                roll_back_before_install web
                exit 1
            fi
            service nginx_port stop
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the service stop nginx_port command during backup.\e[0m"
                roll_back_before_install web
                exit 1
            fi
            chkconfig nginx_port off
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the service chkconfig off nginx_port command during backup.\e[0m"
                roll_back_before_install web
                exit 1
            fi
            chkconfig --del nginx_port
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the service chkconfig --del nginx_port command during backup.\e[0m"
                roll_back_before_install web
                exit 1
            fi
            if [ -e /etc/init.d/gunicorn_port ];then
                cp -prf /etc/init.d/gunicorn_port $backup_path
                if [ "0" != "$?" ]; then
                    echo -e "\e[1;31mFailed to execute the backup gunicorn_port.\e[0m"
                    roll_back_before_install web
                    exit 1
                fi
            else
                echo -e "\e[1;31m/etc/init.d/gunicorn_port does not exist. Please check the system.\e[0m"
                exit 1
            fi
            if [ -e /etc/init.d/nginx_port ]; then
                cp -prf /etc/init.d/nginx_port $backup_path
                if [ "0" != "$?" ]; then
                    echo -e "\e[1;31mFailed to back up nginx_port.\e[0m"
                    roll_back_before_install web
                    exit 1
                fi
            else
                echo -e "\e[1;31m/etc/init.d/nginx_port does not exist. Please check the system.\e[0m"
                exit 1
            fi
        else
            systemctl stop gunicorn_port
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the systemctl stop gunicorn_port command during backup.\e[0m"
                roll_back_before_install web
                exit 1
            fi
            systemctl stop nginx_port
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the systemctl stop nginx_port command during backup.\e[0m"
                roll_back_before_install web
                exit 1
            fi
            systemctl disable gunicorn_port
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the systemctl disable guncorn_port command during backup.\e[0m"
                roll_back_before_install web
                exit 1
            fi
            systemctl disable nginx_port
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the systemctl disable nginx_port command during backup.\e[0m"
                roll_back_before_install web
                exit 1
            fi

            if [ -e /etc/systemd/system/gunicorn_port.service ]; then
                cp -prf /etc/systemd/system/gunicorn_port.service $backup_path
                if [ "0" != "$?" ]; then
                    echo -e "\e[1;31mFailed to back up gunicorn_port.service.\e[0m"
                    roll_back_before_install web
                    exit 1
                fi
            else
                echo -e "\e[1;31m/etc/systemd/system/gunicorn_port.service does not exist. Please check the system.\e[0m"
                exit 1
            fi

            if [[ "$os_type" == "redhat" ]]; then
                if [ -e /usr/lib/systemd/system/nginx_port.service ];then
                    cp -prf /usr/lib/systemd/system/nginx_port.service $backup_path
                    if [ "0" != "$?" ]; then
                        echo -e "\e[1;31mFailed to back up nginx_port.service.\e[0m"
                        roll_back_before_install web
                        exit 1
                    fi
                else
                    echo -e "\e[1;31m/usr/lib/systemd/system/nginx_port.service does not exist. Please check the system.\e[0m"
                    exit 1
                fi
            else
                if [ -e /etc/systemd/system/nginx_port.service ]; then
                    cp -prf /etc/systemd/system/nginx_port.service $backup_path
                    if [ "0" != "$?" ]; then
                        echo -e "\e[1;31mBackup nginx_port.service failed\e[0m"
                        roll_back_before_install web
                        exit 1
                    fi
                else
                    echo -e "\e[1;31m/etc/systemd/system/nginx_port.service does not exist. Please check the system.\e[0m"
                    exit 1
                fi
            fi
        fi
    fi

    #进行备份
    if [ -d $install_path ]; then 
        cp -rfp $install_path $backup_path
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to back up portadv and to execute the cp command. Please check the system.\e[0m"
            roll_back_before_install web
            exit 1
        fi
    else
        echo -e "\e[1;31mThe portadv dir does not exist. Please check the system.\e[0m"
        exit 1
    fi
    rm -rf $install_path

    #初始化还原的的路径
    bak_sql_path=$backup_path/portadv/tools/portal/resource/db.sqlite3
    bak_sql_usrmag_migration=$backup_path/portadv/tools/portal/resource/usermanager/migrations
    bak_sql_tasksmag_migration=$backup_path/portadv/tools/portal/resource/taskmanager/migrations
    bak_sql_operationlogmag_migration=$backup_path/portadv/tools/portal/resource/operationlog/migrations
    bak_logconf_path=$backup_path/portadv/config/log.conf

    sed -i "s/INSTALL_SOURCE_FLG='0'/INSTALL_SOURCE_FLG='1'/" $current_dir/common_path.sh
    source $current_dir/.install.sh
    install_main web $UPGRADE_PATH
    if [ "0" != "$?" ]; then
        #升级失败回归
        sed -i "s/INSTALL_SOURCE_FLG='1'/INSTALL_SOURCE_FLG='0'/" $current_dir/common_path.sh
        echo -e "\e[1;31mUpgrade failed during installation of the Kunpeng Porting Advisor of the new version.\e[0m"
        echo -e "\e[1;31mPlease check possible reasons, such as lack of disk space or available ports.\e[0m"
        roll_back web
        exit 1
    fi
    sed -i "s/INSTALL_SOURCE_FLG='1'/INSTALL_SOURCE_FLG='0'/" $current_dir/common_path.sh
    #备份需要更新的文件
    if [ -d $install_path/tools ]; then 
        mv -f $install_path/tools $install_path/../tools_portbak
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to back up the new version of the tool.\e[0m"
            roll_back web
            exit 1
        fi
    else
        echo -e "\e[1;31mAn error occurred during installation of the new tool version because dir of the tool is missing.\e[0m"
        roll_back web
        exit 1
    fi

    if [ -d $install_path/config ]; then
        mv -f  $install_path/config $install_path/../config_portbak
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to back up configuration of the new version.\e[0m"
            roll_back web
            exit 1
        fi
    else
        echo -e "\e[1;31mAn error occurred during installation of the new tool version because configuration dir is missing.\e[0m"
        roll_back web
        exit 1
    fi

    if [ -d $install_path/resource ]; then
        mv -f $install_path/resource $install_path/../resource_portbak
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to restore the tool information.\e[0m"
            roll_back web
            exit 1
        fi
    else
        echo -e "\e[1;31mAn error occurred during installation of the new tool version because resource dir is missing.\e[0m"
        roll_back web
        exit 1
    fi

    rm -rf $install_path

    if [ -d $backup_path/portadv ]; then
        cp -rpf $backup_path/portadv $UPGRADE_PATH/
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to restore the tool information.\e[0m"
            roll_back web
            exit 1
        fi
    else
        echo -e "\e[1;31mThe backup portadv dir does not exist. Please check the system.\e[0m"
        roll_back web
        exit 1
    fi 
    rm -rf $install_path/tools
    rm -rf $install_path/config
    rm -rf $install_path/resource

    mv -f $install_path/../tools_portbak $install_path/tools
    if [ "0" != "$?" ]; then
        echo -e "\e[1;31mFailed to update the tool information.\e[0m"
        roll_back web
        exit 1
    fi
    mv -f $install_path/../config_portbak $install_path/config
    if [ "0" != "$?" ]; then
        echo -e "\e[1;31mFailed to update the configuration information.\e[0m"
        roll_back web
        exit 1
    fi
    mv -f $install_path/../resource_portbak $install_path/resource
    if [ "0" != "$?" ]; then
        echo -e "\e[1;31mFailed to update the resource information.\e[0m"
        roll_back web
        exit 1
    fi

    # 支持密钥、证书的版本号
    cert_version="222"
    if [ "$old_version" -ge "$cert_version" ]; then
        restore_cert_and_key
    fi

    #暂停服务，还原用户nginx,django服务配置
    if [ -f /.dockerenv ]; then
        sh $install_path/tools/webui/script_port/service_nginx.sh stop "$UPGRADE_PATH"
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to stop nginx_port command during restoration.\e[0m"
            roll_back web
            exit 1
        fi
        kill -9 $(pidof $install_path/tools/venv/bin/python3)
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to stop gunicorn_port command during restoration.\e[0m"
            roll_back web
            exit 1
        fi
    else
        if [[ $os_name =~ (CentOS .* 6\.)  || $os_name =~ (Red Hat .* 6\.) ]]; then
            #service
            if [ -f "/var/spool/cron/root" ]; then
               if grep "$UPGRADE_PATH/portadv/tools/webui/script_port/service_daemon.sh" /var/spool/cron/root
               then
                sed -i '/script_port\/service_daemon.sh/d' /var/spool/cron/root
               fi
            fi
            cd /
            cd $install_path
            service nginx_port stop
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the service stop nginx_port command during restoration.\e[0m"
                roll_back web
                exit 1
            fi
            chkconfig nginx_port off
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the service chkconfig off nginx_port command during restoration.\e[0m"
                roll_back web
                exit 1
            fi
            chkconfig --del nginx_port
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the service chkconfig --del nginx_port command during restoration.\e[0m"
                roll_back web
                exit 1
            fi

            service gunicorn_port stop
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the service stop gunicorn_port command during restoration. \e[0m"
                roll_back web
                exit 1
            fi
            chkconfig gunicorn_port off
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the service chkconfig off gunicorn_port command during restoration.\e[0m"
                roll_back web
                exit 1
            fi
            chkconfig --del gunicorn_port
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the service chkconfig --del gunicorn_port command during restoration.\e[0m"
                roll_back web
                exit 1
            fi
        else
            systemctl stop nginx_port
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the systemctl stop nginx_port command during restoration.\e[0m"
                roll_back web
                exit 1
            fi
            systemctl stop gunicorn_port
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to execute the systemctl stop gunicorn_port command during restoration.\e[0m"
                roll_back web
                exit 1
            fi
        fi
    fi

    #还原用户信息
    if [ -e $bak_sql_path ]; then
        cp -rfp $bak_sql_path $install_path/tools/portal/resource/
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to restore the database file.\e[0m"
            roll_back web
            exit 1
        fi
    else
        echo -e "\e[1;31mBackup sqlite3 file does not exist. Please check the system.\e[0m"
        roll_back web
        exit 1
    fi

    if [ -d $bak_sql_usrmag_migration ]; then
        cp -rfp $bak_sql_usrmag_migration $install_path/tools/portal/resource/usermanager/
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to restore migration dir.\e[0m"
            roll_back web
            exit 1
        fi
    else
        echo -e "\e[1;31mBackup migration dir does not exist. Please check the system.\e[0m"
        roll_back web
        exit 1
    fi

    if [ -d $bak_sql_tasksmag_migration ]; then
        cp -rfp $bak_sql_tasksmag_migration $install_path/tools/portal/resource/taskmanager/
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to restore migration dir.\e[0m"
            roll_back web
            exit 1
        fi
    else
        echo -e "\e[1;31mBackup migration dir does not exist. Please check the system.\e[0m"
        roll_back web
        exit 1
    fi

    if [ -d $bak_sql_operationlogmag_migration ];
    then
        cp -rfp $bak_sql_operationlogmag_migration $install_path/tools/portal/resource/operationlog/
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to restore migration dir.\e[0m"
            roll_back web
            exit 1
        fi
    else
        echo -e "\e[1;31mBackup migration dir does not exist. Please check the system.\e[0m"
        roll_back web
        exit 1
    fi

    if [ -e $bak_logconf_path ]; then
        if [ "$old_version" -ge "$multi_user_version" ]; then
            cp -rfp $bak_logconf_path $install_path/config/
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to restore logconf.\e[0m"
                roll_back web
                exit 1
            fi
        fi
    else
        echo -e "\e[1;31mThe backup log.conf file does not exist. Please check the system.\e[0m"
        roll_back web
        exit 1
    fi

    #数据处理
    cd $current_dir
    tar -zxvf portal.tar.gz portal/resource/weak_password.json
    mv $current_dir/portal/resource/weak_password.json $install_path/tools/portal/resource/weak_password.json
    cd $install_path/tools/portal/resource
    source $install_path/tools/venv/bin/activate
    python3 manage.pyc makemigrations
    python3 manage.pyc migrate
    python3 manage.pyc loaddata weak_password.json
    deactivate

    #删除备份
    rm -rf $backup_dir
    rm -rf $backup_files
    rm -rf $install_path/../tools_portbak
    rm -rf $install_path/../config_portbak
    rm -rf $install_path/../resource_portbak
    rm -f weak_password.json

    chown porting:porting $UPGRADE_PATH/portadv -R
    
    #检查端口冲突，若冲突，怎么修改配置的端口号
    cd $install_path/tools
    bash check_port_conflict.sh  $UPGRADE_PATH
    if [[ "$?" != "0" ]]; then
        exit 1
    fi

    if [[ $os_name =~ (CentOS .* 6\.)  || $os_name =~ (Red Hat .* 6\.) ]]; then
       set_port_service_time
    fi

    echo -e "\e[1;32mUpgraded succeeded.\e[0m"
    exit 0
}

#cmd_upgrade
upgrade_cmd(){
    #清理目录中的备份信息
    if [ -d $backup_dir ]; then
        rm -rf $backup_dir
    fi
    #检查剩余空间大小
    check_avail_space $UPGRADE_PATH
    echo "Checking..."
    #解压cmd.tar.gz，查看版本号
    tar -zxvf cmd.tar.gz >/dev/null 2>&1
    cd $install_path/tools/cmd/bin/
    all_old_version=$(./porting-advisor --version | grep [0-9] -o | awk '{printf$1}')
    old_version=${all_old_version:0:3}
    old_spc_version=${all_old_version:3:6}
    if [ ! $old_spc_version ]; then
        old_spc_version=0
    fi
    cd $current_dir
    all_new_version=$(./cmd/bin/porting-advisor --version | grep [0-9] -o | awk '{printf$1}')
    new_version=${all_new_version:0:3}
    new_spc_version=${all_new_version:3:6}
    if [ ! $new_spc_version ]; then
        new_spc_version=0
    fi
    rm -rf cmd

    #支持多用户版本号
    multi_user_version="220"
    if [ "$old_version" -ge "$multi_user_version" ]; then
        check_cmd_task
    fi


    if [ "$old_version" -gt "$new_version" ]; then
        echo -e "\e[1;31mThe installed tool version is later than the upgrade target version. Please check.\e[0m"
        exit 1
    elif [ "$old_version" -lt "$new_version" ]; then
        echo -e "\e[1;32mChecking the tool version succeeded.\e[0m"
    else
        if [ "$old_spc_version" -lt "$new_spc_version" ]; then
            echo -e "\e[1;32mChecking the tool spc_version succeeded.\e[0m"
        else
            echo -e "\e[1;31mThe installed tool version is the same as or later than the upgrade target version. Please check.\e[0m"
            exit 1
        fi
    fi

    #创建备份目录并备份
    if [ ! -d "$backup_path" ]; then
        mkdir -p $backup_path
    fi 

    if [ -d $install_path ]; then 
        cp -rfp $install_path $backup_path
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to back up portadv and to execute the cp command. Please check the system.\e[0m"
            roll_back_before_install cmd
            exit 1
        fi
    else
        echo -e "\e[1;31mThe portadv dir does not exist. Please check the system.\e[0m"
        exit 1
    fi
    rm -rf $install_path

    #初始化还原的的路径
    bak_report_path=$backup_path/portadv/tools/cmd/report
    bak_log_path=$backup_path/portadv/tools/cmd/logs
    bak_logconf_path=$backup_path/portadv/tools/cmd/config/log.conf

    sed -i "s/INSTALL_SOURCE_FLG='0'/INSTALL_SOURCE_FLG='1'/" $current_dir/common_path.sh
    source $current_dir/.install.sh
    install_main cmd $UPGRADE_PATH
    if [ "0" != "$?" ]; then
        #升级失败回滚
        sed -i "s/INSTALL_SOURCE_FLG='1'/INSTALL_SOURCE_FLG='0'/" $current_dir/common_path.sh
        echo -e "\e[1;31mUpgrade failed during installation of the Kunpeng Porting Advisor of the new version.\e[0m"
        echo -e "\e[1;31mPlease check possible reasons, such as no disks space.\e[0m"
        roll_back cmd
        exit 1
    fi
    sed -i "s/INSTALL_SOURCE_FLG='1'/INSTALL_SOURCE_FLG='0'/" $current_dir/common_path.sh
    #还原信息
    if [ -d $bak_report_path ]; then
        cp -rfp $bak_report_path $install_path/tools/cmd/
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to restore the CLI repoort.\e[0m"
            roll_back cmd
            exit 1
        fi
    else
        echo -e "\e[1;31mThe report dir does not exist. The user report is lost.\e[0m"
        exit 1
    fi

    if [ -e $bak_log_path ]; then
        cp -rfp $bak_log_path $install_path/tools/cmd/
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to restore logs.\e[0m"
            roll_back cmd 
            exit 1
        fi
    else
        echo -e "\e[1;31mThe logs dir does not exist. The user operation logs are lost.\e[0m"
        exit 1
    fi

    if [ -e $bak_logconf_path ]; then
        if [ "$old_version" -ge "$multi_user_version" ]; then
            cp -rfp $bak_logconf_path $install_path/tools/cmd/config/
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to restore logconf.\e[0m"
                roll_back cmd
                exit 1
            fi
        fi
    else
        echo -e "\e[1;31mThe logs dir does not exist. The user logconf lost.\e[0m"
        exit 1
    fi

    rm -rf $backup_dir
    rm -rf $backup_files
    check_cmd_sourcecode ${install_path}

    echo -e "\e[1;32mUpgrade successed\e[0m"
    exit 0
}

roll_back(){
    echo -e "\e[1;32mStart rollback.\e[0m"
    if [ "$1" == "web" ]; then
        rm -rf $install_path
        if [[ $os_name =~ (CentOS .* 6\.)  || $os_name =~ (Red Hat .* 6\.) ]]; then
            if [ -d $backup_path/portadv ]; then
                cp -rfp $backup_path/portadv $UPGRADE_PATH
                if [ "0" != "$?" ]; then
                    echo -e "\e[1;31mFailed to restore the rollback.\e[0m"
                    exit 1
                fi
            else
                echo -e "\e[1;31mRollback failed. The backup portadv dir does not exist.\e[0m"
            fi

            if [ -e $backup_path/nginx_port ]; then
                mv -f $backup_path/nginx_port /etc/init.d/
                if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to roll back nginx_port.\e[0m"
                exit 1
                fi
            else 
                echo -e "\e[1;31mFailed to roll back because backup nginx_port scrpit does not exist.\e[0m"
            fi

            if [ -e $backup_path/gunicorn_port ]; then
                mv -f $backup_path/gunicorn_port /etc/init.d/
                if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to roll back gunicorn_port.\e[0m"
                exit 1
                fi
            else
                echo -e "\e[1;31mRollback failed. The backup gunicorn_port.service does not exist.\e[0m"
            fi
        else
            if [ -d $backup_path/portadv ]; then
                cp -rfp $backup_path/portadv $UPGRADE_PATH
                if [ "0" != "$?" ]; then
                    echo -e "\e[1;31mFailed to roll back nginx_port.service.\e[0m"
                    exit 1
                fi
            else
                echo -e "\e[1;31mRollback failed. The backup portadv dir does not exist.\e[0m"
            fi

            if [ -e $backup_path/gunicorn_port.service ]; then
                mv -f $backup_path/gunicorn_port.service /etc/systemd/system/
                if [ "0" != "$?" ]; then
                    echo -e "\e[1;31mRoll back gunicorn_port.service failed\e[0m"
                    exit 1
                fi
            else
                echo -e "\e[1;31mRollback failed. The backup gunicorn_port.service does not exist.\e[0m"
            fi
            if [[ "$os_type" == "redhat" ]]; then
                if [ -e $backup_path/nginx_port.service ]; then
                    mv -f $backup_path/nginx_port.service /usr/lib/systemd/system/
                    if [ "0" != "$?" ]; then
                        echo -e "\e[1;31mFailed to roll back nginx_port.service.\e[0m"
                        exit 1
                    fi
                else
                    echo -e "\e[1;31mRollback failed. The backup nginx_port.service does not exist.\e[0m"
                fi
            else
                if [ -e $backup_path/nginx_port.service ]; then
                    mv -f $backup_path/nginx_port.service /etc/systemd/system/
                    if [ "0" != "$?" ]; then
                        echo -e "\e[1;31mFailed to roll back nginx_port.service.\e[0m"
                        exit 1
                    fi
                else
                    echo -e "\e[1;31mRollback failed. The backup nginx_port.service does not exist.\e[0m"
                fi
            fi
        fi
        #重启nginx，django服务
        if [ -f /.dockerenv ]; then
            sh $install_path/tools/webui/script_port/service_nginx.sh start "$UPGRADE_PATH"
            cd $install_path/tools/portal/resource
            $install_path/tools/venv/bin/python3 $install_path/tools/venv/bin/gunicorn -c resource/gunicorn-conf.py  resource.wsgi >/dev/null &
        else
            if [[ $os_name =~ (CentOS .* 6\.)  || $os_name =~ (Red Hat .* 6\.) ]]; then
                service_set_nginx "start" "port"
                service_set_django "start" "port"
                set_port_service_time
            else
                systemctl enable nginx_port.service
                systemctl enable gunicorn_port.service
                systemctl daemon-reload
                systemctl start nginx_port.service
                systemctl start gunicorn_port.service
            fi
        fi
        #删除新版本的备份的目录
        rm -rf $install_path/../tools_portbak
        rm -rf $install_path/../config_portbak
        rm -rf $install_path/../resource_portbak
        rm -rf $backup_dir
    fi

    if [[ $1 == "cmd" ]]; then
        rm -rf $install_path
        if [ -d $backup_path/portadv ]; then
            cp -rfp $backup_path/portadv $UPGRADE_PATH
            if [ "0" != "$?" ]; then
                echo -e "\e[1;31mFailed to restore portadv during the rollback.\e[0m"
                exit 1
            fi
        else
            echo -e "\e[1;31mRollback failed. The backup portadv dir does not exist.\e[0m"
        fi
        rm -rf $backup_dir
    fi
    echo -e "\e[1;32mRollback finished.\e[0m"
}

roll_back_before_install(){
    echo -e "\e[1;32mStart rollback.\e[0m"
    if [ "$1" == "web" ]; then
        ng=$(ps aux | grep "[-c] $UPGRADE_PATH/portadv/tools/nginx-install/conf/nginx_port.conf" | awk '{print $2}')
        gun=$(ps aux | grep "[-c] resource/gunicorn-conf.py" | awk '{print $2}')

        if [ -f /.dockerenv ]; then
            if test -z "$gun"; then
                if [ -e $backup_path/gunicorn_port.service ]; then
                    mv -f $backup_path/gunicorn_port.service /etc/systemd/system/
                    if [ "0" != "$?" ]; then
                        echo -e "\e[1;31mFailed to roll back gunicorn_port.service.\e[0m"
                        exit 1
                    fi
                fi
                cd $install_path/tools/portal/resource
                $install_path/tools/venv/bin/python3 $install_path/tools/venv/bin/gunicorn -c resource/gunicorn-conf.py  resource.wsgi >/dev/null &

            else
                echo -e "\e[1;32mService gunicorn_port is already runing.\e[0m"
            fi

            if test -z "$ng"; then
                if [[ "$os_type" == "redhat" ]]; then
                    if [ -e $backup_path/nginx_port.service ]; then
                        mv -f $backup_path/nginx_port.service /usr/lib/systemd/system/
                        if [ "0" != "$?" ]; then
                            echo -e "\e[1;31mFailed to roll back nginx_port.service.\e[0m"
                            exit 1
                        fi
                    fi
                    sh $install_path/tools/webui/script_port/service_nginx.sh start "$UPGRADE_PATH"
                else
                    if [ -e $backup_path/nginx_port.service ]; then
                        mv -f $backup_path/nginx_port.service /etc/systemd/system/
                        if [ "0" != "$?" ]; then
                            echo -e "\e[1;31mFailed to roll back nginx_port.service.\e[0m"
                            exit 1
                        fi
                    fi
                    sh $install_path/tools/webui/script_port/service_nginx.sh start "$UPGRADE_PATH"
                fi
            else
                echo -e "\e[1;32mService nginx_port is already runing.\e[0m"
            fi
        else
            if [[ $os_name =~ (CentOS .* 6\.)  || $os_name =~ (Red Hat .* 6\.) ]]; then
                if test -z "$gun"; then
                    if [ -e $backup_path/gunicorn_port ]; then
                        mv -f $backup_path/gunicorn_port /etc/init.d/
                        if [ "0" != "$?" ]; then
                            echo -e "\e[1;31mFailed to roll back gunicorn_port.\e[0m"
                            exit 1
                        fi
                    fi
                    service_set_django "start" "port"
                else
                    echo -e "\e[1;32mService gunicorn_port is already runing.\e[0m"
                fi

                if test -z "$ng"; then
                    if [ -e $backup_path/nginx_port ]; then
                        mv -f $backup_path/nginx_port /etc/init.d/
                        if [ "0" != "$?" ]; then
                            echo -e "\e[1;31mFailed to roll back nginx_port.\e[0m"
                            exit 1
                        fi
                    fi
                    service_set_nginx "start" "port"
                else
                    echo -e "\e[1;32mService nginx_port is already runing.\e[0m"
                fi

                set_port_service_time
            else

                if test -z "$gun"; then
                    if [ -e $backup_path/gunicorn_port.service ]; then
                        mv -f $backup_path/gunicorn_port.service /etc/systemd/system/
                        if [ "0" != "$?" ]; then
                            echo -e "\e[1;31mFailed to roll back gunicorn_port.service.\e[0m"
                            exit 1
                        fi
                    fi
                    systemctl enable gunicorn_port
                    systemctl daemon-reload
                    systemctl start gunicorn_port

                else
                    echo -e "\e[1;32mService gunicorn_port is already runing.\e[0m"
                fi

                if test -z "$ng"; then
                    if [[ "$os_type" == "redhat" ]]; then
                        if [ -e $backup_path/nginx_port.service ]; then
                            mv -f $backup_path/nginx_port.service /usr/lib/systemd/system/
                            if [ "0" != "$?" ]; then
                                echo -e "\e[1;31mFailed to roll back nginx_port.service.\e[0m"
                                exit 1
                            fi
                        fi
                        systemctl enable nginx_port
                        systemctl daemon-reload
                        systemctl start nginx_port
                    else
                        if [ -e $backup_path/nginx_port.service ]; then
                            mv -f $backup_path/nginx_port.service /etc/systemd/system/
                            if [ "0" != "$?" ]; then
                                echo -e "\e[1;31mFailed to roll back nginx_port.service.\e[0m"
                                exit 1
                            fi
                        fi
                        systemctl enable nginx_port
                        systemctl daemon-reload
                        systemctl start nginx_port
                    fi
                else
                    echo -e "\e[1;32mService nginx_port is already runing.\e[0m"
                fi
            fi
        fi
        rm -rf $backup_dir
    fi

    if [[ $1 == "cmd" ]]; then
        rm -rf $backup_dir
    fi
    echo -e "\e[1;32mRollback finished.\e[0m"
}

restore_cert_and_key(){
    bak_ca_cert=$backup_path/portadv/tools/webui/ca.crt
    new_ca_cert=$install_path/tools/webui/ca.crt
    bak_cert_key=$backup_path/portadv/tools/webui/cert.key
    bak_cert_pem=$backup_path/portadv/tools/webui/cert.pem
    bak_dhparam_pem=$backup_path/portadv/tools/webui/dhparam.pem
    bak_zeus=$backup_path/portadv/tools/webui/zeus
    bak_crypto_so=$backup_path/portadv/tools/webui/dep_port_crypto.so
    bak_common=$backup_path/portadv/tools/webui/common

    if [ -e $bak_ca_cert ]; then
        cp -rfp $bak_ca_cert $install_path/tools/webui/
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to restore the ca cert file.\e[0m"
            roll_back web
            exit 1
        fi
    else
        if [ -e $new_ca_cert ]; then
            rm -f $new_ca_cert
        fi
    fi

    if [[ -e $bak_cert_key && -e $bak_cert_pem ]]; then
        cp -rfp $bak_cert_key $bak_cert_pem $install_path/tools/webui/
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to restore the cert file.\e[0m"
            roll_back web
            exit 1
        fi
    else
        echo -e "\e[1;31mBackup cert file does not exist. Please check the system.\e[0m"
        roll_back web
        exit 1
    fi

    if [ -e $bak_dhparam_pem ]; then
        cp -rfp $bak_dhparam_pem $install_path/tools/webui/
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to restore the dhparam file.\e[0m"
            roll_back web
            exit 1
        fi
    else
        echo -e "\e[1;31mBackup dhparam file does not exist. Please check the system.\e[0m"
        roll_back web
        exit 1
    fi

    if [ -e $bak_zeus ]; then
        cp -rfp $bak_zeus $install_path/tools/webui/
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to restore the zeus component file.\e[0m"
            roll_back web
            exit 1
        fi
    else
        echo -e "\e[1;31mBackup zeus component file does not exist. Please check the system.\e[0m"
        roll_back web
        exit 1
    fi

    if [ -e $bak_crypto_so ]; then
        cp -rfp $bak_crypto_so $install_path/tools/webui/
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to restore the so file.\e[0m"
            roll_back web
            exit 1
        fi
    else
        echo -e "\e[1;31mBackup so file does not exist. Please check the system.\e[0m"
        roll_back web
        exit 1
    fi

    if [ -d $bak_common ]; then
        cp -rfp $bak_common $install_path/tools/webui/
        if [ "0" != "$?" ]; then
            echo -e "\e[1;31mFailed to restore common dir.\e[0m"
            roll_back web
            exit 1
        fi
    else
        echo -e "\e[1;31mBackup common dir does not exist. Please check the system.\e[0m"
        roll_back web
        exit 1
    fi
}
```

