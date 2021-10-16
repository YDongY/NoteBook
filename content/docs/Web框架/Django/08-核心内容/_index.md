# 1. 上下文处理器

> https://docs.djangoproject.com/zh-hans/3.2/ref/templates/api/#built-in-template-context-processors

上下文处理器是可以返回一些数据，在全局模板中都可以使用。在 `settings.TEMPLATES.OPTIONS.context_processors` 中，有许多内置的上下文处理器。这些上下文处理器的作用如下： 

* `django.template.context_processors.debug`：增加一个 `debug` 和 `sql_queries` 变量。在模板中可以通过它来查看到一些数据库查询
* `django.template.context_processors.request`：增加一个 `request` 变量。这个 `request` 变量也就是在视图函数的第一个参数
* `django.contrib.auth.context_processors.auth`：Django 有内置的用户系统，这个上下文处理器会增加一个 `user` 对象
* `django.contrib.messages.context_processors.messages`：增加一个 `messages` 变量

```python
def view(request):
    ......
    messages.add_message(request, level=messages.INFO, message='这是INFO信息')
    ......
```

```django
{% for message in messages %}
    {{ message.tags }} - {{ message }}
{% endfor %}
```

* `django.template.context_processors.media`：在模板中可以读取 `MEDIA_URL`
* `django.template.context_processors.static`：在模板中可以使用 `STATIC_URL`
* `django.template.context_processors.csrf`：在模板中可以使用 `csrf_token` 变量来生成一个 `csrf token`

**自定义上下文处理器**

> https://docs.djangoproject.com/zh-hans/3.2/ref/templates/api/#writing-your-own-context-processors

上下文处理器就是一个 Python 函数，可以存在于代码库中的任何地方，该函数接收一个 `HttpRequest` 对象，并返回一个添加到模板上下文的字典。

比如登录后的用户信息，在很多页面中都需要使用，那么我们可以放在上下文处理器中，就没有必要在每个视图函数中都返回这个对象传递给 `context` ，通过下面的上下文处理器，在所有的模板中都能够获取到 `userinfo` 对象。（注意：这里之所以不使用 `user` ，这是因为 Django 内置的上下文处理器中有一个 `user` 对象）

```python
def userinfo(request):
    user_id = request.session.get("user_id")
    user = User.objects.filter(pk=user_id).first()
    if user:
        return {'userinfo': user}
    else:
        return {}

# 配置上下文处理器
TEMPLATES = [
    {
        ......
        'OPTIONS': {
            'context_processors': [
                ......
                'app_context.context.userinfo'
            ],
        },
    },
]
```

# 2. 中间件

> https://docs.djangoproject.com/zh-hans/3.2/ref/middleware/

中间件是 Django 请求和响应处理的钩子框架。每一个中间件组件负责实现一些特定的功能。

1. 比如在 `request` 到达视图函数之前，判断当前这个用户有没有登录，如果登录了，就绑定一个 `user` 对象到 `request` 上。
2. 比如在 `response` 到达浏览器之前，想要统一在 `response` 上设置一些 `cookie` 信息等。

## 函数中间件

```python
def simple_middleware(get_response):
    # 中间件初始化的代码，当 Web 服务器启动时调用一次

    def middleware(request):
        # request 到达 view 之前执行的代码

        response = get_response(request)

        # response 到达浏览器之前执行的代码

        return response

    return middleware
```

## 类中间件

```python
class SimpleMiddleware(object):
    def __init__(self, get_response):
        self.get_response = get_response
        # 中间件初始化的代码，当 Web 服务器启动时调用一次

    def __call__(self, request):
        # request 到达 view 之前执行的代码

        response = self.get_response(request)

        # response 到达用户浏览器之前执行的代码

        return response
```

其中 `get_response` 是 Django 提供的一个可调用对象（callable），当中间件位于列表的最后一项， `get_response` 是真正的视图，否则就是下一个中间件。

比如我们写了一个在 `request` 到达视图函数之前，判断这个用户是否登录，如果已经登录就绑定一个 `user` 对象到 `request` 上的中间件，这个中间件放在 `app_middleware` 应用中的 `middlewares` 模块中：

```python
def user_middleware(get_response):
    print("user_middleware ==> init")

    def middleware(request):
        print("user_middleware ==> request before")

        user_id = request.session.get("user_id")
        user = User.objects.filter(pk=user_id).first()
        if user:
            setattr(request, 'userinfo', user)

        response = get_response(request)

        print("user_middleware ==> response before")

        return response

    return middleware
```

在写完中间件后，还需要在 `settings.MIDDLEWARE` 中配置写好的中间件才可以使用。

```python
MIDDLEWARE = [
    ......
    'app_middleware.middlewares.user_middleware'
]
```

## 老版本中的 Middleware

> https://docs.djangoproject.com/zh-hans/3.2/topics/http/middleware/#upgrading-pre-django-1-10-style-middleware

```python
class UserMiddleware(MiddlewareMixin):
    def __init__(self, get_response):
        """服务器响应第一个请求的时候调用一次，用于确定是否启用当前中间件"""
        super().__init__(get_response)

    def process_request(self, request):
        """request 对象产生之后，url 匹配之前调用，返回 None 或 HttpResponse对象"""
        pass

    def process_view(self, request, view_func, *view_args, **view_kwargs):
        """url 匹配之后，视图函数调用之前调用，返回 None 或 HttpResponse对象"""
        pass

    def process_response(self, request, response):
        """视图函数调用之后，所有响应返回浏览器之前被调用，返回 HttpResponse 对象"""
        pass

    def process_exception(self, request, exception):
        """视图抛出异常时调用，返回一个 HttpResponse 对象。"""
        pass
```

# 3. 缓存配置

> https://docs.djangoproject.com/zh-hans/3.2/topics/cache/

{{< tabs "缓存配置" >}}
{{< tab "Redis 缓存配置" >}}

> https://github.com/jazzband/django-redis

