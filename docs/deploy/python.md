# Ptyhon 3.12.2

- 官网地址：https://www.python.org/
- 下载地址：https://www.python.org/ftp/python/

## 1. Windows

### 1.1 安装Ptyhon

![](../../assets/_images/deploy/python/1.png)

![](../../assets/_images/deploy/python/2.png)

![](../../assets/_images/deploy/python/3.png)

![](../../assets/_images/deploy/python/4.png)

![](../../assets/_images/deploy/python/5.png)

![](../../assets/_images/deploy/python/6.png)

### 1.2 安装PyCharm

下载地址：https://www.jetbrains.com/zh-cn/pycharm/

![](../../assets/_images/deploy/python/10.png)

![](../../assets/_images/deploy/python/11.png)

![](../../assets/_images/deploy/python/12.png)

![](../../assets/_images/deploy/python/13.png)

![](../../assets/_images/deploy/python/14.png)

启动

![](../../assets/_images/deploy/python/20.png)

创建项目

![](../../assets/_images/deploy/python/21.png)

设置主题

![](../../assets/_images/deploy/python/22.png)

设置字体

![](../../assets/_images/deploy/python/23.png)

三方包管理

![](../../assets/_images/deploy/python/24.png)

![](../../assets/_images/deploy/python/25.png)

下载三方包

![](../../assets/_images/deploy/python/26.png)

### 1.3 安装Anaconda3

下载地址：https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/Anaconda3-2021.05-Windows-x86_64.exe

#### 1.3.1 设置环境变量

```java
PATH
C:\ProgramData\Anaconda3
C:\ProgramData\Anaconda3\Scripts
C:\ProgramData\Anaconda3\Library\bin
C:\ProgramData\Anaconda3\Library\mingw-w64
```

#### 1.3.2 更换服务器源

打开`Anaconda Prompt`程序，执行`conda config --set show_channel_urls yes`

记事本打开`C:\Users\用户名\.condarc`文件, 将如下内容替换全部保存

```shell
channels:
  - defaults
show_channel_urls: true
default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  msys2: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  bioconda: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  menpo: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  simpleitk: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
```

#### 1.3.3 常用命令

```bash
conda info -e                       # 查看当前系统中创建的虚拟环境，自带一个base环境
conda env list                      # 查看已创建的环境
conda create -n name python=3.8     # 创建名为name、Python版本为x.x的虚拟环境
conda activate name                 # 激活名为name的环境
conda install package_name          # 在当前环境中安装包
conda deactivate                    # 退出虚拟环境
conda remove -n name --all          # 删除名为name的虚拟环境
pip install pyhive pyspark jieba -i https://pypi.tuna.tsinghua.edu.cn/simple    # 在虚拟环境内安装包
```

## 2. Linux

### 2.1 安装Ptyhon

#### 2.1.1 安装依赖

```bash
yum install wget zlib-devel bzip2-devel openssl openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gcc make zlib zlib-devel libffi-devel -y
```

#### 2.1.2 下载编译

[Python requires a OpenSSL 1.1.1 or newer](deploy/haproxy?id=_15-安装openssl)
```bash
cd /opt/software
wget https://www.python.org/ftp/python/3.12.2/Python-3.12.2.tgz
tar -xvf Python-3.12.2.tgz
cd Python-3.12.2
# 配置
./configure --prefix=/usr/local/python3.12.2 --with-openssl=/usr/local/openssl
# 编译
make && make install
```

#### 2.1.3 设置环境变量

```bash
vi /etc/profile
```

```conf
export PYTHON_HOME=/usr/local/python3.12.2
export PATH=$PYTHON_HOME/bin:$PATH
```

使生效
```bash
source /etc/profile
```

#### 2.1.4 创建软连接

```bash
rm -f /usr/bin/python
ln -s /usr/local/python3.12.2/bin/python3.12 /usr/bin/python
# 创建软链接后，会破坏yum程序的正常使用，需要修改以下两个文件
/usr/bin/yum
/usr/libexec/urlgrabber-ext-down
```

使用vi编辑器，将这2个文件的第一行，从
```conf
#!/usr/bin/python
```
修改为：
```conf
#!/usr/bin/python2
```

### 2.2 设置虚拟环境

#### 2.2.1 配置pip3源

```bash
mkdir ~/.pip
touch ~/.pip/pip.conf
vi ~/.pip/pip.conf
```

添加配置

```conf
[global]
index-url=http://mirrors.aliyun.com/pypi/simple
[install]
trusted-host=mirrors.aliyun.com
```

