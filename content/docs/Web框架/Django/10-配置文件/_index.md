# Django 配置

> https://docs.djangoproject.com/zh-hans/3.1/ref/settings/

Django 的配置文件是一个 Python 模块，使用 Django 是必须通过环境变量 `DJANGO_SETTINGS_MODULE` 指定当前工程所使用的配置文件，默认情况下在 `manage.py` 中指定。

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

如果使用 `WSGI` 部署 Django 应用程序，需要在 `wsgi.py` 中指定配置文件

```python
import os

from django.core.wsgi import get_wsgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'my_bbs.settings')

application = get_wsgi_application()
```

如果配置文件缺少某个配置的设置，则会使用默认配置文件的默认值，路径是 `django/conf/global_settings.py` 。所以 Django 编译配置文件顺序：

1. 加载 `global_settings.py`
2. 加载工程目录下的 `settings.py` ，使用工程指定的配置重写默认值

## django.conf.settings

在 Django 应用程序中可以通过导入 `django.conf.settings` 来引用 Django 配置信息，例如：

```python
from django.conf import settings
if settings.DEBUG:
    pass
```

## django.setup

如果只使用 Django 的一部分代码而不是启动整个项目，例如加载某个模板并渲染它，或者使用 ORM 来加载数据，此时可以使用 `django.setup()` 来加载配置文件

```python
import django
from django.conf import settings
import os
import sys
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'my_bbs.settings')
django.setup()
```

# 缓存

默认值：

```bash
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
    }
}
CACHE_MIDDLEWARE_KEY_PREFIX = ''   # 缓存中间件所生成的缓存 key 的前缀
CACHE_MIDDLEWARE_SECONDS = 600     # 缓存中间件缓存一个页面默认时长，单位秒
CACHE_MIDDLEWARE_ALIAS = 'default' # 缓存中间件所使用的缓存
```

* `default`：缓存别名
* `BACKEND`：缓存的存储引擎
  + `'django.core.cache.backends.db. DatabaseCache'`
  + `'django.core.cache.backends.dummy. DummyCache'`
  + `'django.core.cache.backends.filebased. FileBasedCache'`
  + `'django.core.cache.backends.locmem. LocMemCache'`
  + `'django.core.cache.backends.memcached. MemcachedCache'`
  + `'django.core.cache.backends.memcached. PyLibMCCache'`

可选参数：