```python
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        # "LOCATION": "redis://127.0.0.1:6379/1",
        # "LOCATION": "redis://[:password]@localhost:6379/0"
        # "LOCATION": "rediss://[:password]@localhost:6379/0"
        # "LOCATION": "unix://[:password]@/path/to/socket.sock?db=0"
        "LOCATION": [ # 主从配置没有在生产环境中经过大量测试（警告）
            "redis://127.0.0.1:6379/1", # master 服务器
            "redis://127.0.0.1:6378/1", # slave 服务器
        ],
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
            "PASSWORD": "mysecret", # 密码，不会覆盖 uri 中的密码，如果你已经在 uri 中设置了密码, 此设置将被忽略
            "SOCKET_CONNECT_TIMEOUT": 5,  # socket 建立连接超时设置 单位：秒
            "SOCKET_TIMEOUT": 5,  # 连接建立后的读写操作超时设置 单位：秒
            "COMPRESSOR": "django_redis.compressors.zlib.ZlibCompressor", # 支持压缩, 但默认是关闭的.
            "CONNECTION_POOL_KWARGS": {"max_connections": 100} # 配置默认连接池
            # 客户端在将数据发给服务器之前先会序列化数据. django-redis 默认使用 Python pickle 序列化数据
            "SERIALIZER": "django_redis.serializers.json.JSONSerializer",
            # 还支持使用 MsgPack（需要 msgpack 库）进行序列化
            "SERIALIZER": "django_redis.serializers.msgpack.MSGPackSerializer",
        },
        "KEY_PREFIX": "KEY_PREFIX", # 所有缓存键前缀
        "TIMEOUT": 300,             # 过期的秒数，为 None 则不会过期
        "VERSION": 1,               # Django 服务器生成的缓存密钥的默认版本号 1
        "KEY_FUNCTION": 'app.cache.make_key' # 生成键名称的函数路径，用于覆盖默认键组合名称
    },
    "session": {  # 缓存 session
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        }
    },
}

# 配置 session 缓存后端
SESSION_ENGINE = "django.contrib.sessions.backends.cache"
SESSION_CACHE_ALIAS = "session"
```

默认情况下，缓存键不是指定的名称，它是与缓存前缀和键版本组合后的名称，使用冒号连接这三部分生成最终的字符串

```python
def make_key(key, key_prefix, version):
    return '%s:%s:%s' % (key_prefix, version, key)
```

如果想要自定义可以给 `CACHES` 配置 `KEY_FUNCTION` 参数，指定生成键名称的函数。

{{< /tab >}}
{{< tab "其他缓存配置" >}}

```python
# ------------- Memcached 缓存配置 -------------
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache', # 使用 https://pypi.org/project/pymemcache/
        'LOCATION': [
            '172.19.26.240:11211',
            '172.19.26.242:11212',
            '172.19.26.244:11213',
        ],
    }
}

# ------------- 数据库缓存配置 -------------
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.db.DatabaseCache',
        'LOCATION': 'my_cache_table', # 设置为数据库表名 tablename
    }
}

# 与其他缓存后端不同，数据库缓存不支持在数据库级别自动删除过期条目。
# 相反，每次调用 `add()` 、 `set()` 或 `touch()` 时都会删除过期的缓存条目. 
# 其次使用数据库缓存之前，必须通过下面的命令创建缓存表：python manage.py createcachetable

# ------------- 文件系统缓存配置 -------------
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.filebased.FileBasedCache',
        'LOCATION': '/var/tmp/django_cache', # 目录路径应该是绝对路径
    }
}

# 本地内存缓存配置，缓存使用最近最少使用（LRU）的淘汰策略
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
        'LOCATION': 'unique-snowflake', # 标识各个内存存储
    }
}
```

{{< /tab >}}
{{< /tabs >}}

## 视图缓存

使用缓存框架的通用办法是缓存视图结果。 `django.views.decorators.cache` 定义了一个 `cache_page` 装饰器，它将自动缓存视图的响应

```python
# ------------- views.py -------------
from django.shortcuts import render

from django.views.decorators.cache import cache_page

@cache_page(timeout=60 * 15, cache="page_cache", key_prefix="cache_prefix")
# timeout 缓存时间
# cache 指定缓存后端，默认情况下，将使用默认缓存
# key_prefix 覆盖缓存前缀
def cache_view(request):
    return render(request, 'cache_page.html')

# ------------- urls.py -------------
from django.views.decorators.cache import cache_page

urlpatterns = [
    path('cache-view/', views.cache_view, name='cache_view'),

    # 在 URLconf 中指定视图缓存
    path('cache-view/', cache_page(timeout=60 * 15, cache="page_cache", key_prefix="cache_view")(views.cache_view),
         name='cache_view'),
]
```

## 模板片段缓存

```django
{% load cache %} {# 放在模板顶部 #}
{% cache 500 sidebar %}  {# 缓存时间为 500s，缓存片段的名称为 sidebar  #}
    .. sidebar ..
{% endcache %}

{% cache None sidebar %} {# 缓存时间为 None，则永久缓存  #}
    .. sidebar ..
{% endcache %}

{% cache None sidebar %} {# 缓存时间为 None，则永久缓存  #}
    .. sidebar ..
{% endcache %}

{% cache my_timeout sidebar %} {# 缓存失效时间可以是模板变量 #}
    .. sidebar ..
{% endcache %}
```

默认情况下，缓存标签会先尝试使用名为 `"template_fragments"` 的缓存。如果这个缓存不存在，它将回退使用默认缓存。也可以选择一个备用缓存后端与 `using` 关键字参数一起使用，这个参数必须是标签的最后一个参数

```django
{% cache 300 sidebar using="localcache" %} {# 指定缓存模板后端为 localcache #}
    .. sidebar ..
{% endcache %}
```

## 访问缓存

```python
from django.core.cache import caches
default = caches['default]

# 上面两句等价于直接导入
from django.core.cache import cache

# ------------- 添加 -------------
# timeout 参数是可选的，默认为 CACHES 中相应后端的 timeout 参数，它是值存在缓存里的秒数
# timeout 设置为 None 时将永久缓存。timeout 为 0 将不缓存值
>>> cache.set('name', 'jack', 30) 
>>> cache.add('age', 20)                    # 键不存在，添加键。键已经存在，将不会更新缓存
>>> cache.get_or_set('gender', 'male', 100) # 键存在，则返回，键不存在则设置

# ------------- 获取 -------------
>>> cache.get('name')
>>> cache.get('name', 'default value')          # 对象不在缓存中，将返回指定的值
>>> cache.get_many(['name', 'age', 'gender'])   # 获取多个值，返回一个字典
{'name':'jack', 'age':20, 'gender':'male'}
>>> cache.set_many({'name':'jack', 'age':20, 'gender':'male'})  # 设置多个值

# ------------- 删除 -------------
>>> cache.delete('name')    # 键被成功删除返回 True，失败返回 False
>>> cache.delete_many(['name', 'age', 'gender'])
>>> cache.clear()           # 删除缓存里的所有键

# ------------- 过期时间 -------------
>>> cache.touch('name', 10) # 设置一个新的过期时间

# ------------- 递增或递减 -------------
>>> cache.set('num', 1)
>>> cache.incr('num')
2
>>> cache.incr('num', 10)
12
>>> cache.decr('num')
11
>>> cache.decr('num', 5)
6 

# ------------- 关闭缓存的连接 -------------
>>> cache.close()

# ------------- 缓存版本控制 ---------------
>>> cache.set('name', 'jack', version=2)    # 设置键版本为 2
>>> cache.get('name')                       # 默认获取键的版本为 1
>>> cache.get('name', version=2)            # 获取版本为 2 的键
>>> cache.incr_version('name')              # 增加键的版本
>>> cache.decr_version('name')              # 减少键的版本
```

