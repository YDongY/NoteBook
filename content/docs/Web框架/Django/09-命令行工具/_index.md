# django-admin 和 manage.py

> https://docs.djangoproject.com/zh-hans/3.1/ref/django-admin/

* django-admin 是 Django 的命令行工具集，用于处理系统管理员相关操作。一般保存在环境变量中，在命令行或终端都可以直接使用，位于 Python 的 `site-packages/django/bin` 目录下。django-admin 可以针对不同的项目进行设置：

```bash
$ django-admin <command> --settings=mysite.settings
```

或者修改 `DJANGO_SETTINGS_MODULE` 环境变量

* manage.py 是在创建 Django 项目的时候自动生成的，存放在项目目录下，作用与 django-admin 相同，但是只对当前工程有效

```python
#!/usr/bin/env python
"""Django's command-line utility for administrative tasks."""
import os
import sys

def main():
    """Run administrative tasks."""
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'my_bbs.settings')
    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed and "
            "available on your PYTHONPATH environment variable? Did you "
            "forget to activate a virtual environment?"
        ) from exc
    execute_from_command_line(sys.argv)

if __name__ == '__main__':
    main()
```

* 用法

```bash
$ django-admin <command> [options]
$ manage.py <command> [options]
$ python -m django <command> [options]
```

{{< hint danger >}}
**注意**

如果在运行某些 django-admin 命令没有设置 `--settings` 参数，也没有环境变量 `DJANGO_SETTINGS_MODULE` 就会出现异常

{{< /hint >}}

## help

获取帮助信息

```bash
$ django-admin help startproject 
usage: django-admin startproject [-h] [--template TEMPLATE] [--extension EXTENSIONS] [--name FILES] [--version] [-v {0,1,2,3}]
                                 [--settings SETTINGS] [--pythonpath PYTHONPATH] [--traceback] [--no-color] [--force-color]
                                 name [directory]

Creates a Django project directory structure for the given project name in the current directory or optionally in the given
directory.

......
```

## check

检查工程中是否存在错误，默认会检查全部应用

```bash
$ django-admin check [app_label [app_label ...]]
```

Example：

```bash
$ django-admin check polls --settings=my_bbs.settings --pythonpath=/home/ydongy/PycharmProjects/my_bbs
System check identified no issues (0 silenced).
```

使用 `--tag` 参数可以约束 check 命令的检查范围，通过 `--list-tags` 参数查看全部可以用的类别

```bash
$ django-admin check polls --settings=my_bbs.settings --pythonpath=/home/ydongy/PycharmProjects/my_bbs --list-tags
admin
caches
database
models
security
staticfiles
templates
translation
urls

$ django-admin check polls --settings=my_bbs.settings --pythonpath=/home/ydongy/PycharmProjects/my_bbs --tag urls --tag models
```

## compilemessages

将 `.po` 文件编译成国际化和本地化的 `.mo` 文件（使用 makemessages 命令可生成 `.po` 文件）

```bash
$ django-admin compilemessages 
```

可选参数：

* `--local LOCALE, -l LOCALE`：指定待编译区域，如果没有设置则编译全部区域的 `.po` 文件
* `--exclude EXCLUDE, x EXCLUDE`：指定要从处理中排除的区域设置，如果没有提供，则不排除任何地区
* `--use-fuzzy, -f`：将模糊翻译编译到 `.mo` 文件
* `--ignore PATHERN, -i PATHERN`：编译消息文件时忽略与 glob 风格匹配的路径，可多次出现

## createcachetable

使用 settings 中的 CACHES 配置创建缓存表，命令接受两个参数：

* `--database DATABASE`：该参数用于指定常见缓存表的数据库，默认使用 default 数据库
* `-dry-run`：打印出创建缓存表时所执行的 SQL 脚本，并不是真正执行该脚本

Example：

```python
# settings.py

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.db.DatabaseCache',
        'LOCATION': 'my_cache_table'
    }
}
```

执行 createcachetable 命令：

