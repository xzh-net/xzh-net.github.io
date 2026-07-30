# Windows 10 Enterprise LTSC 2019 (x64) 

## 1. 系统安装

1. 选择语言

![](../../assets/_images/deploy/win10/1.png)

2. 完全安装

![](../../assets/_images/deploy/win10/2.png)

3. 磁盘格式化

![](../../assets/_images/deploy/win10/3.png)

4. 等待安装

![](../../assets/_images/deploy/win10/4.png)

5. 区域设置

![](../../assets/_images/deploy/win10/5.png)

6. 设置键盘布局，然后单击跳过

![](../../assets/_images/deploy/win10/6.png)

![](../../assets/_images/deploy/win10/7.png)

7. 网络设置

![](../../assets/_images/deploy/win10/8.png)

8. 创建用户

![](../../assets/_images/deploy/win10/9.png)

9. 隐私设置

![](../../assets/_images/deploy/win10/10.png)

10. 进入桌面

![](../../assets/_images/deploy/win10/11.png)

![](../../assets/_images/deploy/win10/12.png)

11. 网络设置

![](../../assets/_images/deploy/win10/13.png)

12. 激活

下载地址：https://github.com/massgravel/Microsoft-Activation-Scripts

## 2. 软件工具

### 2.1 VirtualBox重置UUID

```bash
cd C:\Program Files\Oracle\VirtualBox
VBoxManage internalcommands sethduuid "D:\VirtualBox VMs\centos7\centos7.vdi"
```

### 2.2 MySQL解压版安装

#### 2.2.1 下载

- 下载地址：https://dev.mysql.com/downloads/mysql/

#### 2.2.2 配置环境变量

新建MYSQL_HOME
```
MYSQL_HOME
D:\mysql-8.0.36-winx64
```

编辑PATH
```
PATH
%MYSQL_HOME%\bin
```

设置成功以后使用命令`mysql`进行验证，如果提示Can't connect to MySQL server on 'localhost'则证明添加成功。如果提示mysql不是内部或外部命令，也不是可运行的程序或批处理文件则表示添加添加失败，请重新检查步骤并重试。



#### 2.2.3 初始化配置

在bin的同级目录下新建一个my.ini文件

```ini
[client]
# 设置mysql客户端默认字符集
default-character-set=utf8

[mysql]
# 设置mysql客户端默认字符集
default-character-set=utf8mb4

[mysqld]
# 设置mysql客户端连接服务端时默认使用的端口3306
port = 3306

# 设置mysql的安装目录
basedir = D:\\mysql-8.0.36-winx64

# 设置mysql数据库的数据的存放目录
datadir = D:\\mysql-8.0.36-winx64\\data

# 允许最大连接数
max_connections=200

# 允许连接失败的次数。这是为了防止有人从该主机试图攻击数据库系统
max_connect_errors=10

# 服务端使用的字符集默认为8比特编码的latin1字符集【mysql8.0】
character_set_server = utf8mb4

# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB

# 默认使用“mysql_native_password”插件认证【mysql8.0】
# default_authentication_plugin=mysql_native_password​​
authentication_policy=*

# 跳过安全检查,如果跳过，可能不能执行修改用户密码sql语句
#skip-grant-tables

#开启查询缓存
#explicit_defaults_for_timestamp=true

# 创建模式 NO_AUTO_CREATE_USER再MYSQL8.0中已经被移除，不能再8.0以上版本配置【mysql8.0】
# sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES 
# sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
```

以管理员身份，运行命令行窗口：

```bash
mysqld --initialize-insecure
# 或者下面方式，生成随机密码
mysqld --initialize --user=mysql --console
```

如果没有出现报错信息，则证明data目录初始化没有问题，此时再查看MySQL目录下已经有data目录生成。

#### 2.2.4 注册服务并启动

```bash
mysqld -install     # 以管理员身份运行
net start mysql     # 启动mysql服务
net stop mysql      # 停止mysql服务
```

#### 2.2.5 登录MySQL

修改默认密码

```bash
mysqladmin -u root password 123456
```

登录

```bash
mysql -uroot -p123456
# mysql -u用户名 -p密码 -h要连接的mysql服务器的ip地址(默认127.0.0.1) -P端口号(默认3306)
```

#### 2.2.6 卸载MySQL

```bash
net stop mysql
mysqld -remove mysql
```

### 2.3 Notepad 主题

设置 `->` 语言格式设置

![](../../assets/_images/deploy/win10/notepad.png)

### 2.4 Xshell 隧道

![](../../assets/_images/deploy/win10/xshell.png)

### 2.5 Nginx 控制台