## 下游缓存

以上的缓存方法都是在 Django 服务端对数据进行缓存，除此之外，可以通过设置 HTTP 响应头使缓存由“下游”缓存执行，这里的下游缓存包括：代理缓存、浏览器等。

默认情况下，Django 的缓存系统根据请求的完整 URL 创建它的缓存密钥，例如 `"https://www.example.com/stories/2005/?order_by=author"` 这意味着对该 URL 的每个请求都将使用相同的缓存版本，而不管用户代理差异。但是，如果这个页面需要基于请求头（如 cookie、语言或用户代理）中的某些差异而产生不同的内容，则需要使用 `Vary` 标头来告诉缓存机制

```python
from django.views.decorators.vary import vary_on_headers, vary_on_cookie

@vary_on_headers('Cookie') # 等价于 @vary_on_cookie
def my_view(request):
    ...

@vary_on_headers('User-Agent', 'Cookie') # 传递多个头参数，参数全部都相同，则使用相同的缓存版本
def my_view(request):
    ...

from django.utils.cache import patch_vary_headers

def my_view(request):
    ...
    response = render(request, 'template_name', context)
    patch_vary_headers(response, ['Cookie'])
    return response

from django.views.decorators.cache import cache_control

@cache_control(private=True) # 指出一个页面的缓存应该是“私有的”
def my_view(request):
    ...

from django.views.decorators.cache import cache_control

@cache_control(max_age=3600) # 没有使用 Django 服务器端的缓存框架，仍然可以告诉客户端使用 max-age 指令缓存视图一段时间
def my_view(request):
    ...

from django.views.decorators.cache import never_cache

@never_cache    # 完全禁用缓存，用来添加头部确保响应不被浏览器或其他缓存进行缓存
def myview(request):
    ...
```

# 4. 日志

{{< expand "日志配置" >}}

```python
LOGGING = {
    'version': 1,  # dictConfig 格式版本，目前只有 1
    'disable_existing_loggers': False,  # 是否禁用已经存在的日志器
    'formatters': {  # 日志信息显示的格式
        'verbose': {  # 输出日志级别名称、日志信息，以及生成日志信息的时间、进程、线程和模块
            'format': '%(levelname)s %(asctime)s %(module)s %(lineno)d %(message)s'
        },
        'simple': {
            'format': '%(levelname)s %(module)s %(lineno)d %(message)s'
        },
    },
    'filters': {  # 定义两个过滤器
        'require_debug_true': {  # django 在 debug 模式下才输出日志
            '()': 'django.utils.log.RequireDebugTrue',
        },
        'special': {
            '()': 'project.logging.SpecialFilter',
            'foo': 'bar',  # 当实例化 SpecialFilter 时，参数 foo 将被赋予一个 bar 的值
        },
    },
    'handlers': {  # 定义两个处理程序
        # console 表示仅在 debug 模式下，它将 INFO 以及更高级别的消息打印到 sys.stderr。该处理程序使用 simple 输出格式
        'console': {
            'level': 'INFO',
            'filters': ['require_debug_true'],
            'class': 'logging.StreamHandler',
            'formatter': 'simple'
        },
        # file 表示将 INFO 以及更高级别的消息输出到指定日志文件，该处理程序使用 verbose 输出格式，每个文件大小 300M 最多 10 个
        'file': {
            'level': 'INFO',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': BASE_DIR / "log.log",  # 日志文件的位置
            'maxBytes': 300 * 1024 * 1024,
            'backupCount': 10,
            'formatter': 'verbose'
        },
    },
    'loggers': {  # 配置记录器
        # 定义了一个名为 django 的记录器，可以同时向终端（sys.stderr）与文件中输出日志
        'django': {
            'handlers': ['console', 'file'],
            'propagate': True,  # 是否继续传递日志信息
            'level': 'INFO',  # 日志器接收的最低日志级别
        },
    }
}
```

{{< /expand >}}

* **使用日志**

```python
import logging

# 获取logger
logger = logging.getLogger('django')

def view(request):
    logger.info('info')
    logger.error('error')
    logger.warning('warning')
    logger.critical('critical')
    logger.debug('debug')
```

# 5. 发送邮件

> https://docs.djangoproject.com/zh-hans/3.2/topics/email/

## send_mail 和 send_mass_mail

```python
send_mail(subject, message, from_email, recipient_list,
              fail_silently=False, auth_user=None, auth_password=None,
              connection=None, html_message=None)

send_mail('Subject here', 'Here is the message.', 'from@example.com', ['to@example.com'], fail_silently=False)

message1 = ('Subject here', 'Here is the message', 'from@example.com', ['first@example.com', 'other@example.com'])
message2 = ('Another Subject', 'Here is another message', 'from@example.com', ['second@test.com'])

# 批量发送邮件
send_mass_mail((message1, message2), fail_silently=False)
```

* `subject`：邮件主题
* `message`：邮件消息，类型则是 `text/plain`
* `from_email`：发送邮件地址，如果为 None ，Django 将使用 `DEFAULT_FROM_EMAIL` 设置的值
* `recipient_list`：接收邮箱地址列表，每个成员都可以在邮件的 "收件人:" 中看到其他的收件人
* `fail_silently`：若为 `False`， `send_mail()` 会在发生错误时抛出 `smtplib. SMTPException`
* `auth_user`：可选的用户名，用于验证登陆 SMTP 服务器。若未提供，Django 会使用 `EMAIL_HOST_USER` 指定的值
* `auth_password`：可选的密码，用于验证登陆 SMTP 服务器。若未提供， Django 会使用 `EMAIL_HOST_PASSWORD` 指定的值
* `connection`：可选参数，发送邮件使用的后端。若未指定，则使用默认的后端
* `html_message`：邮件消息，类型则是 `text/html`