```bash
$ django-admin createcachetable --settings=my_bbs.settings --pythonpath=/home/ydongy/PycharmProjects/my_bbs --dry-run
CREATE TABLE `my_cache_table` (
    `cache_key` varchar(255) NOT NULL PRIMARY KEY,
    `value` longtext NOT NULL,
    `expires` datetime(6) NOT NULL
);
CREATE INDEX `my_cache_table_expires` ON `my_cache_table` (`expires`);
```

## dbshell

连接 settings 中配置的数据库并执行命令行

Example：

```bash
$ django-admin dbshell --settings=my_bbs.settings --pythonpath=/home/ydongy/PycharmProjects/my_bbs
```

## diffsettings

显示当前工程配置文件与 Django 的默认配置文件之间的差别，如果默认配置文件中缺少某项配置则显示 `###` 。

Example：

```bash
$ django-admin diffsettings --settings=my_bbs.settings --pythonpath=/home/ydongy/PycharmProjects/my_bbs
AUTHENTICATION_BACKENDS = ['django.contrib.auth.backends.ModelBackend', 'post.backends.MasterKeyBackend']
AUTH_PASSWORD_VALIDATORS = [{'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator'}, {'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator'}, {'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator'}, {'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator'}]
BASE_DIR = PosixPath('/home/ydongy/PycharmProjects/my_bbs')  ###
CACHES = {'default': {'BACKEND': 'django.core.cache.backends.db.DatabaseCache', 'LOCATION': 'my_cache_table'}}
DATABASES = {'default': {'ENGINE': 'django.db.backends.mysql', 'NAME': 'bbs', 'USER': 'root', 'PASSWORD': 'root', 'HOST': '127.0.0.1', 'PORT': 3306, 'ATOMIC_REQUESTS': False, 'AUTOCOMMIT': True, 'CONN_MAX_AGE': 0, 'OPTIONS': {}, 'TIME_ZONE': None, 'TEST': {'CHARSET': None, 'COLLATION': None, 'MIGRATE': True, 'MIRROR': None, 'NAME': None}}}
DEBUG = True
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'
INSTALLED_APPS = ['django.contrib.admin', 'django.contrib.auth', 'django.contrib.contenttypes', 'django.contrib.sessions', 'django.contrib.messages', 'django.contrib.staticfiles', 'post.apps.PostConfig', 'polls.apps.PollsConfig']
LANGUAGE_CODE = 'zh-hans'
LOGIN_REDIRECT_URL = '/post/topic_list/'
LOGIN_URL = '/login/'
MIDDLEWARE = ['django.middleware.security.SecurityMiddleware', 'django.contrib.sessions.middleware.SessionMiddleware', 'django.middleware.common.CommonMiddleware', 'django.middleware.csrf.CsrfViewMiddleware', 'django.contrib.auth.middleware.AuthenticationMiddleware', 'django.contrib.messages.middleware.MessageMiddleware', 'django.middleware.clickjacking.XFrameOptionsMiddleware']
ROOT_URLCONF = 'my_bbs.urls'  ###
SECRET_KEY = 'django-insecure-wu3(yfiv7ghdsr=x)*bys@fd7(32220rx)m11r*o*8$25lhl8f'
SETTINGS_MODULE = 'my_bbs.settings'  ###
STATIC_URL = '/static/'
TEMPLATES = [{'BACKEND': 'django.template.backends.django.DjangoTemplates', 'DIRS': [], 'APP_DIRS': True, 'OPTIONS': {'context_processors': ['django.template.context_processors.debug', 'django.template.context_processors.request', 'django.contrib.auth.context_processors.auth', 'django.contrib.messages.context_processors.messages']}}]
TIME_ZONE = 'Asia/Shanghai'
USE_L10N = True
WSGI_APPLICATION = 'my_bbs.wsgi.application'
```

默认输出格式比较乱，可以使用 `--output unified` 进行格式化，此时默认配置前会以 `-` 开头

