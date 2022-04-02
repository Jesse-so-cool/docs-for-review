
# 搭建主从
change master to master_host='主库IP地址', master_port=3306, master_user = '数据库账号', master_password='数据库密码', master_auto_position=1 ;

# 查看主从状态
show slave status\G ; 

# 启动主从
start slave ; 

# 停止主从
stop slave; 

# 清理主从信息
reset slave all; 

change master to master_host='172.25.32.94', master_port=3306, master_user = 'arbiter', master_password='2x9M5^jH*Y8bYdND', master_auto_position=1 ;

show slave hosts ;
