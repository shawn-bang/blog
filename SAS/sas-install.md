## SAS Install Proc

- ssh客户端保持连接，安装中途不能中断
```
putty -> Connection -> Seconds between keepalives
```

- 检查安装手册中提到的依赖包(如果没有，挂在镜像yum源进行安装(64 bit))
```shell
yum list | grep glibc
yum list | grep libXp
yum list | grep libXmu
yum list | grep numactl
python --version
```

- 检查hosts配置(**特别注意**)
```
/etc/hosts:
#127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 -- 注掉
#::1         localhost localhost.localdomain localhost6 localhost6.localdomain6 -- 注掉
10.124.109.180  TES-AFS --  只能是字母，中划线,不能包含数字，下划线等
10.124.109.190  DVAFSDB -- 数据库hostname，必须配置
  <br>
如果是多台同用途SAS Server，hostname必须必须要设置为同一个(找机会确认原因并标注)
```

- 创建用户并设置密码
```shell
groupadd sas
useradd -g sas sas
useradd -g sas sasdemo
useradd -g sas sassrv
password sas sas@123456
password sasdemo sas@123456
password sassrv sas@123456
```

- 上传安装介质并更改介质所在文件夹所属用户及权限
```
chown -R sas:sas /home/depot
chmod -R 755 /home/depot
```

- 检查字符串
```
查看系统字符集为 : en_US.UTF-8(支持显示中文，但不能输入中文)
目测不需要调整 : zh_CN.UTF-8(支持显示中文，支持输入中文)
```

- 检查系统限制情况，有必要做相应调整
```
/etc/systemd/system.conf
追加内容：
DefaultLimitNOFILE=350000
DefaultLimitNPROC=65536
<br>
/etc/security/limits.d/20-nproc.conf
调整为为：
*          soft    nproc     65536
root       soft    nproc     unlimited
上述配置 需要reboot生效
```

- 彻底关闭防火墙
```shell
systemctl disable firewalld
systemctl status firewalld
```

- 建立安装配置目录
```shell
mkdir /home/sashome
mkdir /home/sasconfig

chown -R sas:sas /home/sashome
chmod -R 755 /home/sashome

chown -R sas:sas /home/sasconfig
chmod -R 755 /home/sasconfig
```

- 数据库客户端配置(**Schema Name 指的是用户名**)
```
以Oracle为例：
1.安装Oracle对应服务器版本的客户端，配置环境变量
2.配置/etc/hosts
  追加内容：数据库IP “hostname”
3.在sas用户可见的目录下准备好 ojdbc jar
```

- 安装用户sas下配置环境变量
```shell
#export PATH
export ORACLE_BASE=/u01/app/oracle;
export ORACLE_HOME=$ORACLE_BASE/product/11.2.0/db_1;
export NLS_LANG=AMERICAN_AMERICA.ZHS16GBK;
export ORACLE_SID=;
export PATH=/usr/sbin:$PATH;
export PATH=$ORACLE_HOME/bin:$PATH:$HOME/.local/bin:$HOME/bin;
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib;
export CLASSPATH=$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib;
```
`注：修改$ORACLE_BASE文件夹目录权限775`

- 安装过程中特别注意项
1. SAS Metadate Server : Overwirte Backup Location

`Enter:Y`

2. SAS Base安装完后，有几步必要的操作需要做
	- 验证SAS Base命令行是否可用
	- root用户执行 `/sas/sashome/SASFoundation/9.4/utilities/bin/setuid.sh`
	- 初始化对应的SAS SNA数据 `/sas/sashome/SASFoundation/9.4/misc/snamva/dbmsc/ddl/sna_create_*.sql`

- 安装结束后的相关配置

1. QKB setting : 详细配置步骤待补充
2. saswork临时逻辑库目录指定
```
   ./sashome/SASFoundation/9.4/sasv9.cfg
   -WORK /tmp     调整为 -WORK /Specified-path
   -MEMSIZE 2G    调整为 -MEMSIZE Specified-Value
   -SORTSIZE 1G   调整为 -SORTSIZE Specified-Value
   -WORKPERMS 700 调整为 -WORKPERMS Specified-Value(recommend 775)
```
3. SAS WebServers如果有使用到了数据库jdbc加载数据，需要配置对应的jdbc数据源配置信息
   - ./sasconfig/Lev1/Web/WebAppServer/SASServer8_1/conf/context.xml  
   `Appending content : <ResourceLink type="javax.sql.DataSource" name="jdbc/Specified-Name" global="sas/jdbc/Specified-Name" />`
   - ./sasconfig/Lev1/Web/WebAppServer/SASServer8_1/conf/server.xml  
   `在Resource slice部分Appending content : 待补充...`
   
   - ./sasconfig/Lev1/Web/WebAppServer/SASServer8_1/conf/catalina.properties  
   `配置相关新增配置中引用的变量(jdbc properties...)`
   - ./sasconfig/Lev1/Web/WebAppServer/SASServer8_1/conf/Catalina/localhost/SASSNA.xml  
   `Appending content : <ResourceLink type="javax.sql.DataSource" name="jdbc/Specified-Name" global="sas/jdbc/Specified-Name" />`
   - restart SASServer8_1  
   `./sasconfig/Lev1/Web/WebAppServer/SASServer8_1/bin/tcruntime-ctl.sh stop`
   `./sasconfig/Lev1/Web/WebAppServer/SASServer8_1/bin/tcruntime-ctl.sh start`