```bash
$ django-admin diffsettings --output unified --settings=my_bbs.settings --pythonpath=/home/ydongy/PycharmProjects/my_bbs
- AUTHENTICATION_BACKENDS = ['django.contrib.auth.backends.ModelBackend']
+ AUTHENTICATION_BACKENDS = ['django.contrib.auth.backends.ModelBackend', 'post.backends.MasterKeyBackend']
- AUTH_PASSWORD_VALIDATORS = []
+ AUTH_PASSWORD_VALIDATORS = [{'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator'}, {'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator'}, {'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator'}, {'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator'}]
+ BASE_DIR = PosixPath('/home/ydongy/PycharmProjects/my_bbs')
- CACHES = {'default': {'BACKEND': 'django.core.cache.backends.locmem.LocMemCache'}}
+ CACHES = {'default': {'BACKEND': 'django.core.cache.backends.db.DatabaseCache', 'LOCATION': 'my_cache_table'}}
- DATABASES = {}
+ DATABASES = {'default': {'ENGINE': 'django.db.backends.mysql', 'NAME': 'bbs', 'USER': 'root', 'PASSWORD': 'root', 'HOST': '127.0.0.1', 'PORT': 3306, 'ATOMIC_REQUESTS': False, 'AUTOCOMMIT': True, 'CONN_MAX_AGE': 0, 'OPTIONS': {}, 'TIME_ZONE': None, 'TEST': {'CHARSET': None, 'COLLATION': None, 'MIGRATE': True, 'MIRROR': None, 'NAME': None}}}
- DEBUG = False
+ DEBUG = True
```

## dumpdata

在标准输出设备中输出指定应用程序数据，输出的数据可用于 loaddata 命令

```bash
$ django-admin dumpdata [app_label[.ModelName] [app_label[.ModelName] ...]]
```

Example：

* 使用 `--indent` 参数格式化 json 字符串

```bash
$ django-admin dumpdata polls.question --indent=4  --settings=my_bbs.settings --pythonpath=/home/ydongy/PycharmProjects/my_bbs
[
{
    "model": "polls.question",
    "pk": 1,
    "fields": {
        "question_text": "你喜欢Django吗？",
        "pub_date": "2021-08-11T11:08:00"
    }
}
]
```

* 使用 `--output` 参数输出到文件

```bash
$ django-admin dumpdata polls.question --output=./question.json --indent=4  --settings=my_bbs.settings --pythonpath=/home/ydongy/PycharmProjects/my_bbs
[...........................................................................]
```

## flush

清空当前数据库中的数据，但是会保留 migration 的变更。如果想要一个干净的数据库，Django 官方推荐通过删除数据库并重建而不是清空数据库

```bash
$ django-admin flush
```

## inspectdb

查询数据库表或者视图对应的 Django 模型，如果没有提供参数，则仅在使用 `--include-views` 选项时为视图创建模型。通过该命令可以方便的将以后的数据库表或者视图转换为 Django 模型

```bash
$ django-admin inspectdb [table [table ...]]
```

Example：

* 查看 polls_question 表对应的 Django 模型

```bash
$ django-admin inspectdb polls_question --settings=my_bbs.settings --pythonpath=/home/ydongy/PycharmProjects/my_bbs
```

* 查看数据库视图对应的 Django 模型

首先在数据库创建一个视图

```mysql
mysql> select q.question_text,q.pub_date,c.choice_text,c.votes from polls_question as q inner join polls_choice as c on q.id = c.question_id
+------------------+---------------------+-------------+-------+
| question_text    | pub_date            | choice_text | votes |
+------------------+---------------------+-------------+-------+
| 你喜欢Django吗？   | 2021-08-11 11:08:00 | 喜欢        | 2     |
| 你喜欢Django吗？   | 2021-08-11 11:08:00 | 不喜欢      | 1      |
| 你喜欢Django吗？   | 2021-08-11 11:08:00 | 不知道      | 0     |
+------------------+---------------------+-------------+-------+

mysql> create view question_choice_view as select q.question_text,q.pub_date,c.choice_text,c.votes from polls_question as q inner join polls_choice as c on q.id = c.question_id;
```