#### 2.2.2 安装虚拟环境

```bash
pip3 install virtualenv
```

#### 2.2.3 创建虚拟环境

```bash
mkdir /data/python3/code -p
cd /data/python3/code
# 创建环境
virtualenv --python=python3 jmp_venv1
# 激活环境
source /data/python3/code/jmp_venv1/bin/activate
# 退出环境
deactivate
```

## 3. Flask Web

### 3.1 Hello World

1. 生成requirements.txt

```bash
vi requirements.txt
```

编辑内容

```txt
flask==2.0.2
Werkzeug==2.2.2
```

2. 安装依赖

```bash
pip install -r requirements.txt
```

3. Hello World

创建资源路径和模板路径
```bash
cd code
mkdir static
mkdir templates
```

```python
from flask import Flask

app = Flask(__name__, static_folder="static", template_folder="templates")
 
@app.route('/')
def hello_world():
    return 'Hello World!'
 
if __name__ == '__main__':
    app.run(port=5000, debug=True, host='0.0.0.0')
```

4. 启动服务

```bash
python main.py
```

### 3.2 获取URL参数

服务端
```python
from flask import Flask, request

app = Flask(__name__)
 
@app.route("/")
def hello():
    print(request.args.__str__())
    print(request.args.get('info'))
    print(request.args.get('user', 'admin'))
    print(request.args.getlist('p'))
    return "Hello " + request.args.get('user', 'admin')
 
if __name__ == '__main__':
    app.run(port=5000, debug=True)
```

客户端
```bash
pip3 install requests
```

```python
import requests

# get请求
r = requests.get('http://127.0.0.1:5000/?info=hello&p=123&p=abc')
print(r.text)
```


### 3.3 获取POST表单参数

服务端
```python
from flask import Flask, request

app = Flask(__name__)
 
# post请求，表单提交
@app.route('/register', methods=['POST'])
def register():
    print(request.headers)
    # print(request.stream.read()) # 不要用，否则下面的form取不到数据
    print(request.form)
    print(request.form['name'])
    print(request.form.get('name'))
    print(request.form.getlist('name'))
    print(request.form.get('password', default='123456'))
    return 'success'
 
if __name__ == '__main__':
    app.run(port=5000, debug=True)
```

客户端
```python
import requests

# POST表单提交
user_info = {'name': 'xzh', 'password': '123'}
r = requests.post("http://127.0.0.1:5000/register", data=user_info)
print(r.text)
```


### 3.4 获取JSON参数

服务端
```python
from flask import Flask, request, Response, jsonify
import json

app = Flask(__name__)
 
# post请求，json接收和返回
@app.route('/add', methods=['POST'])
def add():
    print(type(request.json))
    print(request.json)
    result = request.json['a'] + request.json['b']
    result = {'sum': request.json['a'] + request.json['b']}
    # return jsonify(result)
    return Response(json.dumps(result),  mimetype='application/json')
 
if __name__ == '__main__':
    app.run(port=5000, debug=True)
```

客户端
```python
import requests

# JSON请求
json_data = {'a': 1, 'b': 2}
r = requests.post("http://127.0.0.1:5000/add", json=json_data)
print(r.text)
```

### 3.5 上传文件

服务端
```python
from flask import Flask, request
from werkzeug.utils import secure_filename
import os

app = Flask(__name__)

# 文件上传目录
app.config['UPLOAD_FOLDER'] = 'static/uploads/'
# 支持的文件格式
app.config['ALLOWED_EXTENSIONS'] = {'png', 'jpg', 'jpeg', 'gif'}  # 集合类型
# 判断文件名是否是我们支持的格式
def allowed_file(filename):
    return '.' in filename and \
           filename.rsplit('.', 1)[1] in app.config['ALLOWED_EXTENSIONS']

# 上传文件
@app.route('/upload', methods=['POST'])
def upload():
    upload_file = request.files['image']
    if upload_file and allowed_file(upload_file.filename):
        filename = secure_filename(upload_file.filename)
        # 将文件保存到 static/uploads 目录，文件名同上传时使用的文件名
        upload_file.save(os.path.join(app.root_path, app.config['UPLOAD_FOLDER'], filename))
        return 'info is '+request.form.get('info', '')+'. success'
    else:
        return 'failed'
 
if __name__ == '__main__':
    app.run(port=5000, debug=True)
```


客户端
```python
import requests

# 文件上传
file_data = {'image': open('head.jpg', 'rb')}
user_info = {'info': 'zhangsan'} 
r = requests.post("http://127.0.0.1:5000/upload", data=user_info, files=file_data)
print(r.text)
```

