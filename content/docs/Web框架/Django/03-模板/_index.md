# 1. 模板渲染

```python
def index(request):
    # 1. 创建模板对象
    template = Template('<h1>{{ name }}</h1>')
    context = Context({"name": "jack"})
    content = template.render(context)
    return HttpResponse(content)

    # 2. 获取模板对象
    template = loader.get_template('index.html')
    content = template.render({"name": "jack"})
    return HttpResponse(content)

    # 3. render_to_string 加载一个模板 get_template()，并立即调用它的 render() 方法
    context = {"name": "jack"}
    content = loader.render_to_string('index.html', context, request)
    return HttpResponse(content)

    # 4. render 直接将模板渲染成字符串和包装成 HttpResponse 对象一步到位完成
    return render(request, 'index.html', context={"name": "jack"})
```

# 2. 模板查找路径配置

首先在项目的 `settings.py` 文件中。有一个 `TEMPLATES` 配置，这个配置包含了模板引擎的配置，模板查找路径的配置，模板上下文的配置等。

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR, 'templates')],
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

模板路径可以在两个地方配置：

1. `DIRS`：这是一个列表，在这个列表中可以存放所有的模板路径，以后在视图中使用 `render` 或者 `render_to_string` 渲染模板的时候，会在这个列表的路径中查找模板。
2. `APP_DIRS`：默认为 False，通过 `djang-admin` 创建的项目自动设置为 True，表示会在 `INSTALLED_APPS` 中安装过的应用下的 `templates` 目录中查找模板。

模板查找顺序：

1. 先会在 `DIRS` 这个列表中依次查找路径下有没有这个模板，如果有，就返回
2. 如果 `DIRS` 列表中所有的路径都没有找到，那么会先检查当前这个视图所处的 app 是否已经安装，如果已经安装了，那么就先在当前这个 app 下的 `templates` 文件夹中查找模板
3. 如果没有找到，那么会在其他已经安装了的 app 中查找
4. 如果所有路径下都没有找到，那么会抛出一个 `TemplateDoesNotExist` 的异常。
# 3. 模板变量

变量需要通过视图函数渲染，视图函数在使用 `render` 或者 `render_to_string` 的时候可以传递一个 `context` 的参数，这个参数是一个字典类型。以后在模板中的变量就从这个字典中读取值的。示例代码如下：

```python
from django.shortcuts import render

# views.py 代码
def profile(request):
    return render(request, 'profile.html', context={'username': '张三'})

# profile.html 模板代码
<p>{{ username }}</p>
```

模板中的变量同样也支持点( `.` )的形式，比如：

```python
from django.shortcuts import render

# views.py 代码
def profile(request):
    # person 是一个模型对象
    return render(request, 'profile.html', context={'person': person})

# profile.html 模板代码
<p>{{ person.username }}</p>
```

使用 `.` 的方式访问模板变量是按照以下方式进行解析的：

1. 如果 person 是一个字典，那么就会查找这个字典的 username 这个 key 对应的值。等价于 `person['username']`
2. 如果 person 是一个对象，那么就会查找这个对象的 username 属性。等价于 `person.username`
3. 如果 person 是一个对象，那么就会查找这个对象的 username 方法。等价于 `person.username()`
4. 如果出现的是 `person.1`，会判断 person 是否是一个列表或者元组或者任意的可以通过下标访问的对象，如果是的话就取这个列表的第 1 个值。如果不是就获取到的是一个空的字符串。等价于 `person[1]`

注意：

* 不能通过中括号的形式访问字典和列表中的值，比如 `dict['key']` 和 `list[1]` 是不支持的
* 同时对于字典而言，添加 key 的时候，千万不能和字典中的一些属性重复。比如 items，items 是字典的方法，那么如果给这个字典添加一个 items 作为 key，那么以后就不能再通过 `dict.items` 来访问这个字典的键值对了
# 4. 模板标签

> https://docs.djangoproject.com/zh-hans/3.2/ref/templates/builtins/#built-in-tag-reference

## if 标签

if 标签相当于 Python 中的 if 语句，有 elif 和 else 相对应，但是所有的标签都需要用标签符号 `{%...%}` 进行包裹。if
标签中可以使用 `==、!=、<、<=、>、>=、in、not in、is、is not` 等判断运算符。示例代码如下：

```django
{% if "张三" in persons %}
		<p>张三</p>
{% else %}
		<p>李四</p>
{% endif %}
```