执行 inspectdb 命令

```bash
$ django-admin inspectdb question_choice_view --settings=my_bbs.settings --pyth
onpath=/home/ydongy/PycharmProjects/my_bbs --include-views
```

{{< hint danger >}}
**注意**  

* 当 inspectdb 命令无法将数据库中的字段类型转换为 Django 模型字段类型时将会使用 TextField ，同时插入一条注释：`This field type is a aguess`
* 如果数据库字段名是 Python 保留字，会自动添加一个 `_field` 的后缀
* inspectdb 命令不会根据数据库字段默认值生成 model 字段的默认值
* 默认情况下 inspectdb 命令生成的模型是非 Django 托管模型（managed=False），如果想要生成托管模型，可以使用 managed 参数

* Oracle
  + 使用 `--includ-views`，则会为物化视图创建模型
* PostgreSQL
  + 为外部表创建模型
  + 如果使用 `--includ-views`，则会为物化视图创建模型
  + 如果使用 `--include-partitions`，则为分区表创建模型

{{< /hint >}}

## loaddata

加载数据到数据库

```bash
$ django-admin loaddata fixture [fixture ...]
```

Example：

创建 fixture 文件 question.json ，并放在 manage.py 同级目录

```json
[{
    "model": "polls.question",
    "pk": 1,
    "fields": {
        "question_text": "Django 框架好用吗?",
        "pub_date": "2021-08-11T11:08:00"
    }
}]
```

执行 loaddata 命令

```bash
$ django-admin loaddata question.json --settings=my_bbs.settings --pythonpath=/home/ydongy/PycharmProjects/my_bbs
Installed 1 object(s) from 1 fixture(s)
```

{{< hint danger >}}
**注意**  

fixture 是序列化好的数据文件，文件格式包含 json 和 xml。Django 只能从以下 3 类位置查找 fixture：

1. 应用程序下的 fixture 文件
2. 配置文件中的 FIXTURE_DIRS 指定的路径
3. fixture 文件路径

loaddata 命令可以在压缩文件中查找 fixture，支持 zip、gz 和 bz2 等压缩格式

{{< /hint >}}

## makemessages

查找整个源代码路径以找出全部翻译字符串并生成一个新的消息文件或更新已有的消息文件

```bash
$ django-admin makemessages -l zh_HANS -l en --settings=my_bbs.settings --pythonpath=/home/ydongy/PycharmProjects/my_bbs
processing locale zh_HANS
processing locale en
```

## startproject

创建 Django 项目

```bash
$ django-admin startproject name [directory]
```

## startapp

创建 Django 应用程序

```bash
$ django-admin startapp name [directory]
```

可选参数：

* `--template TEMPLATE`：导入外部应用程序模板，TEMPLATE 可以是包含模板文件的路径、包含压缩包的路径或者 URL

```bash
$ django-admin startapp --template=xxx app_name
```

## runserver

启动一个轻量级 Web 服务器，默认端口 8000

```bash
$ django-admin runserver
$ django-admin runserver 127.0.0.1:8000
$ django-admin runserver 7000
$ django-admin [fe80::13b1:9bc1:93e1:a11e]:7000
```

## sendtestemail

发送测试邮件已检测邮箱设置是否正确

```bash
$ django-admin sendtestemail [email_addr [email_addr] ...]
```

## shell

启动一个 Python 交互窗口

```bash
$ django-admin shell --interface {ipthon,bpython,python}
$ django-admin shell -i {ipthon,bpython,python}
```

## 迁移

> https://docs.djangoproject.com/zh-hans/3.1/topics/migrations/#module-django.db.migrations

迁移可以看作是数据库架构的版本控制系统。makemigrations 负责将模型修改打包进独立的迁移文件中，类似提交修改，而 migrate 负责将其应用至数据库。

### makemigrations

根据模型的变化生成迁移代码，改代码用于更新数据库。