* KEY_PREFIX：默认值为`""`，这个字符串会被自动添加在 Django 生成的缓存 Key 前面作为前缀
* LOCATION：默认值为`""`，表示缓存地址。如果是文件缓存，则表示文件目录，如果是 memcache，则表示 memcache 服务器的主机号和端口号，或者本地内存缓存名字

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
        'LOCATION': '/var/tmp/django_cache',
    }
}
```

* OPTIONS：默认值 None，传递给 BACKEND 的额外参数。
  + `MAX_ENTRIES`：删除旧值之前允许缓存的最大条目。默认 300
  + `CULL_FREQUENCY`：当达到 MAX_ENTRIES 时被淘汰的部分条目，实际比率为 1/CULL_FREQUENCY ，默认为 3

* TIMEOUT：默认值 300，缓存过期时间，单位秒。如果为 None，则永不过期
* VERSION：默认值 1，通过 Django 生成的缓存默认版本号
# 数据库

## DATABASES

> https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#databases

默认值：

```python
DATABASES = {}
```

Django 支持许多不同的数据库服务器，官方支持 PostgreSQL、MariaDB、MySQL、Oracle 和 SQLite。为了能在 Django 中使用其他数据库，还需要提供第三方的后端。

* PostgreSQL ==> psycopg2
* MySQL / MariaDB ==> mysqlclient

Django 提供 4 中数据库引擎：

* `django.db.backends.postgresql`
* `django.db.backends.mysql`
* `django.db.backends.sqlite3`
* `django.db.backends.oracle`

```python
DATABASES = {
    # 第一个数据库
    'default': {
        'ENGINE': 'django.db.backends.postgresql',	# 数据库引擎
        'NAME': 'mydatabase',						# 数据库的名称
        'USER': 'mydatabaseuser',					# 数据库用户名
        'PASSWORD': 'mypassword',					# 数据库密码
        'HOST': '127.0.0.1',						# 数据库服务器地址
        'PORT': '5432',								# 数据库服务器端口号
    },
    # 第二个数据库
    'MyDjango': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
```

更多配置：

```python
DATABASES = {
    # 'default': {
    #     'ENGINE': 'django.db.backends.sqlite3',
    #     'NAME': BASE_DIR / 'db.sqlite3',
    # }
    'default': {
        'ENGINE': '',              # 数据库引擎
        'NAME': '',                # 数据库名
        'USER': '',                # 连接数据库的用户名
        'PASSWORD': '',            # 数据库连接密码
        'HOST': '',                # 数据库所在主机名
        'PORT': '',                # 数据库开发端口号
        'ATOMIC_REQUESTS': False,  # 如果为 True 将会把每个视图的数据库操作封装到一个数据库事务中
        'AUTOCOMMIT': True,        # 如果为 False 那么将终止 Django 默认的事务管理功能
        'CONN_MAX_AGE': 0,         # 数据库会话的生命周期，单位秒，0 表示请求结束立刻关闭，None 表示没有限制
        'OPTIONS': {},             # 连接数据库是需要使用的附加参数
        'TIME_ZONE': None,         # 数据库所使用的时区
        'DISABLE_SERVER_SIDE_CURSORS': False,  # 为 True 表示在 QuerySet.iterator() 中禁用服务器端游标
        'TEST': {                  # 用于配置测试数据库
            'NAME': None,       # 运行测试套件时要使用的数据库名称，默认情况下 SQLite 使用一个内存数据库，其他数据库名称 'test_' + DATABASE_NAME
            'CHARSET': None,    # 创建数据库时所使用的编码
            'COLLATION': None,  # 创建数据库时所使用的排序规则，目前只有 MySQL 支持
            'DEPENDENCIES': ['default'],  # 创建数据库时的依赖顺序
            'MIRROR': True,  # 测试过程中，需要进行镜像的数据库的别名，用于主从复制
            # ......
        },
    }
}
```

## DATABASE_ROUTERS

数据库路由配置，默认值： `[]` 。

要想使用数据库路由，首先需要创建数据库路由类，该类必须实现以下方法：

* `db_for_read(model, **hints)`：指定对 model 进行读操作的数据库
* `db_for_write(model, **hints)`：指定对 model 进行写操作的数据库
* `allow_relation(obj1, obj2, **hints)`：如果允许 obj1 和 obj2 之间存在关联返回 True，如果禁止关联返回 False，没有限制返回 None
* `allow_migrate(db, app_label, model_name=None, **hints)`：该方法确定数据库是否可以进行 mirgation 操作，返回 True 表示允许，False 表示不允许，None 表示没有特殊规定
  + `db`：数据库别名
  + `app_label`：进行 migration 操作的应用程序名

Example：

```python
DATABASES = {
    'default': {},
    'auth_db': {
        'NAME': 'auth_db_name',
        'ENGINE': 'django.db.backends.mysql',
        'USER': 'mysql_user',
        'PASSWORD': 'swordfish',
    },
    'primary': {
        'NAME': 'primary_name',
        'ENGINE': 'django.db.backends.mysql',
        'USER': 'mysql_user',
        'PASSWORD': 'spam',
    },
    'replica1': {
        'NAME': 'replica1_name',
        'ENGINE': 'django.db.backends.mysql',
        'USER': 'mysql_user',
        'PASSWORD': 'eggs',
    },
    'replica2': {
        'NAME': 'replica2_name',
        'ENGINE': 'django.db.backends.mysql',
        'USER': 'mysql_user',
        'PASSWORD': 'bacon',
    },
}
```

下面创建一个用于处理 auth 应用向 auth_db 数据库发送请求的路由类

```python
class AuthRouter:
    """
    用于处理所有 auth 和 contenttypes 应用的数据库请求的路由
    """
    route_app_labels = {'auth', 'contenttypes'}

    def db_for_read(self, model, **hints):
        """
        尝试在 auth_db 数据库中读取 auth 和 contenttypes model
        """
        if model._meta.app_label in self.route_app_labels:
            return 'auth_db'
        return None

    def db_for_write(self, model, **hints):
        """
        尝试在 auth_db 数据库中对 auth 和 contenttypes model 进行写操作
        """
        if model._meta.app_label in self.route_app_labels:
            return 'auth_db'
        return None

    def allow_relation(self, obj1, obj2, **hints):
        """
        允许 auth 和 contenttypes 应用中的 model 存在关系
        """
        if (
            obj1._meta.app_label in self.route_app_labels or
            obj2._meta.app_label in self.route_app_labels
        ):
           return True
        return None

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        """
        仅允许 auth 和 contenttypes 应用中的 auth_db 进行 migration 操作
        """
        if app_label in self.route_app_labels:
            return db == 'auth_db'
        return None
```

创建一个路由，用于处理其他所有应用向数据库集群发送的请求，对于读操作，则在集群中随机选择一个数据库

```python
import random

class PrimaryReplicaRouter:
    def db_for_read(self, model, **hints):
        """
        Reads go to a randomly-chosen replica.
        """
        return random.choice(['replica1', 'replica2'])

    def db_for_write(self, model, **hints):
        """
        Writes always go to primary.
        """
        return 'primary'

    def allow_relation(self, obj1, obj2, **hints):
        """
        Relations between objects are allowed if both objects are
        in the primary/replica pool.
        """
        db_set = {'primary', 'replica1', 'replica2'}
        if obj1._state.db in db_set and obj2._state.db in db_set:
            return True
        return None

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        """
        All non-auth models end up in this pool.
        """
        return True
```

最后，在配置文件中，我们添加下面的代码（用定义路由器的模块的实际 Python 路径替换 path.to）：

```python
DATABASE_ROUTERS = ['path.to.AuthRouter', 'path.to.PrimaryReplicaRouter']
```

## DEFAULT_INDEX_TABLESPACE

默认值： `""` ，字段索引的默认表空间

## DEFAULT_TABLESPACE

默认值： `""` ，模型的默认表空间

# 电子邮件

## ADMINS

默认值： `[]` ，能够获得代码错误信息的人员列表。当配置文件设置了 DEBUG=False，同时 LOGGING 参数设置了 AdminEmailHandler 时（默认已设置），Django 会将 Web 请求中发生的详细异常信息发送给列表中的所有人

```python
[('Full Name', 'email@example.com'), ('Full Name', 'anotheremail@example.com')]
```

## DEFAULT_FROM_EMAIL

默认值： `'webmaster@localhost'` ，默认的电子邮件地址，用于站点管理员发送的各种自动通信

## 其他选项

```python
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend' # 默认的邮件引擎

EMAIL_HOST = 'localhost'  # 邮件服务器
EMAIL_PORT = 25           # SMTP 服务的端口号

EMAIL_USE_LOCALTIME = False  # 是否以当地时区（True）或 UTC（False）发送 SMTP Date 邮件头

EMAIL_HOST_USER = ''       # 用于登录邮件服务的用户名
EMAIL_HOST_PASSWORD = ''   # 用于登录邮件服务的用户密码
EMAIL_USE_TLS = False      # SMTP 服务是否使用 TLS 连接
EMAIL_USE_SSL = False      # SMTP 服务是否使用 SSL 连接
EMAIL_SSL_CERTFILE = None  # 如果使用 TLS 或 SSL，指定一个 PEM 格式的证书链文件的路径
EMAIL_SSL_KEYFILE = None   # 如果使用 TLS 或 SSL，指定用于 SSL 连接的 PEM 格式化私钥文件的路径
EMAIL_TIMEOUT = None       # 发送邮件超时时间

SERVER_EMAIL = 'root@localhost' # 用于发送错误信息的邮件地址
MANAGERS = ADMINS
```

# 文件上传

```python
DEFAULT_FILE_STORAGE = 'django.core.files.storage.FileSystemStorage' # 文件存储类
FILE_UPLOAD_HANDLERS = [                                             # 文件上传处理程序
    'django.core.files.uploadhandler.MemoryFileUploadHandler',
    'django.core.files.uploadhandler.TemporaryFileUploadHandler',
]
FILE_UPLOAD_MAX_MEMORY_SIZE = 2621440  # i.e. 2.5 MB  # 允许上传文件最大体积，单位字节
FILE_UPLOAD_PERMISSIONS = 0o644        # 为新上传的文件设置权限
FILE_UPLOAD_DIRECTORY_PERMISSIONS = None # 上传文件过程中所创建的文件夹设置权限
FILE_UPLOAD_TEMP_DIR = None            # 文件上传时的临时存放路径，为 None 默认临时路径 /tmp
MEDIA_ROOT = ''        # 用于保存用户上传文件的绝对路径  Example: "/var/www/example.com/media/"
MEDIA_URL = ''         # 用于处理 MEDIA_ROOT 中所保存文件的 URL
```

如果 MEDIA_URL 的值不为空，则必须以 `/` 结尾，例如： `"http://example.com/media/", "http://media.example.com/"` 。在使用 MEDIA_ROOT 之前需要进行下面的设置：

```python
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # ... the rest of your URLconf goes here ...
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

如果希望在模板中使用 `{{ MEDIA_URL }}` ，需要将 `'django.template.context_processors.media'` 添加到 TEMPLATES 配置中：

```python
TEMPLATES = [
    {
        ......
        'OPTIONS': {
            'context_processors': [
                ......
                'django.template.context_processors.media' # 添加此处
            ],
        },
    },
]
```

# 静态文件

```python
STATIC_ROOT = None # 指定在使用 django-admin collectstatic 命令收集静态文件的存放目录
STATIC_URL = None  # 引用 STATIC_ROOT 中静态文件时所使用的 URL ，Example: "http://example.com/static/", "http://static.example.com/"
STATICFILES_STORAGE = 'django.contrib.staticfiles.storage.StaticFilesStorage' # 使用 collectstatic 命令收集静态文件时所使用的文件存储引擎
```

## STATICFILES_DIRS

默认值： `[]` ，除了默认的静态文件路径（Django 项目的 static 文件夹）外，额外的静态文件路径。collecstatic 命令会同时收集这些路径下的静态文件：

```python
STATICFILES_DIRS = [
    "/home/special.polls.com/polls/static",
    "/home/polls.com/polls/static",
    "/opt/webfiles/common",
]
```

为了方便使用，还可以给静态文件路径添加命令空间：

```python
STATICFILES_DIRS = [
    # ...
    ("downloads", "/opt/webfiles/stats"),
]
```

假设你将 STATIC_URL 设置为 `'/static/'` ，则 collectstatic 管理命令将收集 STATIC_ROOT 的 `'downloads'` 子目录中的“stats”文件。这将允许你在你的模板中用 `'/static/downloads/polls_20101022.tar.gz'` 引用本地文件 `'/opt/webfiles/stats/polls_20101022.tar.gz'` ，例如：

```django
<a href="{% static 'downloads/polls_20101022.tar.gz' %}">
```

## STATICFILES_FINDERS

用于查找静态文件的文件检索引擎，默认值：

```python
STATICFILES_FINDERS = [
    'django.contrib.staticfiles.finders.FileSystemFinder',
    'django.contrib.staticfiles.finders.AppDirectoriesFinder',
    # 'django.contrib.staticfiles.finders.DefaultStorageFinder',
]
```

* `FileSystemFinder`：用于查找 STATICFILES_DIRS 配置项下面的文件
* `AppDirectoriesFinder`：用于查找每个应用程序的 static 目录
* `DefaultStorageFinder`：查找 DEFAULT_FILE_STORAGE 配置项下的静态文件
# 国际化

> https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#globalization-i18n-l10n

## TIME_ZONE

默认值： `'America/Chicago'` ，因为 Django 的第一个发布版本是该时区，但是创建的新项目默认使用 `UTC` 进行重写。

中国所在时区为 `Asia/Shanghai` ，所以可以设置为：

```python
TIME_ZONE = 'Asia/Shanghai'
```

不过需要注意在 Windows 系统上 Django 不能准确的使用其他不同于本地时区的时区，因此在 Windows 系统上只能设置为本地时区。

## USE_TZ

默认值：False，Django 会使用 TIME_ZONE 参数所指定的时区来保存全部日期时间（包括数据库日期），如果设置为 True，那么 TIME_ZONE 是 Django 模板中显示日期时间的默认时区

## LANGUAGE_CODE

默认值： `en-us` ，当前程序所使用的语言，简体中文为： `zh-Hans` 。为了使本参数有效必须将 USE_I18N 设置为 True

## USE_L10N

默认值：False，规定 Django 是否启用本地化系统，为 True，Django 会对数字、日期、时间字符串进行格式化

# HTTP

```python
DATA_UPLOAD_MAX_MEMORY_SIZE = 2621440  # i.e. 2.5 MB # 每次请求所允许的最大字节数
DATA_UPLOAD_MAX_NUMBER_FIELDS = 1000   # 一次 GET 或 POST 请求所允许传递的最大参数数量
DEFAULT_CHARSET = 'utf-8' # 指定 MIME 字符集用于所有 HttPResponse 对象中的 Content-Type
DISALLOWED_USER_AGENTS = []
FORCE_SCRIPT_NAME = None  # 如果该参数不为 None，那么参数作为任何 HTTP 请求的 SCRIPT_NAME 环境变量
INTERNAL_IPS = []  # 一组 IP 地址字符串，使用场景：
# 1.允许 debug() 方法向模板添加变量
# 2.非注册用户使用管理员页面
# 3.在 AdminEmailHandler 邮件中标注为 internal，也只对应的是 EXTERNAL
```

## DISALLOWED_USER_AGENTS

默认值： `[]` ，一组已经编译的用于匹配 User-Agent 字符串的正则表达式对象，当匹配成功不能访问网站任何页面，主要用于防止互联网爬虫。使用该参数需要确保已经安装了 CommonMiddleware 中间件。

Example：

```python
import re

DISALLOWED_USER_AGENTS = [
    re.compile(r'^OmniExplorer_Bot'),
    re.compile(r'^Googlebot'),
]
```

# 安全

> https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#http

## SECRET_KEY

默认： `""` ，Django 工程的秘钥，用于加密签名，值是一个随机字符串。想要项目中使用建议不要直接使用，如果一定要用，需要使用 `django.utils.encoding.force_text()` 或者 `django.utils.encoding.force_bytes()` 进行安装转换

用途：

* 在除了默认的 session 引擎外的其他引擎中使用，或者在 `get_session_auth_hash()` 方法中使用
* 当使用 CookieStorage 和 FallbackStorage 时，在所有的消息中使用
* 在 PasswordResetView 视图的 token 中使用
* 在加密签名中使用

## ALLOWED_HOSTS

Django 网站可以提供服务的一组主机或域名

```python
ALLOWED_HOSTS = [
    'www.example.com'
    '.example.com'
    '*'
]
```

## SECURE_BROWSER_XSS_FILTER 

默认：False，设置为 True， SecurityMiddleware 将会在每个 HTTP 响应头添加 X-XSS-Protection:1; mode=block。如果浏览器具有阻止 XSS 攻击能力，则浏览器将会阻止一切可疑的 XSS 攻击脚本

# CSRF

```python
# Dotted path to callable to be used as view when a request is
# rejected by the CSRF middleware.
CSRF_FAILURE_VIEW = 'django.views.csrf.csrf_failure'

# Settings for CSRF cookie.
CSRF_COOKIE_NAME = 'csrftoken'
CSRF_COOKIE_AGE = 60 * 60 * 24 * 7 * 52
CSRF_COOKIE_DOMAIN = None
CSRF_COOKIE_PATH = '/'
CSRF_COOKIE_SECURE = False
CSRF_COOKIE_HTTPONLY = False
CSRF_COOKIE_SAMESITE = 'Lax'
CSRF_HEADER_NAME = 'HTTP_X_CSRFTOKEN'
CSRF_TRUSTED_ORIGINS = []
CSRF_USE_SESSIONS = False
```

## CSRF_COOKIE_AGE

默认值： `CSRF_COOKIE_AGE = 60 * 60 * 24 * 7 * 52` ，CSRF Cookie 的有效期。设置超长 CSRF Cookie 有效期的目的是为了防止用户关闭浏览器或者设置了浏览器书签，然后从缓存重新打开浏览器时出现问题。

如果将 `CSRF_COOKIE_AGE=None` 使用基于 session 的 CSRF Cookie，CSRF Cookie 保存在内存中

## CSRF_COOKIE_DOMAIN

默认值： `None` ，设置 CSRF Cookie 时所使用的域名，这个设置可以方便的将跨子域请求与跨站点请求区分开。参数值应该是一个域名： `example.com` ，此时 `a.example.com` 的 POST 请求就可以正常提交给 `b.example.com` 的视图了

## CSRF_COOKIE_HTTPONLY

默认值：False，表示是否在 CSRF Cookie 中使用 HttpOnly。如果设置为 True，那么客户端的 JavaScript 脚本将不能访问 CSRF Cookie。如果使用 AJAX 提交 CSRF token 的时候不能直接读取 Cookie，需要使用一个隐藏的表单控件

## CSRF_COOKIE_NAME

默认值： `csrftoken` ，CSRF 认证中所使用的 Cookie 名

## CSRF_COOKIE_PATH

默认值： `/` ，CSRF Cookie 的路径。

## CSRF_COOKIE_SAMESITE

默认值： `Lax` ，SameSite 可以阻止浏览器将 Cookie 与跨站点请求一起发送。主要目的是降低跨域信息泄露的风险

## CSRF_COOKIE_SECURE

默认值：False，是否为 CSRF Cookie 设置 secure 属性。如果设置为 True，那么这个 Cookie 将只会被 HTTPS 请求发送

## CSRF_USE_SESSIONS

默认值：False，是否将 CSRF token 保护在用户会话中而不是保护在 Cookie 中，该设置需要用到 `django.contrib.sessions`

## CSRF_FAILURE_VIEW

默认值： `'django.views.csrf.csrf_failure'` ，当请求被 CSRF 保护机制拒绝时所调用的视图。

## CSRF_HEADER_NAME

默认值： `'HTTP_X_CSRFTOKEN'` ，用于 CSRF 认证的请求头的名字，保存在 request. META 中。从服务端获取 CSRF_HEADER_NAME 中的全部字符都被自动转换为大写字母，连字符 `-` 转换为下划线 `_` ，同时额外增加 `HTTP_` 的前缀

## CSRF_TRUSTED_ORIGINS

默认值： `[]` ，为不安全（例如 POST 请求）请求提供可信任的主机列表。为了保护不安全的请求，Django 的 CSRF 保护机制要求请求具有一个 Referer 头且与原始请求头的 Host 头匹配，这种机制阻止了不同子域之间相互访问。

# 日志

```python
# The callable to use to configure logging
LOGGING_CONFIG = 'logging.config.dictConfig'

# Custom logging configuration.
LOGGING = {}
```

## LOGGING

默认值： `{}` ，日志配置信息，LOGGING 的内容会被当做参数传递给 LOGGING_CONFIG 所指向的方法。默认配置信息存储在 `django/utils/log/py`

## LOGGING_CONFIG

默认值： `logging.config.dictConfig` ，该参数指向一个用于配置日志系统的可调用对象的路径。如果为 None，系统将跳过日志配置过程

# 模板

默认值： `TEMPLATES = []`

Example：

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR, 'templates'),
                 os.path.join(BASE_DIR, 'index/templates')],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