## for...in... 标签

`for...in...` 类似于 Python 中的 `for...in...` 。可以遍历列表、元组、字符串、字典等一切可以遍历的对象。示例代码如下：

```django
{% for person in persons %}
    <p>{{ person.name }}</p>
{% endfor %}
```

如果想要反向遍历，那么在遍历的时候就加上一个 `reversed` 。示例代码如下：

```django
{% for person in persons reversed %}
    <p>{{ person.name }}</p>
{% endfor %}
```

遍历字典的时候，需要使用 `items` 、 `keys` 和 `values` 等方法。遍历字典示例代码如下：

```django
{% for key,value in person.items %}
    <p>key：{{ key }}</p>
    <p>value：{{ value }}</p>
{% endfor %}
```

在 for 循环中，DTL 提供了一些变量可供使用。这些变量如下：

* `forloop.counter`：当前循环的下标。以 1 作为起始值。
* `forloop.counter0`：当前循环的下标。以 0 作为起始值。
* `forloop.revcounter`：当前循环的反向下标值。比如列表有 5 个元素，那么第一次遍历这个属性是等于 5，第二次是 4，以此类推。并且是以 1 作为最后一个元素的下标
* `forloop.revcounter0`：类似于`forloop.revcounter`。不同的是最后一个元素的下标是从 0 开始。
* `forloop.first`：是否是第一次遍历。
* `forloop.last`：是否是最后一次遍历。
* `forloop.parentloop`：如果有多个循环嵌套，那么这个属性代表的是上一级的 for 循环。

## for...in...empty 标签

这个标签使用跟 `for...in...` 是一样的，只不过是在遍历的对象如果没有元素的情况下，会执行 `empty` 中的内容。示例代码如下：

```django
{% for person in persons %}
    <li>{{ person }}</li>
{% empty %}
    暂时还没有任何人
{% endfor %}
```

## with 标签

在模版中定义变量。有时候一个变量访问的时候比较复杂，那么可以先把这个复杂的变量缓存到一个变量上，以后就可以直接使用这个变量就可以了。示例代码如下：

```django
context = {
		"persons": ["张三","李四"]
}

{% with lisi=persons.1 %}
    <p>{{ lisi }}</p>
{% endwith %}
```

注意：

* 在 with 语句中定义的变量，只能在`{%with%}...{%endwith%}`中使用，不能在这个标签外面使用。
* 定义变量的时候，不能在**等号左右**两边留有空格。比如`{% with lisi = persons.1%}`是错误的。
* 还有另外一种写法同样也是支持的：

```django
{% with persons.1 as lisi %}
    <p>{{ lisi }}</p>
{% endwith %}
```

## url 标签

在模版中，我们经常要写一些 URL，比如某个 a 标签中需要定义 href 属性。当然如果通过硬编码的方式直接将这个 URL，写死在里面也是可以的。但是这样对于以后项目维护可能不是一件好事。因此建议使用这种反转的方式来实现，类似于 django 中的 reverse 一样。示例代码如下：

```django
<a href="{% url 'book:list' %}">图书列表页面</a>
```

如果 URL 反转的时候需要传递参数，那么可以在后面传递。但是参数分位置参数和关键字参数。位置参数和关键字参数不能同时使用。示例代码如下：

```django
# path 部分
path('detail/<book_id>/',views.book_detail,name='detail')

# url 反转，使用位置参数
<a href="{% url 'book:detail' 1 %}">图书详情页面</a>

# url 反转，使用关键字参数
<a href="{% url 'book:detail' book_id=1 %}">图书详情页面</a>
```

如果想要在使用 URL 标签反转的时候要传递查询字符串的参数，那么必须要手动在在后面添加。示例代码如下：

```django
<a href="{% url 'book:detail' book_id=1 %}?page=1">图书详情页面</a>
```

如果需要传递多个参数，那么通过空格的方式进行分隔。示例代码如下：

```django
<a href="{% url 'book:detail' book_id=1 page=2 %}">图书详情页面</a>
```

## spaceless 标签

移除 html 标签中的空白字符。包括空格、tab 键、换行等。示例代码如下：

```django
{% spaceless %}
    <p>
        <a href="foo/">Foo</a>
    </p>
{% endspaceless %}
```

那么在渲染完成后，会变成以下的代码：

```django
<p><a href="foo/">Foo</a></p>
```