## EmailMessage 类

EmailMessage 用于创建邮件消息，上面的 `send_mail()` 和 `send_mass_mail()` 函数就是对该类 `EmailMessage` 的简单封装利用

```python
# from django.core.mail import EmailMessage

class EmailMessage:
    def __init__(self, subject='', body='', from_email=None, to=None, bcc=None,
                 connection=None, attachments=None, headers=None, cc=None,
                 reply_to=None):
                 ......
```

* `subject`: 邮件的主题
* `body`: 邮件内容，需要为纯文本格式
* `from_email`: 发件人地址
* `to`: 一个包含收件人地址的列表或元组
* `bcc`: 一个包含地址的列表或元组，指定“密送”对象
* `connection`: 一个邮件后端的实例。若在发送多份邮件时，若想复用连接，则设置此参数。如果省略，在调用 `send()` 时总会创建新连接
* `attachments`: 附加在邮件中的附件列表。可以是 `MIMEBase` 的实例，或 (文件名，内容，`mimetype`) 的元组。
* `headers`: 一个字典，包含邮件中额外的头信息。字典的关键字是头的名称，值为头的值
* `cc`: 一个包含收件人地址的列表或元组，指定“抄送”对象
* `reply_to`: 一个包含收件人地址的列表或元组，指定“回复”对象

## 邮件配置与测试

```python
# EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
# EMAIL_HOST = 'localhost'
EMAIL_HOST = 'smtp.163.com'
# EMAIL_PORT = 25
EMAIL_PORT = 25
EMAIL_USE_LOCALTIME = False
EMAIL_HOST_USER = '😜'
EMAIL_HOST_PASSWORD = '😜'
EMAIL_USE_TLS = False
EMAIL_USE_SSL = False
EMAIL_SSL_CERTFILE = None
EMAIL_SSL_KEYFILE = None
EMAIL_TIMEOUT = None

# DEFAULT_FROM_EMAIL = 'webmaster@localhost'
DEFAULT_FROM_EMAIL = 'ydongy97@163.com'  # 收件人
```

{{< tabs "邮件配置与测试" >}}
{{< tab "单个邮件发送" >}}

```python
from django.http import HttpResponse
from django.core.mail import send_mail, send_mass_mail

def send_mail_view(request):
    ret = send_mail(
        subject='Subject',
        message='message',
        from_email=None,  # 使用 DEFAULT_FROM_EMAIL
        recipient_list=['ydongy97@163.com']
    )
    print(ret)  # 1
    return HttpResponse("send_mail_view success")
```

{{< /tab >}}
{{< tab "多个邮件发送" >}}

```python
def send_mass_mail_view(request):
    message1 = ('Subject 1', 'message 1', 'ydongy97@163.com', ['ydongy97@163.com'])
    message2 = ('Subject 2', 'message 2', 'ydongy97@163.com', ['ydongy97@163.com'])
    ret = send_mass_mail((message1, message2), fail_silently=False)
    print(ret)  # 2
    return HttpResponse("send_mass_mail_view success")
```

{{< /tab >}}
{{< tab "附件邮件发送" >}}

```python
from django.core.mail import EmailMessage
import os

def email_message_view(request):
    message = EmailMessage(
        subject='Subject',
        body='Body',
        from_email='ydongy97@163.com',
        to=['ydongy97@163.com'],
        cc=['ydongy97@163.com'],
        bcc=['ydongy97@163.com'],
        reply_to=['ydongy97@163.com'],
        headers={'Message-ID': 'foo'},
    )

    # 发送附件，方式一： message.attach('name.png', img_data, 'image/png')
    # with open(os.path.join(os.path.dirname(__file__), 'design.jpg'), 'rb') as f:
    #     img_data = f.read()
    # message.attach('design.jpg', img_data, 'image/jpg')

    # 发送附件，方式二：message.attach_file('/images/weather_map.png')
    message.attach_file(os.path.join(os.path.dirname(__file__), 'design.jpg'))

    message.send(fail_silently=False)

    return HttpResponse("email_message_view success")
```

{{< /tab >}}
{{< tab "指定类型邮件发送" >}}

```python
# 通过 send_mail 指定 html_message
def send_html_mail_view(request):
    ret = send_mail(
        subject='Subject',
        message=None,
        from_email=None,  # 使用 DEFAULT_FROM_EMAIL
        recipient_list=['ydongy97@163.com'],
        html_message='<h1>message</h1>',
    )
    print(ret)  # 1
    return HttpResponse("send_mail_view success")

# 通过 EmailMultiAlternatives.attach_alternative 指定不同类型的消息
from django.core.mail import EmailMultiAlternatives

def email_multi_alternatives_view(request):
    message = EmailMultiAlternatives(
        subject='Subject',
        body='Body',
        from_email='ydongy97@163.com',
        to=['ydongy97@163.com'],
        cc=['ydongy97@163.com'],
        bcc=['ydongy97@163.com'],
        reply_to=['ydongy97@163.com'],
        headers={'Message-ID': 'foo'},
    )

    message.attach_alternative("<h1>Message</h1>", "text/html")
    message.send()

    return HttpResponse("email_multi_alternatives_view success")

# 通过 EmailMessage.content_subtype 指定消息类型
def email_message_view(request):
    message = EmailMessage(
        ...
    )
    message.content_subtype = 'html'
    message.send(fail_silently=False)

    return HttpResponse("email_message_view success")
```

{{< /tab >}}
{{< /tabs >}}

# 6. 分页

> https://docs.djangoproject.com/zh-hans/3.2/topics/pagination/

Django 提供了 `Paginator` 和 `Page` 类都是用来做分页的，分别位于 `django.core.paginator.Paginator` 和 `django.core.paginator. Page`

{{< tabs "分页" >}}
{{< tab "Paginator 常用属性和方法" >}}

| 属性         | 描述                                        |
| ------------ | ------------------------------------------- |
| `count` | 总共有多少条数据                            |
| `num_pages` | 总共有多少页                                |
| `page_range` | 页面的区间。比如有三页，那么就 `range(1,4)` |

{{< /tab >}}
{{< tab "Page 常用属性和方法" >}}

