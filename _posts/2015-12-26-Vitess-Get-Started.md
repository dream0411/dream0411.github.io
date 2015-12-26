---
layout: default
title: Get started to Vitess
---

{{ page.title }}
===

# 启动前由于我本机环境中后面的vttablet步骤中的mysql初始化过程较慢，会导致后续步骤全部错误，我这里修改了vttablet-up.sh脚本并另存为vttablet-up2.sh，修改内容为68行，添加了较大的`wait_time`参数值：
    68  action="-logtostderr init -init_db_sql_file $init_db_sql_file -wait_time 5m"
# 启动docker容器，并暴露web管理端口15000和命令行管理端口15999，附加宿主机的bashrc配置，并修改了vttablet-up.sh脚本，增加初次mysql时的等待时间：
    sudo docker run -ti --name vitess -p 15000:15000 -p 15999:15999 -v `pwd`/examples/local/vttablet-up2.sh:/vt/src/github.com/youtube/vitess/examples/local/vttablet-up2.sh  vitess/base bash

# 进入脚本目录
    cd $VTROOT/src/github.com/youtube/vitess/examples/local

# 启动Zookeeper集群
    ./zk-up.sh

# 启动管理服务器
    ./vtctld-up.sh

# 启动包含3个tablet的集群
    ./vttablet-up2.sh


# 用做初始化
    $VTROOT/bin/vtctlclient -server localhost:15999 RebuildKeyspaceGraph test_keyspace
    $VTROOT/bin/vtctlclient -server localhost:15999 InitShardMaster -force test_keyspace/0 test-0000000100

# 验证
    $VTROOT/bin/vtctlclient -server localhost:15999 ListAllTablets test

# 初始化表
    $VTROOT/bin/vtctlclient -server localhost:15999 ApplySchema -sql "$(cat create_test_table.sql)" test_keyspace

# 备份
    $VTROOT/bin/vtctlclient -server localhost:15999 Backup test-0000000101

# 查看备份
    $VTROOT/bin/vtctlclient -server localhost:15999 ListBackups test_keyspace/0

# 启动vtgate
    ./vtgate-up.sh

# 启动一个示例客户端
    ./client.sh

# 停止集群
    ./vtgate-down.sh
    ./vttablet-down.sh
    ./vtctld-down.sh
    ./zk-down.sh

# 删除集群的数据
    cd $VTDATAROOT
    rm -rf *