spaceless 只会移除 html 标签之间的空白字符。而不会移除标签与文本之间的空白字符。例如以下代码将不会移除 strong 中的空白字符。

```django
{% spaceless %}
    <strong>
        Hello
    </strong>
{% endspaceless %}
```

## autoescape 标签

开启和关闭这个标签内元素的自动转义功能。比如 `<` 会被转义成 `&lt; ` ，而 `>` 会被自动转义成 `&gt; ` 。模板中默认是已经开启了自动转义的。autoescape 的示例代码如下：

```django
# 传递的上下文信息
context = {
 "info":"<a href='www.baidu.com'>百度</a>"
}

# 模板中关闭自动转义
{% autoescape on %}
    {{ info }}
{% endautoescape %}
```

那么就会显示百度的一个超链接。如果把 on 成 off，那么就会显示成一个普通的字符串。

## verbatim 标签

默认在 DTL 模板中是会去解析那些特殊字符的。比如 `{%` 和 `%}` 以及 `{{` 等。如果你在某个代码片段中不想使用 DTL 的解析引擎。那么你可以把这个代码片段放在 verbatim 标签中。示例代码下：

```django
{% verbatim %}
    {{if dying}}Still alive.{{/if}}
{% endverbatim %}
```

# 5. 模板过滤器

> https://docs.djangoproject.com/zh-hans/3.2/ref/templates/builtins/#built-in-filter-reference

过滤器使用的是 `|` 来使用。比如使用 add 过滤器，那么示例代码如下：

```django
{{ value|add:"2" }}
```

## add

1. 将值和参数转换成整形然后进行相加
2. 如果转换成整形过程中失败了，那么会将值和参数进行拼接。 
   - 如果是字符串，那么会拼接成字符串
   - 如果是列表，那么会拼接成一个列表

示例代码如下：

```django
{{ value|add:"2" }}
```

如果 value 是等于 4，那么结果将是 6。如果 value 是等于一个普通的字符串，比如 abc，那么结果将是 abc2。add 过滤器的源代码如下：

```python
def add(value, arg):
    """Add the arg to the value."""
    try:
        return int(value) + int(arg)
    except (ValueError, TypeError):
        try:
            return value + arg
        except Exception:
            return ''
```

## cut

移除值中所有指定的字符串。类似于 python 中的 `replace(args, "")` 。示例代码如下：

```django
{{ value|cut:" " }}
```

以上示例将会移除 value 中所有的空格字符。cut 过滤器的源代码如下：

```python
def cut(value, arg):
    """Remove all values of arg from the given string."""
    safe = isinstance(value, SafeData)
    value = value.replace(arg, '')
    if safe and arg != ';':
        return mark_safe(value)
    return value
```

## date

将一个日期按照指定的格式，格式化成字符串。示例代码如下：

```django
# 数据
context = {
    "birthday": datetime.now()
}

# 模版
{{ birthday|date:"Y/m/d" }}
```

那么将会输出 `2018/02/01` 。其中 Y 代表的是四位数字的年份，m 代表的是两位数字的月份，d 代表的是两位数字的日。

| 格式字符 | 描述 | 示例 |
| --- | --- | --- |
| Y | 四位数字的年份 | 2018 |
| m | 两位数字的月份 | 01-12 |
| n | 月份，1-9前面没有0前缀 | 1-12 |
| d | 两位数字的天 | 01-31 |
| j | 天，但是1-9前面没有0前缀 | 1-31 |
| g | 小时，12小时格式的，1-9前面没有0前缀 | 1-12 |
| h | 小时，12小时格式的，1-9前面有0前缀 | 01-12 |
| G | 小时，24小时格式的，1-9前面没有0前缀 | 1-23 |
| H | 小时，24小时格式的，1-9前面有0前缀 | 01-23 |
| i | 分钟，1-9前面有0前缀 | 00-59 |
| s | 秒，1-9前面有0前缀 | 00-59 |

## default

如果值被评估为 False。比如 `[]，""，None，{}` 等这些在 if 判断中为 False 的值，都会使用 default 过滤器提供的默认值。示例代码如下：

```django
{{ value|default:"nothing" }}
```

如果 value 是等于一个空的字符串。比如 `""` ，那么以上代码将会输出 nothing。

## default_if_none