* `BACKEND`：定义模板引擎，内置的模板有Django Templates和jinja2. Jinja2
* `DIRS`：设置模板文件所在路径
* `APP_DIRS`：是否在 App 里查找模板文件
* `OPTIONS`：额外传递给模板引擎的参数，不同的引擎包含不同的参数，如果 DTL 支持 autoescape， Jinja2 支持 context_processors
# 路由

```python
ROOT_URLCONF = 'projectname.urls' # URL 配置的全路径
APPEND_SLASH = True  # Django 自动在 URL 后追加斜杠（/）。注意：只有在安装 CommonMiddleware 中间件情况下才起作用
PREPEND_WWW = False # 优先使用 WWW 作为 URL 前缀，必须确保安装了 CommonMiddleware
```

# 会话

```python
# Cache to store session data if using the cache session backend.
SESSION_CACHE_ALIAS = 'default'
# Cookie name. This can be whatever you want.
SESSION_COOKIE_NAME = 'sessionid'
# Age of cookie, in seconds (default: 2 weeks).
SESSION_COOKIE_AGE = 60 * 60 * 24 * 7 * 2
# A string like "example.com", or None for standard domain cookie.
SESSION_COOKIE_DOMAIN = None
# Whether the session cookie should be secure (https:// only).
SESSION_COOKIE_SECURE = False
# The path of the session cookie.
SESSION_COOKIE_PATH = '/'
# Whether to use the HttpOnly flag.
SESSION_COOKIE_HTTPONLY = True
# Whether to set the flag restricting cookie leaks on cross-site requests.
# This can be 'Lax', 'Strict', 'None', or False to disable the flag.
SESSION_COOKIE_SAMESITE = 'Lax'
# Whether to save the session data on every request.
SESSION_SAVE_EVERY_REQUEST = False
# Whether a user's session cookie expires when the Web browser is closed.
SESSION_EXPIRE_AT_BROWSER_CLOSE = False
# The module to store session data
SESSION_ENGINE = 'django.contrib.sessions.backends.db'
# Directory to store session files if using the file session module. If None,
# the backend will use a sensible default.
SESSION_FILE_PATH = None
# class to serialize session data
SESSION_SERIALIZER = 'django.contrib.sessions.serializers.JSONSerializer'
```