创建脚本文件 nginx.bat，用于启动、停止和查看Nginx状态。（文件格式：ANSI，否则乱码）

```bash
@echo off
chcp 936 >nul
setlocal enabledelayedexpansion

:: ============================================================
:: 请根据实际安装路径修改 NGINX_HOME 变量
:: ============================================================
set NGINX_HOME=D:\tools\openresty-1.27.1.2-win64
set NGINX_EXE=%NGINX_HOME%\nginx.exe

if not exist "%NGINX_EXE%" (
    echo [错误]：未找到 nginx.exe，请检查 NGINX_HOME 路径。
    pause
    exit /b 1
)

cd /d "%NGINX_HOME%"

:: ============================================================
:: 交互式菜单
:: ============================================================
:MENU
cls
echo ============================================
echo        Nginx 控制台 (OpenResty)
echo ============================================
echo  1. 启动 / 平滑重载配置
echo  2. 强制停止 Nginx
echo  3. 查看运行状态
echo  0. 退出
echo ============================================
set "choice="
set /p choice=请输入选项并按回车: 

if "%choice%"=="1" goto START_OR_RELOAD
if "%choice%"=="2" goto FORCE_STOP
if "%choice%"=="3" goto CHECK_STATUS
if "%choice%"=="0" exit /b 0
echo [提示]：输入无效，请重新选择。
timeout /t 2 >nul
goto MENU

:: ============================================================
:: 1. 启动或平滑重载
:: ============================================================
:START_OR_RELOAD
cls
echo 正在检测 Nginx 运行状态...
tasklist /FI "IMAGENAME eq nginx.exe" 2>NUL | find /I /N "nginx.exe" >NUL

if errorlevel 1 (
    :: 未运行，先校验配置再启动
    echo [信息]：Nginx 未运行，正在校验配置...
    "%NGINX_EXE%" -t >NUL 2>&1
    if errorlevel 1 (
        echo [错误]：配置文件存在语法错误，无法启动！
        echo ----------------------------------------
        "%NGINX_EXE%" -t
        echo ----------------------------------------
    ) else (
        echo [信息]：配置校验通过，正在启动 Nginx...
        start "" "%NGINX_EXE%"
        timeout /t 2 /nobreak >nul
        tasklist /FI "IMAGENAME eq nginx.exe" 2>NUL | find /I /N "nginx.exe" >NUL
        if errorlevel 1 (
            echo [错误]：启动失败，请检查 Nginx 错误日志。
        ) else (
            echo [成功]：Nginx 已成功启动。
        )
    )
) else (
    :: 正在运行，准备平滑重载
    echo [信息]：Nginx 正在运行，准备重载配置...
    
    :: 【核心优化】重载前再次检查配置语法，防止错误配置导致服务异常
    "%NGINX_EXE%" -t >NUL 2>&1
    if errorlevel 1 (
        echo [警告]：配置文件存在语法错误，已取消重载！
        echo ----------------------------------------
        "%NGINX_EXE%" -t
        echo ----------------------------------------
    ) else (
        "%NGINX_EXE%" -s reload
        if errorlevel 1 (
            echo [错误]：重载失败，请检查错误日志。
        ) else (
            echo [成功]：配置已成功平滑重载。
        )
    )
)
echo.
echo 按任意键返回菜单...
pause >nul
goto MENU

:: ============================================================
:: 2. 强制停止 (立即终止进程)
:: ============================================================
:FORCE_STOP
cls
echo 正在强制停止 Nginx...
"%NGINX_EXE%" -s stop
if errorlevel 1 (
    echo [警告]：常规停止失败，尝试强制结束进程...
    taskkill /F /IM nginx.exe >NUL 2>&1
    if errorlevel 1 (
        echo [信息]：未找到运行中的 Nginx 进程。
    ) else (
        echo [成功]：Nginx 进程已强制终止。
    )
) else (
    echo [成功]：Nginx 已停止。
)
echo.
echo 按任意键返回菜单...
pause >nul
goto MENU

:: ============================================================
:: 3. 查看运行状态
:: ============================================================
:CHECK_STATUS
cls
echo 正在检查 Nginx 运行状态...
echo ============================================
tasklist /FI "IMAGENAME eq nginx.exe" 2>NUL | find /I /N "nginx.exe" >NUL
if errorlevel 1 (
    echo [信息]：Nginx 当前未运行。
) else (
    echo [信息]：Nginx 正在运行中。
    echo.
    echo 进程列表：
    tasklist /FI "IMAGENAME eq nginx.exe"
)
echo ============================================
echo.
echo 按任意键返回菜单...
pause >nul
goto MENU
```