如果值是 None，那么将会使用 default_if_none 提供的默认值。这个和 default 有区别，default 是所有被评估为 False 的都会使用默认值。而 default_if_none 则只有这个值是等于 None 的时候才会使用默认值。示例代码如下：

```django
{{ value|default_if_none:"nothing" }}
```

如果 value 是等于 `""` 也即空字符串，那么以上会输出空字符串。如果 value 是一个 None 值，以上代码才会输出 nothing。

## first

返回列表/元组/字符串中的第一个元素。示例代码如下：

```django
{{ value|first }}
```

如果 value 是等于 `['a', 'b', 'c']` ，那么输出将会是 a。

## last

返回 列表 / 元组 / 字符串 中的最后一个元素。示例代码如下：

```django
{{ value|last }}
```

如果 value 是等于 `['a', 'b', 'c']` ，那么输出将会是 c。

## floatformat

使用四舍五入的方式格式化一个浮点类型。如果这个过滤器没有传递任何参数。那么只会在小数点后保留一个小数，如果小数后面全是 0，那么只会保留整数。当然也可以传递一个参数，标识具体要保留几个小数。

* **没有传递参数**

| `value` | 模板 | 输出 |
| --- | --- | --- |
| `34.23234` | `{{ value|floatformat }}` | `34.2` |
| `34.00000` | `{{ value|floatformat }}` | `34` |
| `34.26000` | `{{ value|floatformat }}` | `34.3` |

* **传递参数**

| `value` | 模板 | 输出 |
| --- | --- | --- |
| `34.23234` | `{{ value|floatformat:3 }}` | `34.232` |
| `34.00000` | `{{ value|floatformat:3 }}` | `34.000` |
| `34.26000` | `{{ value|floatformat:3 }}` | `34.260` |

## join

类似与 Python 中的 join，将列表 / 元组 / 字符串用指定的字符进行拼接。示例代码如下：

```django
{{ value|join:"/" }}
```

如果 value 是等于 `['a', 'b', 'c']` ，那么以上代码将输出 a / b / c。

## length

获取一个 列表 / 元组 / 字符串 / 字典的长度。示例代码如下：

```django
{{ value|length }}
```

如果 value 是等于 `['a', 'b', 'c']` ，那么以上代码将输出 3。如果 value 为 None，那么以上将返回 0。

## lower

将值中所有的字符全部转换成小写。示例代码如下：

```django
{{ value|lower }}
```

如果 value 是等于 Hello World。那么以上代码将输出 hello world。

## upper

类似于 lower，只不过是将指定的字符串全部转换成大写。

## random

在被给的列表/字符串/元组中随机的选择一个值。示例代码如下：

```django
{{ value|random }}
```

如果 value 是等于 `['a', 'b', 'c']` ，那么以上代码会在列表中随机选择一个。

## safe

标记一个字符串是安全的。也即会关掉这个字符串的自动转义。示例代码如下：

```django
{{value|safe}}
```

如果 value 是一个不包含任何特殊字符的字符串，比如 `<a>` 这种，那么以上代码就会把字符串正常的输入。如果 value 是一串 html 代码，那么以上代码将会把这个 html 代码渲染到浏览器中。

## slice

类似于 Python 中的切片操作。示例代码如下：

```django
{{ some_list|slice:"2:" }}
```

以上代码将会给 some_list 从 2 开始做切片操作。

## stringtags

删除字符串中所有的 html 标签。示例代码如下：

```django
{{ value|striptags }}
```

如果 value 是 `<strong>hello world</strong>` ，那么以上代码将会输出 hello world。

## truncatechars

如果给定的字符串长度超过了过滤器指定的长度。那么就会进行切割，并且会拼接三个点来作为省略号。示例代码如下：

```django
{{ value|truncatechars:5 }}
```

如果 value 是等于 `北京欢迎您~` ，那么输出的结果是 `北京...` 。可能你会想，为什么不会 `北京欢迎您...` 呢。因为三个点也占了三个字符，所以北京+三个点的字符长度就是 5。

## truncatechars_html

类似于 truncatechars，只不过是不会切割 html 标签。示例代码如下：

```django
{{ value|truncatechars:5 }}
```

如果 value 是等于 `<p>北京欢迎您~</p>` ，那么输出将是 `<p>北京...</p>` 。

# 6. 自定义过滤器