| 属性                   | 描述                             |
| ---------------------- | -------------------------------- |
| `has_next` | 是否还有下一页                   |
| `has_previous` | 是否还有上一页                   |
| `next_page_number` | 下一页的页码                     |
| `previous_page_number` | 上一页的页码                     |
| `number` | 当前页                           |
| `start_index` | 当前这一页的第一条数据的索引值   |
| `end_index` | 当前这一页的最后一条数据的索引值 |
| `object_list` | 当前分页的对象列表 |

{{< /tab >}}
{{< /tabs >}}

{{< expand "分页案例" >}}

* **视图**

```python
from django.shortcuts import render
from django.core.paginator import Paginator
from app_serializer.models import Article

def article_paginator_view(request):
    """函数视图"""
    articles = Article.objects.order_by('title')
    page = request.GET.get('p', 1)
    # 每页显示多少条数据
    paginator = Paginator(object_list=articles, per_page=2)
    # page 表示当前页
    page_obj = paginator.page(page)
    return render(request, 'article_list.html', {'paginator': paginator, "page_obj": page_obj})

class ArticleListView(ListView):
    """通用类视图"""
    paginate_by = 2
    model = Article
    page_kwarg = 'p'
    ordering = 'title'
    template_name = 'article_list.html'
```

* **路由**

```python
from django.urls import path
from . import views

app_name = 'paginator'
urlpatterns = [
    path('page/', views.article_paginator_view, name='article_paginator_view')
]
```

* **模板**

```django
<!doctype html>
<html lang="en">
<head>
    <title>Document</title>
    <!-- 最新版本的 Bootstrap 核心 CSS 文件 -->
    <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/3.4.1/css/bootstrap.min.css"
          integrity="sha384-HSMxcRTRxnN+Bdg0JdbxYKrThecOKuH5zCYotlSAcp1+c8xmyTe9GYg1l9a69psu" crossorigin="anonymous">

    <!-- 最新的 Bootstrap 核心 JavaScript 文件 -->
    <script src="https://stackpath.bootstrapcdn.com/bootstrap/3.4.1/js/bootstrap.min.js"
            integrity="sha384-aJ21OjlMXNL5UyIl/XNwTMqvzeRMZH2w8c5cRVpzpU8Y5bApTppSuUkhZXN0VxHd"
            crossorigin="anonymous"></script>
</head>
<body>
<nav aria-label="Page navigation">
    <ul>
        {% for object in page_obj.object_list %}
            <li>{{ object.title }} - {{ object.content }}</li>
        {% endfor %}
    </ul>
    <ul class="pagination">
        {% if page_obj.has_previous %}
            <li><a href="{% url 'paginator:article_paginator_view' %}?p={{ page_obj.previous_page_number }}"
                   aria-label="Previous"><span aria-hidden="true">上一页</span></a></li>
        {% else %}
            <li class="disabled"><a href="#" aria-label="Previous"><span aria-hidden="true">上一页</span></a></li>
        {% endif %}

        {% for page in paginator.page_range %}
            {% if page == page_obj.number %}
                <li class="active"><a href="javascript:void(0)">{{ page }}</a></li>
            {% else %}
                <li><a href="{% url 'paginator:article_paginator_view' %}?p={{ page }}">{{ page }}</a></li>
            {% endif %}
        {% endfor %}

        {% if page_obj.has_next %}
            <li><a href="{% url 'paginator:article_paginator_view' %}?p={{ page_obj.next_page_number }}"
                   aria-label="Previous"><span aria-hidden="true">下一页</span></a></li>
        {% else %}
            <li class="disabled"><a href="#" aria-label="Previous"><span aria-hidden="true">下一页</span></a></li>
        {% endif %}
    </ul>
</nav>
</body>
</html>
```

{{< /expand >}}

# 7. 消息框架

> https://docs.djangoproject.com/zh-hans/3.2/ref/contrib/messages/

消息是通过一个中间件类和相应的上下文处理器来实现的，通过 `django-admin startproject` 创建的项目默认配置了消息框架：

```python
INSTALLED_APPS = [
    ......
    'django.contrib.messages',
    ......
]

MIDDLEWARE = [
    ......
    'django.contrib.sessions.middleware.SessionMiddleware',
    ......
    'django.contrib.messages.middleware.MessageMiddleware',
    ......
]

TEMPLATES = [
    {
        ......
        'OPTIONS': {
            'context_processors': [
                ......
                'django.contrib.messages.context_processors.messages',
                ......
            ],
        },
    },
]
```

* **添加一个消息**

```python
from django.shortcuts import render
from django.contrib import messages

def messages_view(request):
    messages.add_message(request, messages.INFO, 'Hello world.')
    messages.debug(request, '%s SQL statements were executed.' % 2)
    messages.info(request, 'Three credits remain in your account.')
    messages.success(request, 'Profile details updated.')
    messages.warning(request, 'Your account expires in three days.')
    messages.error(request, 'Document deleted.')
    return render(request, 'messages.html')

```

* **显示消息**

```django
{% if messages %}
    <ul class="messages">
        {% for message in messages %}
            <li{% if message.tags %} class="{{ message.tags }}"{% endif %}>{{ message }}</li>
        {% endfor %}
    </ul>
{% endif %}
```

# 8. 序列化

> https://docs.djangoproject.com/zh-hans/3.2/topics/serialization/

{{< expand "测试模型" >}}

```python
from django.db import models
from django.contrib.auth.models import User

class AuthorManager(models. Manager):

    def get_by_natural_key(self, name):
        return self.get(name=name)

class Author(models. Model):

    name = models.CharField(max_length=100, unique=True)
    birthdate = models.DateField()

    objects = AuthorManager()

    def __str__(self):
        return self.name

    def natural_key(self):
        return self.name

class Article(models. Model):

    title = models.CharField(max_length=20)
    content = models.TextField()
    create_time = models.DateTimeField(auto_now_add=True)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)

    def __str__(self):
        return self.title
```

{{< /expand >}}

{{< tabs "序列化" >}}
{{< tab "直接序列化" >}}