# 认证

```python
AUTH_USER_MODEL = 'auth.User'

AUTHENTICATION_BACKENDS = ['django.contrib.auth.backends.ModelBackend']

LOGIN_URL = '/accounts/login/'

LOGIN_REDIRECT_URL = '/accounts/profile/'

LOGOUT_REDIRECT_URL = None

# The number of days a password reset link is valid for
PASSWORD_RESET_TIMEOUT_DAYS = 3

# The number of seconds a password reset link is valid for (default: 3 days).
PASSWORD_RESET_TIMEOUT = 60 * 60 * 24 * 3

# the first hasher in this list is the preferred algorithm.  any
# password using different algorithms will be converted automatically
# upon login
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.PBKDF2PasswordHasher',
    'django.contrib.auth.hashers.PBKDF2SHA1PasswordHasher',
    'django.contrib.auth.hashers.Argon2PasswordHasher',
    'django.contrib.auth.hashers.BCryptSHA256PasswordHasher',
]

AUTH_PASSWORD_VALIDATORS = []
```

# 信号

```python
SIGNING_BACKEND = 'django.core.signing.TimestampSigner'
```

# 测试

```python
# The name of the class to use to run the test suite
TEST_RUNNER = 'django.test.runner.DiscoverRunner'

# Apps that don't need to be serialized at test database creation time
# (only apps with migrations are to start with)
TEST_NON_SERIALIZED_APPS = []
```