模版过滤器必须要放在 app 中，并且这个 app 必须要在 `INSTALLED_APPS` 中进行安装。然后再在这个 app 下面创建一个Python 包叫做 `templatetags` 。再在这个包下面创建一个 Python 文件。比如 app 的名字叫做 book，那么目录结构如下：

```bash
- book
    - views.py
    - urls.py
    - models.py
    - templatetags
        - my_filter.py
```

过滤器实际上就是 Python 中的一个函数，只不过是把这个函数注册到模板库中，以后在模板中就可以使用这个函数了：

1. 第一个参数必须是这个过滤器需要处理的值
2. 第二个参数可有可无，如果有，那么就意味着在模板中可以传递参数。并且**过滤器的函数最多只能有两个参数**
3. 使用`django.template.Library`对象注册进去

```python
from django import template

# 创建模板库对象
register = template.Library()

# 过滤器函数
@register.filter(name='mycut') # 将函数注册到模板库中
def mycut(value, mystr):
    return value.replace(mystr)

# 将函数注册到模板库中
# register.filter("mycut", mycut)
```

以后想要在模板中使用这个过滤器，就要在模板中 `load` 一下这个过滤器所在的模块的名字（也就是这个 Python 文件的名字）。示例代码如下：

```django
{% load my_filter %}
```

## 自定义时间计算过滤器

有时候经常会在朋友圈、微博中可以看到一条信息发表的时间，并不是具体的时间，而是距离现在多久。比如刚刚，1 分钟前等。这个功能 DTL 是没有内置这样的过滤器的，因此我们可以自定义一个这样的过滤器。示例代码如下：

```python
# time_filter.py文件

from datetime import datetime
from django import template

register = template.Library()

@register.filter(name='time_since')
def time_since(value):
    """
    time距离现在的时间间隔
    1. 如果时间间隔小于1分钟以内，那么就显示“刚刚”
    2. 如果是大于1分钟小于1小时，那么就显示“xx分钟前”
    3. 如果是大于1小时小于24小时，那么就显示“xx小时前”
    4. 如果是大于24小时小于30天以内，那么就显示“xx天前”
    5. 否则就是显示具体的时间 2017/10/20 16:15
    """
    if isinstance(value, datetime.datetime):
        now = datetime.datetime.now()
        timestamp = (now - value).total_seconds()
        if timestamp < 60:
            return "刚刚"
        elif 60 <= timestamp < 60 * 60:
            minutes = int(timestamp / 60)
            return "%s分钟前" % minutes
        elif 60 * 60 <= timestamp < 60 * 60 * 24:
            hours = int(timestamp / (60 * 60))
            return "%s小时前" % hours
        elif 60 * 60 * 24 <= timestamp < 60 * 60 * 24 * 30:
            days = int(timestamp / (60 * 60 * 24))
            return "%s天前" % days
        else:
            return value.strftime("%Y/%m/%d %H:%M")
    else:
        return value
```

在模版中使用的示例代码如下：

```django
{% load time_filter %}
...
{% value|time_since %}
...
```

# 7. 模板结构

## 引入模板：include

有时候一些代码是在许多模版中都用到的，一般我们可以把这些重复性的代码抽取出来，就类似于 Python 中的函数一样，以后想要使用这些代码的时候，就通过 `include` 包含进来。这个标签就是 `include` 。示例代码如下：

```django
# header.html
<p>我是header</p>

# footer.html
<p>我是footer</p>

# main.html
{% include 'header.html' %}
<p>我是main内容</p>
{% include 'footer.html' %}
```

include 标签寻找路径的方式。也是跟 render 渲染模板的函数是一样的。默认 include 标签包含模版会自动的使用父模版中的上下文，也即可以自动的使用父模版中的变量。如果想传入一些其他的参数，那么可以使用 with 语句。示例代码如下：

```django
# header.html
<p>用户名：{{ username }}</p>

# main.html
{% include "header.html" with username='张三' %}
```

## 模板继承：extends

模版继承类似于 Python 中的类，在父类中可以先定义好一些变量和方法，然后在子类中实现。模版继承也可以在父模版中先定义好一些子模版需要用到的代码，然后子模版直接继承就可以了。并且因为子模版肯定有自己的不同代码，因此可以在父模版中定义一个 `block` 接口，然后子模版再去实现。以下是父模版的代码：

