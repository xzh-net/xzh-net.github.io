# Ptyhon 3.12.2

- 官方网站：https://www.python.org/
- 下载地址：https://www.python.org/ftp/python/

## 1. Windows

### 1.1 安装 Ptyhon

![](../../assets/_images/deploy/python/1.png)

![](../../assets/_images/deploy/python/2.png)

![](../../assets/_images/deploy/python/3.png)

![](../../assets/_images/deploy/python/4.png)

![](../../assets/_images/deploy/python/5.png)

![](../../assets/_images/deploy/python/6.png)

### 1.2 安装 PyCharm

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

### 1.3 安装 Anaconda3

下载地址：https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/Anaconda3-2021.05-Windows-x86_64.exe

#### 1.3.1 配置环境变量

```java
PATH
C:\ProgramData\Anaconda3
C:\ProgramData\Anaconda3\Scripts
C:\ProgramData\Anaconda3\Library\bin
C:\ProgramData\Anaconda3\Library\mingw-w64
```

#### 1.3.2 设置全局镜像源

打开 `Anaconda Prompt` 程序，先运行 `conda config --set show_channel_urls yes` 生成配置文件。记事本打开 `C:\Users\用户名\.condarc` 文件, 将如下内容替换全部保存

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

#### 1.3.3 创建虚拟环境

```bash
# 创建虚拟环境
conda create -n vllm-dev python=3.12 -y
# 激活
conda activate vllm-dev
# 安装依赖
pip install modelscope
# 退出环境
conda deactivate
# 删除环境
conda env remove --name vllm-dev -y
```

## 2. Linux

### 2.1 安装 Ptyhon

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

#### 2.1.3 配置环境变量

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
```

如果是 `Centons7` 环境创建软链接后，会破坏 `YUM` 程序的正常使用，需要修改以下两个文件
- /usr/bin/yum
- /usr/libexec/urlgrabber-ext-down

分别将这两个文件的第一行，从 `#!/usr/bin/python` 修改为 `#!/usr/bin/python2`

### 2.2 安装 virtualenv（可选）

#### 2.2.1 在线安装

设置全局镜像源

```bash
mkdir ~/.pip
touch ~/.pip/pip.conf
vi ~/.pip/pip.conf
```

编辑内容

```conf
[global]
index-url=https://mirrors.aliyun.com/pypi/simple
[install]
trusted-host=mirrors.aliyun.com
```

执行安装

```bash
pip3 install virtualenv
```

#### 2.2.3 创建虚拟环境

```bash
# 创建虚拟环境
virtualenv --python=python3.12 vllm-dev
# 激活
source vllm-dev/bin/activate
# 安装依赖
pip install modelscope
# 退出环境
deactivate
# 删除环境
rm -rf vllm-dev 
```

### 2.3 安装 uv（可选）

#### 2.3.1 在线安装

使用curl下载脚本并执行
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

如果没有curl指令，可以使用wget安装

```bash
wget -qO- https://astral.sh/uv/install.sh | sh
```

需要下载指定版本使用下面的命令

```bash
curl -LsSf https://astral.sh/uv/0.10.2/install.sh | sh
```
 
执行后看到以下结果，表示安装成功。

```lua
root@ubuntu:~# curl -LsSf https://astral.sh/uv/install.sh | sh
downloading uv 0.10.2 x86_64-unknown-linux-gnu
no checksums to verify
installing to /root/.local/bin
  uv
  uvx
everything's installed!

To add $HOME/.local/bin to your PATH, either restart your shell or run:

    source $HOME/.local/bin/env (sh, bash, zsh)
    source $HOME/.local/bin/env.fish (fish)
```

配置环境变量
```bash
export UV_HOME=/root/.local
export PATH=$PATH:$UV_HOME/bin

source /etc/profile
```

#### 2.3.2 创建虚拟环境