```python

from django.core import serializers

queryset = Article.objects.all()

# 直接序列化
json_data = serializers.serialize('json', queryset)
xml_data = serializers.serialize('json', queryset)
yaml_data = serializers.serialize('yaml', queryset) # 需要安装 PyYAML
json_data = [{"model": "app_serializer.article", "pk": 1,
                "fields": {"title": "红楼梦", "content": "林黛玉和贾宝玉", "create_time": "2021-10-15T14:37:03.318Z",
                            "author": 1}},
                {"model": "app_serializer.article", "pk": 2,
                "fields": {"title": "西游记", "content": "三打白骨精", "create_time": "2021-10-15T14:37:18.015Z",
                            "author": 2}},
                {"model": "app_serializer.article", "pk": 3,
                "fields": {"title": "水浒传", "content": "武松打虎", "create_time": "2021-10-15T14:37:33.363Z",
                            "author": 3}},
                {"model": "app_serializer.article", "pk": 4,
                "fields": {"title": "三国演义", "content": "火烧赤壁", "create_time": "2021-10-15T14:37:46.522Z",
                            "author": 4}}]
                             
# 指定序列化字段
json_data = serializers.serialize('json', queryset, fields=('title', 'content'))
```

{{< /tab >}}
{{< tab "创建序列化器对象" >}}

```python
XMLSerializer = serializers.get_serializer("xml")
xml_serializer = XMLSerializer()
xml_serializer.serialize(queryset)
xml_data = xml_serializer.getvalue()
```

{{< /tab >}}
{{< tab "序列化数据到文件" >}}

```python
XMLSerializer = serializers.get_serializer("xml")
xml_serializer = XMLSerializer()
with open("file.xml", "w") as out:
    xml_serializer.serialize(queryset, stream=out)
```

{{< /tab >}}
{{< tab "反序列化" >}}

```python
for obj in serializers.deserialize("xml", data):
    # obj 是 DeserializedObject 实例，调用 DeserializedObject.save() 保存对象到数据库
    # obj.save() # 保存对象到数据库
    do_something_with(obj)

# 如果模型中不存在序列化字段，将引发 DeserializationError 错误
# 除非将 ignorenonexistent 参数为 True 传入
serializers.deserialize("xml", data, ignorenonexistent=True)
```

{{< /tab >}}
{{< /tabs >}}

# 9. 静态文件管理

```python
# 存放 collectstatic 命令收集的静态文件
STATIC_ROOT = "/var/www/example.com/static/"

# 当访问 URL 路径为 /static/ 或者 http://static.example.com/ 时将访问 STATIC_ROOT 中的静态文件
STATIC_URL = "/static/"

# 静态文件查找目录，collectstatic 和 findstatic 命令也会收集和查找这些目录
STATICFILES_DIRS = [
    "/home/special.polls.com/polls/static",
    "/home/polls.com/polls/static",
    "/opt/webfiles/common",

    # 假设 STATIC_URL = '/static/'，则 collectstatic 管理命令将收集 STATIC_ROOT 的 'downloads' 子目录中的 stats 文件
    # 允许在模板中用 /static/downloads/polls_20101022.tar.gz 引用本地文件 /opt/webfiles/stats/polls_20101022.tar.gz
    ("downloads", "/opt/webfiles/stats"), # (prefix, path)
]

# 使用 collectstatic 管理命令收集静态文件时要使用的文件存储引擎
STATICFILES_STORAGE = 'django.contrib.staticfiles.storage.StaticFilesStorage'

# 指定静态文件查找器，如果存在多个同名文件，将使用第一个找到的文件
STATICFILES_FINDERS = [
    # 查找存储在 STATICFILES_DIRS 配置中的文件
    'django.contrib.staticfiles.finders.FileSystemFinder',

    # 每个应用程序的 static 子目录中的文件
    'django.contrib.staticfiles.finders.AppDirectoriesFinder',

    # 默认禁用，如果添加到该配置中，它将在 DEFAULT_FILE_STORAGE 配置所定义的默认文件存储中查找静态文件。
    # 'django.contrib.staticfiles.finders.DefaultStorageFinder'
]
```

# 10. contenttypes 框架

> https://docs.djangoproject.com/zh-hans/3.2/ref/contrib/contenttypes/

## ContentType 模型

Django 在 `INSTALLED_APPS` 包含了一个 `django.contrib.contenttypes` 应用程序，它可以跟踪所有安装在 Django 项目中的模型。其应用的核心是 `ContentType` 模型：

```python
class ContentType(models.Model):
    # 模型所属应用程序的名称
    app_label = models.CharField(max_length=100)
    # 模型类的名称
    model = models.CharField(_('python model class name'), max_length=100)
    objects = ContentTypeManager()
```

例如，我们可以查找 `ContentType` 的 `User` 模型：

```python
>>> from django.contrib.contenttypes.models import ContentType
>>> user_type = ContentType.objects.get(app_label='auth', model='user')
>>> user_type
<ContentType: user>
>>> user_type.model_class() # 返回当前 ContentType 实例所代表的模型类
<class 'django.contrib.auth.models.User'>
>>> user_type.get_object_for_this_type(username='Guido') # 根据当前 ContentType 实例获取模型对象
<User: Guido>
```

## Generic relations

在 ContentType 中添加一个模型外键，可以实现模型与模型之间的任意绑定，例如下面的 Comment 评论模型既可以给 Post （博客）评论也可以给 Article（文章评论）：

```python
from django.contrib.auth.models import User
from django.contrib.contenttypes.fields import GenericForeignKey, GenericRelation
from django.contrib.contenttypes.models import ContentType
from django.db import models

class Comment(models.Model):
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    body = models.TextField(blank=True)
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveIntegerField()  # 这里需要设定为关联模型的主键类型
    content_object = GenericForeignKey('content_type', 'object_id')  # 等价于 GenericForeignKey() 不能设置 on_delete 默认级联删除

    # content_type : 唯一确定某个应用下的某个模型类
    # object_id : 唯一确定某个模型类的某一条记录

    class Meta:
        # 查询优化，因为 content_type, object_id 是一起使用的，用于唯一确定某个应用下的某个模型类中的某一条记录
        index_together = ('content_type', 'object_id')

class Post(models.Model):
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    title = models.CharField(max_length=20)
    content = models.TextField(blank=True)
    comments = GenericRelation(Comment)

class Article(models.Model):
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    title = models.CharField(max_length=20)
    content = models.TextField(blank=True)
    comments = GenericRelation(Comment, related_query_name='article')
    # comments = GenericRelation(Comment, related_query_name='article',
    #                            content_type_field='content_type_fk',
    #                            object_id_field='object_primary_key')
    # 后面两个字段用于关联查询
```