```django
{% load static %}
<!DOCTYPE html>
<html lang="en">
<head>
    <link rel="stylesheet" href="{% static 'style.css' %}" />
    <title>{% block title %}我的站点{% endblock %}</title>
</head>

<body>
    <div id="sidebar">
        {% block sidebar %}
        <ul>
            <li><a href="/">首页</a></li>
            <li><a href="/blog/">博客</a></li>
        </ul>
        {% endblock %}
    </div>
    <div id="content">
        {% block content %}{% endblock %}
    </div>
</body>
</html>
```

这个模版，我们取名叫做 `base.html` ，定义好一个简单的 HTML 骨架，然后定义好两个 `block` 接口，让子模版来根据具体需求来实现。子模板然后通过 `extends` 标签来实现，示例代码如下：

```django
{% extends "base.html" %}

{% block title %}博客列表{% endblock %}

{% block content %}
    {% for entry in blog_entries %}
        <h2>{{ entry.title }}</h2>
        <p>{{ entry.body }}</p>
    {% endfor %}
{% endblock %}
```

`extends` 标签必须放在模版的第一行。 子模板中的代码必须放在 `block` 中，否则将不会被渲染。其次如果在某个 `block` 中需要使用父模版的内容，那么可以使用 `{{block.super}}` 来继承。

```django
{% block title %}
   博客列表
   {{ block.super }}
{% endblock %}
```

在定义 `block` 的时候，除了在 `block` 开始的地方定义这个 `block` 的名字，还可以在 `block` 结束的时候定义名字，这在大型模版中显得尤其有用，能让你快速的看到 block 包含在哪里。

```django
{% block title %}
	...
{% endblock title %}
```

# 8. 加载静态文件

在 DTL 中，使用 `static` 标签来加载静态文件。要使用 `static` 标签，首先需要 `{% load static %}` 。加载静态文件的步骤如下：

1. 首先确保`django.contrib.staticfiles`已经添加到`settings.INSTALLED_APPS` 中。
2. 确保在`settings.py`中设置了`STATIC_URL`。

```python

STATIC_URL = '/static/'
```

3. 如果有一些静态文件是不和任何 app 挂钩的。那么可以在`settings.py`中添加`STATICFILES_DIRS`，以后 DTL 就会在这个列表的路径中查找静态文件。比如可以设置为:

```python
STATICFILES_DIRS = [
     os.path.join(BASE_DIR,"static")
 ]
```

4. 在模版中使用`load`标签加载`static`标签。比如要加载在项目的`static`文件夹下的`style.css`的文件。那么示例代码如下：

```django
{% load static %}
<link rel="stylesheet" href="{% static 'style.css' %}">
```

5. 如果不想每次在模版中加载静态文件都使用 load 加载 static 标签，那么修改 settings.py：

```python
TEMPLATES = [
    {
        # ...
        'OPTIONS': {
            # ...
            'builtins': [
                'django.templatetags.static'
            ]
        },
    },
]
```

6. 如果没有在`settings.INSTALLED_APPS`中添加`django.contrib.staticfiles`。那么我们就需要手动的将请求静态文件的`url`与静态文件的路径进行映射了。示例代码如下：

```python
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # 其他的 url 映射
] + static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)
```

# 9. jinja2 模板引擎配置

> https://docs.djangoproject.com/zh-hans/3.2/topics/templates/#django.template.backends.jinja2.Jinja2

1. 安装`jinja2`

```bash
$ pip install jinja2
```

2. 创建`projectname/jinja2_env.py`文件，并配置`jinja2`环境变量

```python
from django.contrib.staticfiles.storage import staticfiles_storage
from django.urls import reverse

from jinja2 import Environment

def environment(**options):
    env = Environment(**options)
    env.globals.update({
        'static': staticfiles_storage.url,
        'url': reverse,
    })
    return env
```

3. 配置模板引擎，同时注销关于`admin`的配置，因为`admin`使用的是 DTL

```python
# 只是用jinja2

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.jinja2.Jinja2',
        'DIRS': [BASE_DIR / 'templates'],
        'APP_DIRS': True, # 在安装的应用程序的 jinja2 子目录中查找模板
        'OPTIONS': {
            # 配置 jinja2 环境变量
            'environment':'projectname.jinja2_env.environment'
            'context_processors': [],
        },
    },
]
```

4. 在 Jinja2 模板中使用

```jinja
<img src="{{ static('path/to/company-logo.png') }}" alt="Company Logo">

<a href="{{ url('admin:index') }}">Administration</a>
```