```bash
# 创建虚拟环境
uv venv vllm-dev --python 3.12 --seed
# 激活环境
source vllm-dev/bin/activate
# 安装依赖
pip install modelscope
# 退出环境
deactivate
# 删除环境
rm -rf vllm-dev
```

### 2.5 安装 Miniconda（可选）

#### 2.5.1 在线安装

官方网址：https://repo.anaconda.com/miniconda/

```bash
# 使用清华镜像，速度更快
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
# 执行脚本
bash Miniconda3-latest-Linux-x86_64.sh
# 刷新配置
source ~/.bashrc
# 首次运行需要接受条款
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r
```

默认环境路径 `/root/miniconda3/envs/`

!> 访问默认的 Conda 资源库可能会比较慢。为了加快包的下载速度，可以设置国内的镜像源。

```bash
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/pkgs/main
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/pkgs/free
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/condaforge
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/msys2
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/bioconda
conda config --set show_channel_urls yes
```

删除镜像源

```bash
conda config --remove-key channels
```

查看镜像源

```bash
conda config --show channels
```

#### 2.5.2 创建虚拟环境

```bash
# 创建虚拟环境
conda create -n vllm-dev python=3.12 -y
# 激活
conda activate vllm-dev
# 安装依赖
pip install modelscope
# 退出环境
conda deactivate
# 删除环境
conda env remove --name vllm-dev -y
```

#### 2.5.3 环境迁移

源主机备份

```bash
# 激活
conda activate copaw-flash-9b
# 安装依赖
conda install conda-pack
# 将环境打包
conda pack -n copaw-flash-9b -o copaw-flash-9b.tar.gz
```

目标主机还原

```bash
cd ~/miniconda3/envs/
# 创建目录
mkdir copaw-flash-9b
# 解压到指定目录下
tar -xzvf copaw-flash-9b.tar.gz -C copaw-flash-9b
# 进入目录
cd copaw-flash-9b
# 激活环境
source activate ./
# 如果源路径和目标路径不一致，需要解决硬编码路径，请执行
conda-unpack
# 激活成功后可以在环境列表中查看
conda env list
# 验证
conda activate copaw-flash-9b
```


## 3. 开发框架

### 3.1 Flask Web

Flask 是轻量级 Web 开发框架，让你用灵活代码构建后端服务与自定义前端界面

#### 3.1.1 Hello World

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

3. 服务端

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

#### 3.1.2 获取URL参数

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


#### 3.1.3 获取POST表单参数

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


#### 3.1.4 获取JSON参数

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

#### 3.1.5 上传文件

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

#### 3.1.6 Restful URL

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

#### 3.1.7 参数类型转换

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

#### 3.1.8 自定义路由转换器

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

#### 3.1.9 url_for生成链接

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

#### 3.1.10 重定向

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

#### 3.1.11 Jinja2模板引擎

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
        'email': 'xzh@163.com',
        'age':0,
        'github': 'https://github.com/xzh-net'
    }
    return render_template('user_info.html', page_title='xzh\'s info', user_info=user_info)
 
 
if __name__ == '__main__':
    app.run(port=5000, debug=True, host='0.0.0.0')
```

访问地址：http://127.0.0.1:5000/user

#### 3.1.12 全局异常404

服务端

```python
from flask import Flask, render_template_string, abort
 
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'hello world'

@app.route('/user')
def user():
    abort(401)  # Unauthorized

@app.errorhandler(401)
def page_unauthorized(error):
    return render_template_string('<h1> 请登录 </h1> <h3>{{ error_info }}</h3>', error_info=error), 401

if __name__ == '__main__':
    app.run(port=5000, debug=True, host='0.0.0.0')
```

访问地址：http://127.0.0.1:5000/user

#### 3.1.13 用户会话

服务端

```python
from flask import Flask, render_template_string, \
    session, request, redirect, url_for

app = Flask(__name__)
app.secret_key = '(qo24t%@4=+pY_sHI%z8Rn-&5HCF'
 
@app.route('/')
def hello_world():
    return 'hello world'