# 其他

## ABSOLUTE_URL_OVERRIDES

默认值： `{}` ，字典类型，Key 是 app_label.model_name 格式字符串，value 是一个方法，该方法接受一个 Django 模型对象并返回对应的 URL、通过该配置可以为每一个 Django 程序添加或重写 get_absolute_url()

```python
ABSOLUTE_URL_OVERRIDES = {
    'blogs.weblog': lambda o: "/blogs/%s/" % o.slug,
    'news.story': lambda o: "/stories/%s/%s/" % (o.pub_year, o.slug),
}
```

{{< hint danger >}}
**注意**

不管对应 model 是大写形式还是小写形式， Key 中的 model 字符串必须是小写形式
{{< /hint >}}

## FIXTURE_DIRS

默认值： `[]` ，一组保存 fixture 文件的目录。

## INSTALLED_APPS

默认值： `[]` ，一组当前 Django 项目可用的应用程序，每一个列表项都一个 Python 路径：一个应用程序的配置类或者一个包含应用程序的包

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    'post',
    'polls.apps.PollsConfig'
]
```

## DEFAULT_EXCEPTION_REPORTER_FILTER

默认值： `django.views.debug. SafeExceptionReporterFilter` ，当 HttpResponse 对象是 None 时所使用的默认异常过滤类

## MIDDLEWARE

默认值： `[]` ，Django 项目中所使用的中间件

```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

## AUTH_PASSWORD_VALIDATORS

默认值： `[]` ，密码验证器

```python
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]
```

## DEFAULT_AUTO_FIELD

默认值： `django.db.models. AutoField` ，默认模型主键字段类型

## DEBUG

默认值：False，用于指定当前网站是否运行在调试模式下。

* `DEBUG=True`：Django 会记录执行过的每一条 SQL 语句
* `DEBUG=False`：Django 会认为应用程序部署在生成环境，此时需要设置 `ALLOWED_HOSTS`

## DEBUG_PROPAGATE_EXCEPTIONS

默认值：False，如果为 True，Django 的视图方法出现异常或者需要返回 HTTP 500 时，异常信息将会被 Web 服务器接受并处理而不是有 Django 来处理。意思就是如果想看到详细异常信息就不应该设置为 True

## FORM_RENDERER

```python
FORM_RENDERER = 'django.forms.renderers.DjangoTemplates' # 用于渲染表单挂件的 Python 类
```