{{< tabs "通用关系" >}}
{{< tab "添加删除" >}}

```python
from .models import Comment, Post, Article
from django.contrib.auth.models import User

# 1. 创建对象添加
user = User.objects.get(id=1)
post = Post.objects.create(author=user, title='西游记', content='三打白骨精')
comment1 = Comment.objects.create(author=user, body='三打白骨精这一集孙悟空被师父冤枉了', content_object=post)

# 2. add
comment2 = Comment(author=user, body='孙悟空心态炸了')
post.comments.add(comment2, bulk=False)

# 3. create
post.comments.create(author=user, body='孙悟空太难了')

# 4. set
post.comments.set([comment1, comment2]) # 删除其他关联的 comment，仅存在 comment1,comment2

# 5. remove
post.comments.remove(comment1) # 将批量删除指定的模型对象

# 6. clear
post.comments.clear() # 批量删除一个实例的所有相关对象
```

{{< /tab >}}
{{< tab "关联查询" >}}

```python
article1 = Article.objects.create(author=user,title='Django',content='大而全的 Python Web 框架')
comment1 = Comment.objects.create(author=user, body='很好啊',content_object=article1)
article2 = Article.objects.create(author=user,title='Flask',content='小而美的 Python Web 框架')
comment2 = Comment.objects.create(author=user, body='很好啊',content_object=article1)

# 根据 Comment 的 related_query_name 查询 article__title__contains='Django' 
comment = Comment.objects.filter(article__title__contains='Django')

# 也可以手动进行相同类型的查询
article = Article.objects.filter(title__contains='Django')
article_type = ContentType.objects.get_for_model(Article)
Comment.objects.filter(content_type__pk=article_type.id, object_id__in=article)

# 如果定义了 comments = GenericRelation(Comment, related_query_name='article',
#                               content_type_field='content_type_fk',
#                               object_id_field='object_primary_key')
Comment.objects.filter(content_type_fk__pk=article_type.id, object_primary_key__in=article)
```

{{< /tab >}}
{{< /tabs >}}

# 11. 信号

> https://docs.djangoproject.com/zh-hans/3.2/topics/signals/

```python
django.db.models.signals.pre_save       # 一个模型的 save() 方法被调用之前
django.db.models.signals.post_save      # 一个模型的 save() 方法被调用之后
django.db.models.signals.pre_delete     # 一个模型的 delete() 方法或查询结果集的 delete() 方法被调用之前
django.db.models.signals.post_delete    # 一个模型的 delete() 方法或查询结果集的 delete() 方法被调用之后
django.db.models.signals.m2m_changed    # 一个模型的 ManyToManyField 更改后发出
django.core.signals.request_started     # Django 发起一个 HTTP 请求后
django.core.signals.request_finished    # Django 结束一个 HTTP 请求后
```

* **测试模型**

```python
from django.db import models

class Post(models.Model):
    title = models.CharField(max_length=20)

    def __str__(self):
        return self.title
```

{{< tabs "信号" >}}
{{< tab "pre_save/post_save" >}}

```python
def pre_save_post(sender, instance, **kwargs):
    print(sender)
    print(instance)
    print("pre save do something")

def post_save_post(sender, instance, **kwargs):
    print(sender)   # 发送者，例如：<class 'app_signal.models.Post'>
    print(instance) # 模型实例，例如：Post 对象
    print("post save do something")

pre_save.connect(pre_save_post, sender=Post)
post_save.connect(post_save_post, sender=Post)  # sender 用于指定监听 Post 模型的 save

# >>> Post.objects.create(title='first title') 
# >>> p = Post()
# >>> p.title = 'second title'
# >>> p.save()
# 先执行 pre_save_post 再执行 create/save 最后执行 post_save_post，create 本质还是调用 save
```

{{< /tab >}}
{{< tab "pre_delete/post_delete" >}}

```python
from django.db.models.signals import pre_delete, post_delete
from django.dispatch import receiver

@receiver(pre_delete, sender=Post)
def pre_delete_post(sender, instance, **kwargs):
    print("pre delete do something")

@receiver(post_delete, sender=Post)
def post_delete_post(sender, instance, **kwargs):
    print(sender)
    print(instance)
    print("post delete do something")

# >>> p = Post.objects.first()
# >>> p.delete()  
# >>> Post.objects.first().delete()
# 先执行 pre_delete_post 再执行 delete 最后执行 post_delete_post
```

{{< /tab >}}
{{< tab "request_started/request_finished" >}}

```python
from django.core.signals import request_finished, request_started
from django.dispatch import receiver

@receiver(request_started)
def request_started_signal(sender, **kwargs):
    print(sender)  # <class 'django.core.handlers.wsgi.WSGIHandler'>
    print("Request started!")

@receiver(request_finished)
def request_finished_signal(sender, **kwargs):
    print(sender)
    print("Request finished!")
```

{{< /tab >}}
{{< /tabs >}}

* **信号存放位置**

为了避免代码的耦合性，通常在应用目录下创建一个 `signals.py` 的子模块，该模块统一存放与当前应用相关的所有信号。由于信号接收器在应用目录下的 `apo.py` 中的配置类下的 `ready()` 方法中连接，所以只需要在该方法中导入 `signals.py` 即可。

```python
from django.apps import AppConfig

class AppSignalConfig(AppConfig):
    ......

    def ready(self):
        from . import signal
```

# 12. 文件上传

{{< expand "单一文件上传" >}}

1. 新建 forms.py 

```python
from django import forms

class UploadFileForm(forms.Form):
    file = forms.FileField()
```

2. 编写处理表单的视图

```python
import os

from django.http import HttpResponse
from django.shortcuts import render
from django.conf import settings

from .forms import UploadFileForm

def handle_uploaded_file(f):
    p = os.path.join(settings.MEDIA_ROOT, 'upload', f.name)
    with open(p, 'wb+') as destination:
        for chunk in f.chunks():
            destination.write(chunk)

def upload_file(request):
    if request.method == 'POST':
        form = UploadFileForm(request.POST, request.FILES)
        if form.is_valid():
            f = request.FILES['file']
            handle_uploaded_file(f)
            return HttpResponse("Success")
    else:
        form = UploadFileForm()
        return render(request, 'upload.html', {'form': form})
```

3. 编写模板文件

```django
<form action="{% url 'files:upload' %}" method="post" enctype="multipart/form-data">
    {% csrf_token %}
    {{ form }}
    <input type="submit" value="submit">
</form>
```