### 3.6 Restful URL

服务端
```python
from flask import Flask
 
app = Flask(__name__)
 
@app.route('/user/<username>')
def user(username):
    print(username)
    print(type(username))
    return 'hello ' + username
 
 
@app.route('/user/<username>/friends')
def user_friends(username):
    print(username)
    print(type(username))
    return 'hello ' + username
 
 
if __name__ == '__main__':
    app.run(port=5000, debug=True)
```


访问地址：http://127.0.0.1:5000/user/admin

### 3.7 参数类型转换

服务端
```python
from flask import Flask
 
app = Flask(__name__)
 
@app.route('/page/<int:num>')
def page(num):
    print(num)
    print(type(num))
    return 'hello world'
 
 
if __name__ == '__main__':
    app.run(port=5000, debug=True)
```

访问地址：http://127.0.0.1:5000/page/100

### 3.8 自定义路由转换器

自定义的转换器是一个继承werkzeug.routing.BaseConverter的类，修改to_python和to_url方法即可。to_python方法用于将url中的变量转换后供被`@app.route包装的函数使用`，to_url方法用于flask.url_for`中的参数转换

服务端
```python
from flask import Flask, url_for
from werkzeug.routing import BaseConverter
 
class MyIntConverter(BaseConverter):
 
    def __init__(self, url_map):
        super(MyIntConverter, self).__init__(url_map)
 
    def to_python(self, value):
        return int(value) * 2
 
#    def to_url(self, value):
#        return int(value) * 10
 
app = Flask(__name__)
app.url_map.converters['my_int'] = MyIntConverter
 
@app.route('/')
def hello_world():
    return 'hello world'
 
@app.route('/page/<my_int:num>')
def page(num):
    print(num)
    print(url_for('page', num=55))
    return 'hello world'
 
if __name__ == '__main__':
    app.run(port=5000, debug=True)
```

访问地址：http://127.0.0.1:5000/page/100

### 3.9 url_for生成链接

服务端
```python
from flask import Flask, url_for
 
app = Flask(__name__)
 
@app.route('/')
def hello_world():
    pass
 
@app.route('/user/<name>')
def user(name):
    pass
 
@app.route('/page/<int:num>')
def page(num):
    pass
 
@app.route('/test')
def test():
    print(url_for('hello_world'))
    print(url_for('user', name='zhangsan'))
    print(url_for('page', num=1, q='hadoop mapreduce 10%3'))
    print(url_for('static', filename='uploads/01.jpg'))
    return 'Hello'
 
if __name__ == '__main__':
    app.run(debug=True)
```

访问地址：http://127.0.0.1:5000/test

### 3.10 重定向

服务端
```python
from flask import Flask, url_for, redirect
 
app = Flask(__name__)
 
@app.route('/')
def hello_world():
    return 'hello world'
 
@app.route('/test1')
def test1():
    print('this is test1')
    return redirect(url_for('test3'))
 
@app.route('/test2')
def test3():
    print('this is test2')
    return 'this is test2'
 
if __name__ == '__main__':
    app.run(debug=True)
```

访问地址：http://127.0.0.1:5000/test1

### 3.11 Jinja2模板引擎

1. 创建并编辑模板

```bash
vi templates/default.html
```

```html
<html>
<head>
    <title>
        {% if page_title %}
            {{ page_title }}
        {% endif %}
    </title>
</head>

<body>
    {% block user %}{% endblock %}
</body>
```

标签中定义了一个名为`user`的block，用来被其他模板文件继承。

2. 创建并编辑用户模板

```bash
vi templates/user_info.html
```
 
```html
{% extends "default.html" %}

{% block user %}
    {% for key in user_info %}
         {{ key }}: {{ user_info[key] }} <br>
    {% endfor %}
{% endblock %}
```

3. 服务端

```python
from flask import Flask, render_template

app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'hello world'

@app.route('/user')
def user():
    user_info = {
        'name': 'xzh',
        'email': 'xcg992224@163.com',
        'age':0,
        'github': 'https://github.com/xzh-net'
    }
    return render_template('user_info.html', page_title='xzh\'s info', user_info=user_info)
 
 
if __name__ == '__main__':
    app.run(port=5000, debug=True, host='0.0.0.0')
```

访问地址：http://127.0.0.1:5000/user

### 3.12 全局异常404

### 3.13 用户会话

### 3.14 使用Cookie








