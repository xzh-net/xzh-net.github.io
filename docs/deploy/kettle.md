# pdi-ce-7.1.0.0-12

Kettle是一款开源的ETL工具，纯java编写，可以在Window、Linux、Unix上运行，绿色无需安装，数据抽取高效稳定
 - 勺子(Spoon.bat/spoon.sh) :是1个图形化的界面，可以让我们用图形化的方式开发转换和作业。windows选择.bat; Linux选择.sh。
 - 煎锅(Pan.bat/pan.sh) : 利用Pan可以用命令行的形式调用Trans。
 - 厨房(Kitchen batkitchen.sh) : 利用Kitchen可以使用命令行调用job
 - 菜单(Carte.bat/ Carte.sh): Carte是一 个轻量级的Web容器，用于建立专用、远程的ETL Server。

下载地址： https://sourceforge.net/projects/pentaho/files/Data%20Integration

## 1. 安装

### 1.1 DB连接

下载完成解压到任意路径，打开文件夹，找到Spoon.bat，创建桌面快捷方式，打开

![](../../assets/_images/deploy/kattle/1.png)

在文件->新建装换，新建转换后在左边的主对象树中建立DB连接用以连接数据库

![](../../assets/_images/deploy/kattle/2.png)

连接oralce 

![](../../assets/_images/deploy/kattle/3.png)


### 1.2 导出Excel

#### 1.2.1 表输入
   
![](../../assets/_images/deploy/kattle/5.png)

#### 1.2.2 Excel输出

![](../../assets/_images/deploy/kattle/6.png)

设置保存位置和导出字段

#### 1.2.3 关联

按住【Shift】，鼠标左键点击【表输入】，向右拉到【Excel输出】上即可连接二者然后【Ctrl+S】保存转换

### 1.3 调用存储过程

#### 1.3.1 sql

```sql
CREATE TABLE TEST(
  ID   VARCHAR2(100),
  NAME VARCHAR2(100),
  YEAR NUMBER
)

CREATE OR REPLACE PROCEDURE TEST_KATTLE(I_NAME VARCHAR2, I_YEAR NUMBER) AS
  P_ID VARCHAR2(100);
BEGIN
  SELECT SYS_GUID() INTO P_ID FROM DUAL;
  INSERT INTO TEST (ID, NAME, YEAR) VALUES (P_ID, I_NAME, I_YEAR);
END;
```

#### 1.3.2 添加控件

在右侧【核心对象】中搜索【表输入】，【调用DB存储过程】将其拖到新创建的转换中

![](../../assets/_images/deploy/kattle/8.png)

```sql
SELECT '${P_NAME}' as I_NAME,'${P_YEAR}' as I_YEAR FROM DUAL
```

![](../../assets/_images/deploy/kattle/7.png)


#### 1.3.3 设置转换

右键点击转换空白处，打开【转换设置】->【命名参数】， 配置调用存储过程要使用的输入参数（与【表输入】的SQL语句中的变量参数一致）

![](../../assets/_images/deploy/kattle/9.png)

![](../../assets/_images/deploy/kattle/10.png)


### 1.4 定时调度

#### 1.4.1 转换

![](../../assets/_images/deploy/kattle/11.png)

![](../../assets/_images/deploy/kattle/12.png)

#### 1.4.2 运行

![](../../assets/_images/deploy/kattle/13.png)