@app.route('/login')
def login():
    page = '''
    <form action="{{ url_for('do_login') }}" method="post">
        <p>name: <input type="text" name="user_name" /></p>
        <input type="submit" value="Submit" />
    </form>
    '''
    return render_template_string(page)

@app.route('/do_login', methods=['POST'])
def do_login():
    name = request.form.get('user_name')
    session['user_name'] = name
    return 'success'

@app.route('/show')
def show():
    return session['user_name']

@app.route('/logout')
def logout():
    session.pop('user_name', None)
    return redirect(url_for('login'))

if __name__ == '__main__':
    app.run(port=5000, debug=True, host='0.0.0.0')
```

访问地址：http://127.0.0.1:5000/login


#### 3.1.14 使用Cookie

服务端
```python
from flask import Flask, request, Response, make_response
import time

app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'hello world'

@app.route('/add')
def login():
    res = Response('add cookies')
    res.set_cookie(key='name', value='yyy', expires=time.time()+6*60)
    return res

@app.route('/show')
def show():
    return request.cookies.__str__()

@app.route('/del')
def del_cookie():
    res = Response('delete cookies')
    res.set_cookie('name', '', expires=0)
    return res

if __name__ == '__main__':
    app.run(port=5000, debug=True, host='0.0.0.0')
```

访问地址：http://127.0.0.1:5000/add



### 3.2 Gradio

Gradio 是专为机器学习模型演示而生的工具，让你用极简代码生成前端界面

#### 3.2.1 安装依赖

```bash
pip install gradio
```

#### 3.2.2 标准

```python
import numpy as np
import gradio as gr

def sepia(input_img):
    sepia_filter = np.array([
        [0.393, 0.769, 0.189],
        [0.349, 0.686, 0.168],
        [0.272, 0.534, 0.131]
    ])
    sepia_img = input_img.dot(sepia_filter.T)
    sepia_img /= sepia_img.max()
    return sepia_img

demo = gr.Interface(sepia, gr.Image(), "image", api_name="predict")
demo.launch(server_name="0.0.0.0", server_port=7860)
```

#### 3.2.3 仅输出

```python
import time

import gradio as gr

def fake_gan():
    time.sleep(1)
    images = [
            "https://images.unsplash.com/photo-1507003211169-0a1dd7228f2d?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=387&q=80",
            "https://images.unsplash.com/photo-1554151228-14d9def656e4?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=386&q=80",
            "https://images.unsplash.com/photo-1542909168-82c3e7fdca5c?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxzZWFyY2h8MXx8aHVtYW4lMjBmYWNlfGVufDB8fDB8fA%3D%3D&w=1000&q=80",
    ]
    return images

demo = gr.Interface(
    fn=fake_gan,
    inputs=None,
    outputs=gr.Gallery(label="Generated Images", columns=2),
    title="FD-GAN",
    description="This is a fake demo of a GAN. In reality, the images are randomly chosen from Unsplash.",
    api_name="predict",
)

demo.launch(server_name="0.0.0.0", server_port=7860)
```

#### 3.2.4 仅输入

```python
import random
import string
import gradio as gr

def save_image_random_name(image):
    random_string = ''.join(random.choices(string.ascii_letters, k=20)) + '.png'
    image.save(random_string)
    print(f"Saved image to {random_string}!")

demo = gr.Interface(
    fn=save_image_random_name,
    inputs=gr.Image(type="pil"),
    outputs=None,
    api_name="predict",
)
demo.launch(server_name="0.0.0.0", server_port=7860)
```

#### 3.2.5 统一

```python
import gradio as gr
from transformers import pipeline

generator = pipeline('text-generation', model = 'gpt2')

def generate_text(text_prompt):
  response = generator(text_prompt, max_length = 30, num_return_sequences=5)
  return response[0]['generated_text']  

textbox = gr.Textbox()

demo = gr.Interface(generate_text, textbox, textbox, api_name="predict")

demo.launch(server_name="0.0.0.0", server_port=7860)
```