4. 配置路由

```python
from django.urls import path
from . import views

app_name = 'files'

urlpatterns = [
    path('upload/', views.upload_file, name='upload'),
]
```

![upload](upload.png "upload")

{{< /expand >}}

{{< expand "多文件上传" >}}

1. 编写 UploadMultiFileForm 表单

```python
class UploadMultiFileForm(forms.Form):
    file = forms.FileField(widget=forms.ClearableFileInput(attrs={'mutiple': True}))
```

2. 编写处理表单的视图

```python
from django.views.generic.edit import FormView
from django.conf import settings

from .forms import UploadMultiFileForm

def handle_uploaded_file(f):
    p = os.path.join(settings.MEDIA_ROOT, 'upload', f.name)
    with open(p, 'wb+') as destination:
        for chunk in f.chunks():
            destination.write(chunk)

class UploadMultiFileView(FormView):
    form_class = UploadMultiFileForm
    template_name = 'multyupload.html'
    success_url = '/'

    def post(self, request, *args, **kwargs):
        form_class = self.get_form_class()
        form = self.get_form(form_class)
        files = request.FILES.getlist('file')
        if form.is_valid():
            for f in files:
                handle_uploaded_file(f)
            return self.form_valid(form)
        else:
            return self.form_invalid(form)
```

3. 编写模板文件

```django
<form action="{% url 'files:multyupload' %}" method="post" enctype="multipart/form-data">
    {% csrf_token %}
    {{ form }}
    <input type="submit" value="submit">
</form>
```

4. 配置路由

```python
from django.urls import path
from . import views

app_name = 'files'

urlpatterns = [
    path('multyupload/', views.UploadMultiFileView.as_view(), name='multyupload'),
]
```

![multyupload](multyupload.png "multyupload")

{{< /expand >}}

{{< tabs "文件上传" >}}
{{< tab "使用模型来处理上传的文件" >}}

在定义模型的时候，我们可以给存储文件的字段指定为 FileField，这个 Field 可以传递一个 `upload_to` 参数，用来指定上传上来的文件保存到哪里。
​
比如我们让他保存到项目的 `files` 文件夹下，那么示例代码如下：

```python
from django.shortcuts import render
from django.http import HttpResponse
from django.db import models

# models.py
class Article(models.Model):
    title = models.CharField(max_length=100)
    content = models.TextField()
    thumbnail = models.FileField(upload_to="files")

# views.py
def index(request):
    if request.method == 'GET':
        return render(request, 'index.html')
    else:
        title = request.POST.get('title')
        content = request.POST.get('content')
        thumbnail = request.FILES.get('thumbnail')
        article = Article(title=title, content=content, thumbnail=thumbnail)
        article.save()
        return HttpResponse('success')
```

* **指定 MEDIA_ROOT 和 MEDIA_URL**

以上我们是使用了 `upload_to` 来指定上传的文件的目录。我们也可以指定 `MEDIA_ROOT` ，就不需要在 `FileField` 中指定 `upload_to` ，他会自动的将文件上传到 `MEDIA_ROOT` 的目录下。

```python
MEDIA_ROOT = os.path.join(BASE_DIR,'media')
MEDIA_URL = '/media/'
```

然后我们可以在 `urls.py` 中添加 `MEDIA_ROOT` 目录下的访问路径。示例代码如下：

```python
from django.urls import path
from front import views
from django.conf.urls.static import static
from django.conf import settings

urlpatterns = [
    # ...
] + static(settings.MEDIA_URL,document_root=settings.MEDIA_ROOT)
```

如果我们同时指定 `MEDIA_ROOT` 和 `upload_to` ，那么会将文件上传到 `MEDIA_ROOT` 下的 `upload_to` 文件夹中。示例代码如下：

```python
from django.db import models
class Article(models.Model):
    title = models.CharField(max_length=100)
    content = models.TextField()
    thumbnail = models.FileField(upload_to="%Y/%m/%d/")
```

{{< /tab >}}
{{< tab "限制上传的文件拓展名" >}}

如果想要限制上传的文件的拓展名，那么我们就需要用到表单来进行限制。我们可以使用普通的 Form 表单，也可以使用 ModelForm，直接从模型中读取字段。示例代码如下：

```python
from django.db import models
from django import forms
from django.core import validators
# models.py
class Article(models.Model):
    title = models.CharField(max_length=100)
    content = models.TextField()
    thumbnail = models.FileField(upload_to='%Y/%m/%d/',validators=[validators.FileExtensionValidator(['txt','pdf'])])

# forms.py
class ArticleForm(forms.ModelForm):
    class Meta:
        model = Article
        fields = "__all__"
```

{{< /tab >}}
{{< tab "上传图片" >}}

上传图片跟上传普通文件是一样的。只不过是上传图片的时候 Django 会判断上传的文件是否是图片的格式（除了判断后缀名，还会判断是否是可用的图片）。如果不是，那么就会验证失败。我们首先先来定义一个包含 `ImageField` 的模型。示例代码如下：

```python
from django.db import models
class Article(models.Model):
    title = models.CharField(max_length=100)
    content = models.TextField()
    thumbnail = models.ImageField(upload_to="%Y/%m/%d/")
```

**注意：使用 ImageField，必须要先安装 Pillow 库： `pip install pillow` 。**

因为要验证是否是合格的图片，因此我们还需要用一个表单来进行验证。表单我们直接就使用 ModelForm 就可以了。示例代码如下：

```python
from django import forms
class MyForm(forms.ModelForm):
    class Meta:
        model = Article
        fields = "__all__"
```

{{< /tab >}}
{{< /tabs >}}

**注意事项**

1. 只有 POST 请求方式才会触发文件上传动作
2. `request.FILES` 是一个字典对象，包含所有上传文件，字典的 Key 是表单类的字段名
3. 用于上传文件的表单需要包含 `enctype="multipart/form-data"` 属性
4. 为了避免 `read()` 方法一次性将文件读取到内存，使用 `f.chunks()` 方式将文件分块处理
5. 文件上传到 `settings.MEDIA_ROOT` 所指定的路径下的 `upload` 文件夹下，注意该路径必须提前创建好
6. 文件上传如果小于 2.5MB，那么 Django 会将文件保存在内存中。如果大于 2.5MB，Django 就会在系统的临时目录中生成临时文件。例如：`/tmp/tmpzfp616.upload` 文件。