```bash
$ django-admin makemigrations [app_label [app_label ...]]
```

如果没有填写任何参数，Django 会检查所有应用程序中的模型并生成迁移脚本。makemigrations 命令会检查注册的应用目录下是否存在 migrations 目录，如果没有就创建，并根据表结构定义生成一个 0001_initial.py 文件，里面定义了数据表的 Schema。对于将来应用的每一次表结构定义修改，都需要执行该命令，同时 Django 会重新生成一个新的迁移文件，记录表结构之间的差异，命令规则是对上一个迁移文件的序列号加 1，如 0002_xxx、0003_xxx

Example：

* 为某个应用程序生成迁移

```bash
$ django-admin makemigrations polls
```

* 生成一个空迁移，高级开发人员可以在空迁移中编写代码

```bash
$ django-admin makemigrations --empty
```

* 生成一个带名称的迁移

```bash
$ django-admin makemigrations --empty --name empty
```

### migrate

将模型的最新状态部署到数据库

```bash
$ django-admin migrate [app_label] [migration_name]
```

migrate 命令会根据迁移文件生成对应的数据库表，不过为了保证完成的迁移工作不重复执行，Django 会把每一次数据库迁移记录到 django_migrations 表中，每一次执行 migrate 前都会比较迁移文件是否已经记录表中，只要没出现过的才会执行。

Example：

* 迁移所有应用程序模型

```bash
$ django-admin migrate 
```

* 迁移指定应用程序模型

```bash
$ django-admin migrate polls
```

* 通过 migrate 传递上一次迁移的编号来撤销迁移。例如，要撤销迁移 books.0003

```bash
$ python manage.py migrate books 0002
Operations to perform:
  Target specific migration: 0002_auto, from books
Running migrations:
  Rendering model states... DONE
  Unapplying books.0003_auto... OK
```

* 撤消应用于一个应用的所有迁移，请使用名称 zero：

```bash
$ python manage.py migrate books zero
Operations to perform:
  Unapply all migrations: books
Running migrations:
  Rendering model states... DONE
  Unapplying books.0002_auto... OK
  Unapplying books.0001_initial... OK
```

如果迁移包含任何不可逆的操作，则该迁移是不可逆的。 试图撤销这种迁移将引发 IrreversibleError

其他可用参数：

* `--fake`：对于高级用户，仅仅想设置当前的 migration 状态，并不需要真正去更新数据库，可以使用该参数
* `--database DATABASE`：将模型更改应用到指定的数据库，默认更新到 default 数据库

### sqlmigrate

输出某一个 migrate 对应的 SQL 语句

```bash
$ django-admin sqlmigrate app_label migration_name
```

### showmigrations

列出项目的迁移和迁移的状态

```bash
$ django-admin showmigrations [app_label [app_label ...]]
```

Example：

```bash
$ django-admin showmigrations polls --settings=my_bbs.settings --pythonpath=/home/ydongy/PycharmProjects/my_bbs
polls
 [X] 0001_initial
```

* `--list, -l`：按照应用程序显示 migration 记录
* `--plan, -p`：显示所有记录
* `[X]`：表示此次 migration 已经部署到数据库
* `[]`：表示此次 migration 还没有部署到数据库

## changepassword

修改用户密码

```bash
$ django-admin changepassword [<username>]
```

## createsuperuser

为 Django 创建一个超级用户

```bash
$ django-admin createsuperuser
```

## collectstatic

收集静态文件并保存到 STATIC_ROOT 所指定的文件夹

```bash
$ django-admin collectstatic
```

## findstatic

查找一个或多个静态文件路径

```bash
$ django-admin findstatic staticfile [statictile ...]
```

Example：

```bash
$ django-admin findstatic polls/style.css --settings=my_bbs.settings --pythonpath=/home/ydongy/PycharmProjects/my_bbs
Found 'polls/style.css' here:
  /home/ydongy/PycharmProjects/my_bbs/polls/static/polls/style.css
```
