Module-Dotenv
===========


###　写在前面

本周的文章是介绍一个好用的 Python 库 [python-dotenv](https://github.com/theskumar/python-dotenv),该库是用来处理环境变量的工具。

在正式项目或者开源项目中，我们会把诸如 Mysql 密码，App　Secrete等机密信息存为环境变量，而不是硬编码在项目中，举个例子，在 django settings 中的 mysql 设置

不推荐的方式

```python

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'db_name',
        'USER': 'db_user',
        'PASSWORD': 'db_password',
        'HOST': 'db_host',
        'PORT': 'db_port'
    }
}


```

推荐的方式

```python

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': os.environ["MYSQL_DB_NAME"],
        'USER': os.environ["MYSQL_USER"],
        'PASSWORD': os.environ["MYSQL_PASSWORD"],
        'HOST': os.environ["MYSQL_HOST"],
        'PORT': os.environ["MYSQL_PORT"]
    }
}

```

所以，为了达到以上的目的，需要事先将机密数据写入环境变量，目前有许多方式可以达到目的，但在尝试过多种方式之后，觉得 dotenv 库是比较舒服的方式。


### 安装和简单的应用

安装过程以及简单的应用见 python-dotenv 的 [readme](https://github.com/theskumar/python-dotenv)


### 使用心得


在正式的项目中，我一般这么使用


在项目的启动目录中，比如在 django 项目中就是 `manage.py`　同级目录下，建立一个 `.env` 文件。填入相应的内容

.env 
```linux

export MYSQL_NAME=db_name
export MYSQL_PW=db_password

```

记得使用 export 前缀，这是因为我们可以用两种方式将数据写入环境变量。

比如在docker环境中，我们需要先在 mysql 写入部分数据，而不是直接启动 django 应用。这时候我就可以先使用命令

```linux

source .env

```

让数据能够写入环境变量，在mysql数据写入完成之后再启动项目，只需要修改 `manage.py` 文件

```python
import os
import sys

##　修改部分
from dotenv import load_dotenv
load_dotenv()
##

if __name__ == '__main__':
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'PatrickLab.settings')
    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed and "
            "available on your PYTHONPATH environment variable? Did you "
            "forget to activate a virtual environment?"
        ) from exc
    execute_from_command_line(sys.argv)


```

这样，每次启动应用时，都会事先写入环境变量。

### 结尾

xin.zhou 20190518