---
title: "Django REST framework"
---

# 1. Django REST framework

> https://www.django-rest-framework.org/

* **安装**

```python
pip install djangorestframework
pip install markdown       # Markdown support for the browsable API.
pip install django-filter  # Filtering support
```

* **添加应用**

```python
INSTALLED_APPS = [
    ...
    'rest_framework',
]
```

* **测试模型**

{{< expand "测试模型" >}}

```python
from django.db import models
from django.contrib.auth.models import User
from django.shortcuts import reverse
from slugify import slugify

class Post(models.Model):
    STATUS_CHOICES = (
        ('draft', '草稿'),
        ('published', '已发布')
    )
    title = models.CharField(max_length=100, verbose_name='标题')
    content = models.TextField(verbose_name='内容')
    status = models.CharField(max_length=10, choices=STATUS_CHOICES, default='draft')
    slug = models.SlugField(max_length=200, blank=True, unique=True, verbose_name='URL 标记')
    author = models.ForeignKey(to=User, on_delete=models.CASCADE, verbose_name='作者')
    create_time = models.DateTimeField(auto_now_add=True)
    update_time = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.title

    def save(self, *args, **kwargs):
        self.slug = slugify(self.slug)
        super().save(*args, **kwargs)

    def get_absolute_url(self):
        return reverse('posts:rest-post-detail', kwargs={'pk': self.id})

class Tag(models.Model):
    name = models.CharField(max_length=100, verbose_name='名称')
    posts = models.ManyToManyField(to=Post)

    def __str__(self):
        return self.name
```

{{< /expand >}}

# 2. 序列化器

> https://www.django-rest-framework.org/api-guide/serializers/

{{< tabs "序列化器" >}}
{{< tab "Serializer" >}}

```python
from django.contrib.auth.models import User
from rest_framework import serializers

class UserSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    username = serializers.CharField(label='用户名', required=True)
    password = serializers.CharField(label='密码', write_only=True)
    email = serializers.CharField(label='邮箱', allow_blank=True, required=False)
```

{{< /tab >}}
{{< tab "ModelSerializer" >}}

```python
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        # 指明参照哪个模型类
        model = Post
        # 指定要包含的字段（可包含反向关系，例如：author 和 tag_set），fields = '__all__' 表示模型中的所有字段。
        fields = ['id', 'title', 'content', 'status', 'author', 'tag_set']
        # 排除显示某些字段
        exclude = ['slug']
        # 指定只读字段，editable=False 和 AutoField 模型字段默认只读，不需要添加
        read_only_fields = ['id']  
        # 默认 ModelSerializer 将关联属性序列化为主键值，设置 depth 表示将序列化外键属性所有字段，整数值表示遍历的关系深度
        depth = 1
        # 额外关键字参数
        extra_kwargs = {
            'author': {'read_only': True},
            'tag_set':{'read_only': True}
        }
```

{{< /tab >}}
{{< tab "HyperlinkedModelSerializer" >}}

```python
class TagSerializer(serializers.HyperlinkedModelSerializer):
    """
    默认情况下，HyperlinkedModelSerializer 将包含一个 url 字段而不是主键字段
    可以使用该 URL_FIELD_NAME 设置全局覆盖此设置
    且在创建序列化器时，需要传递 context 参数：
        - context={'request': request} 则生成的是 绝对 URL
        - context={'request': None} 则生成的是 相对 URL
    """

    # 1. 指定 HyperlinkedIdentityField
    url = serializers.HyperlinkedIdentityField(view_name='posts:rest-tag-detail', lookup_field='pk') 

    class Meta:
        model = Tag
        fields = ['url', 'id', 'name', 'posts']

        # 2. 通过设置 extra_kwargs 替代 HyperlinkedIdentityField
        extra_kwargs = {
            'url': {'view_name': 'posts:rest-tag-detail', 'lookup_field': 'pk'},
        }

>>> serialzier = TagSerializer(...,context={'request': request})
>>> serialzier = TagSerializer(...,context={'request': None})
```

{{< /tab >}}
{{< /tabs >}}

## 序列化器字段

> https://www.django-rest-framework.org/api-guide/fields/

```python
class PostSerializer(serializers.ModelSerializer):
    # 默认情况下，字段可用于序列化和反序列化，设置 read_only=True 只读，表示只用于序列化输出
    id = serializers.IntegerField(read_only=True)

    # 默认情况下，字段可用于序列化和反序列化，required=True，必须，表示反序列化必填
    title = serializers.CharField(required=True)

    # source 表示该字段序列化为 get_absolute_url 方法的返回值
    # allow_black=True，允许反序列化值不填
    # allow_null=True，允许反序列化值为空
    url = serializers.URLField(source='get_absolute_url', allow_blank=True, allow_null=True)

    # style 控制渲染器如何渲染字段
    status = serializers.ChoiceField(choices=['draft', 'published'], style={'base_template': 'radio.html'})

    # initial 填充表单默认值
    create_time = serializers.DateTimeField(initial=timezone.datetime.now)

    # 根据 method_name 指定的序列化方法返回值进行序列化，如果不指定 method_name 默认方法名为 get_<field_name>，该字段为只读字段
    tag_set = serializers.SerializerMethodField(method_name=None)

    def get_tag_set(self, obj):
        """默认为 get_<field_name>, obj 为 Meta.model 指定的模型实例"""
        ......
```

## 序列化关系

{{< tabs "序列化关系" >}}
{{< tab "由一序列化多" >}}

```python
class AuthorField(serializers.RelatedField):
    """自定义关系字段，继承自 RelatedField 实现 to_representation 方法"""

    def to_representation(self, value):
        """模型实例"""
        return f'{value.username}-{value.email}'

class AuthorHyperlink(serializers.HyperlinkedRelatedField):
    view_name = 'posts:user-detail'
    queryset = User.objects.all()

    def get_url(self, obj, view_name, request, format):
        # /users/<pk>/
        url_kwargs = {
            'pk': obj.pk
        }
        return reverse(view_name, kwargs=url_kwargs, request=request, format=format)

    def get_object(self, view_name, view_args, view_kwargs):
        lookup_kwargs = {
            'pk': view_kwargs['pk']
        }
        return self.get_queryset().get(**lookup_kwargs)

class PostSerializer(serializers.ModelSerializer):
    # 1. 默认是将关联模型的 id 序列化
    author = serializers.PrimaryKeyRelatedField(label='作者', read_only=True)
    author = serializers.PrimaryKeyRelatedField(label='作者', queryset=User.objects.all())

    # 2. 通过 source 可以修改显示外键属性值
    author = serializers.CharField(source='author.username')

    # 3. 显示外键属性值为外键模型的 __str__ 方法返回值
    author = serializers.StringRelatedField()

    # 4. 显示外键连接，但是在创建序列化对象时需要传递 context={'request': request}
    # view_name 相当于 <app_name:name>
    author = serializers.HyperlinkedRelatedField(read_only=True, view_name='posts:user-detail')

    # 5. 显示指定外键属性值，但是需要该字段具有 unique=True
    author = serializers.SlugRelatedField(read_only=True, slug_field='username')

    # 6. 超链接字段，但是在创建序列化对象时需要传递 context={'request': request}
    author = serializers.HyperlinkedIdentityField(view_name='posts:user-detail')
    
    # 7. 自定义关系字段
    author = AuthorField(queryset=User.objects.all())
    author = AuthorField(read_only=True)

    # 8. 自定义超链接字段
    author = AuthorHyperlink()

    # 9. 嵌套关系，通过 UserSerializer 显示外键属性值
    author = UserSerializer()
```

{{< /tab >}}
{{< tab "由多序列化一" >}}

```python
class UserSerializer(serializers.ModelSerializer):
    """因为 post_set 是 User 模型上的反向关系，默认不会被包含，所以需要显式的为它添加一个字段。"""

    # 1. 显示外键属性值为外键模型的 __str__ 方法返回值，需要添加 many=True
    post_set = serializers.StringRelatedField(many=True)

    # 2. 默认是将关联模型的 id 序列化
    post_set = serializers.PrimaryKeyRelatedField(many=True, queryset=Post.objects.all())
    post_set = serializers.PrimaryKeyRelatedField(many=True, read_only=True)

    # 3. 显示外键连接，但是在创建序列化对象时需要传递 context={'request': request}
    # view_name 相当于 <app_name:name>
    # lookup_field 相当于 URL 上的命名参数
    post_set = serializers.HyperlinkedRelatedField(view_name='posts:post-detail', lookup_field='pk', many=True, read_only=True)

    # 4. 显示指定外键属性值
    post_set = serializers.SlugRelatedField(read_only=True, slug_field='content', many=True)

    # 5. 超链接字段，但是在创建序列化对象时需要传递 context={'request': request}
    post_set = serializers.HyperlinkedIdentityField(view_name='posts:post-detail', lookup_field='pk', many=True)

    class Meta:
        model = User
        fields = ['post_set']
        # 对于链接字段段，还可以通过额外参数设置
        extra_kwargs = {
            'post_set': {'view_name': 'posts:post-detail', 'lookup_field': 'pk'},
        }
```

{{< /tab >}}
{{< /tabs >}}

{{< hint danger >}}
由一类获取多类或者由多类获取一类进行嵌套序列化，它们的区别就是获取多类数据时需要添加 `many=True` 关键字参数
{{< /hint >}}

## 序列化字段验证

```python
def isnumeric(value):       # 函数验证器
    """纯数字验证"""
    if str(value).isnumeric():
        raise serializers.ValidationError("密码不能是纯数字")
    return value

class IsNumeric(object):    # 类验证器
    def __init__(self):
        """可以接收参数"""
        pass

    def __call__(self, value):
        if str(value).isnumeric():
            raise serializers.ValidationError("密码不能是纯数字")
        return value

class UserSerializer(serializers.ModelSerializer):
    username = serializers.CharField(label='用户名', required=True)
    password = serializers.CharField(label='密码', write_only=True, validators=[isnumeric])
    password2 = serializers.CharField(label='确认密码', write_only=True, validators=[IsNumeric()])

    class Meta:
        extra_kwargs = { # 通过额外参数校验
            'password': {
                'write_only': True,
                'min_length': 8,
                'max_length': 20,
                'error_messages': {
                    'min_length': '仅允许8-20个字符的密码',
                    'max_length': '仅允许8-20个字符的密码',
                }
            }
        }

    def validate_username(self, value): # 单个字段验证
        """
        通过 validate_<field_name> 方法验证
        验证通过返回 value，否则抛出 serializers.ValidationError 异常
        """
        user = User.objects.filter(username=value).first()
        if user:
            raise serializers.ValidationError("用户名已存在")
        return value 
    
    def validate(self, attrs):  # 多个字段之间验证
        """通过 validate 方法验证"""
        if attrs['password'] != attrs['password2']:
            raise serializers.ValidationError('两次密码不一致')
        return attrs

```

## 序列化和反序列化过程

{{< tabs "序列化和反序列化过程" >}}
{{< tab "序列化" >}}

```python
# 1.  序列化单个对象
user = User.objects.get(pk=pk)
serializer = UserSerializer(instance=user)

# 2. 序列化多个对象，并传递额外参数
users = User.objects.all()
serializer = UserSerializer(users, many=True, context={"request": request})

# 3. 获取序列化后的 json 数据
serializer.data 
```

{{< /tab >}}
{{< tab "反序列化" >}}

```python
# ------------------------ 新建对象 ------------------------

# 传递请求数据给 data
serializer = UserSerializer(data=request.data)

# 数据校验，raise_exception 表示检验失败，返回 400 异常
if serializer.is_valid(raise_exception=True):
    serializer.save()           # 调用 UserSerializer 的 create() 方法，创建一个新对象保存
    serializer.data             # 获取序列化后的 json 数据
    serializer.validated_data   # 验证通过的字段
else:
    serializer.errors # 验证结果信息
    
# ------------------------ 更新对象 ------------------------
# 默认情况下，序列化程序必须为所有必填字段传递值，否则会引发验证错误，使用该 partial 参数来允许部分更新
serializer2 = UserSerializer(instance=user, data=request.data, partial=True)
# 调用 UserSerializer.update() 方法，部分更新一个对象并保存
serializer2.save() 

# ------------------------ 给 create() 和 update() 传递额外参数 ------------------------
serializer.save(owner=request.user)
```

{{< /tab >}}
{{< tab "序列化方法" >}}

```python
class UserSerializer(serializers.Serializer):
    ......

    def create(self, validated_data):
        """ 创建用户 """
        return User.objects.create_user(**validated_data)

    def update(self, instance, validated_data):
        """ 更新用户 """
        # owner = validated_data['owner'] 获取 save 方法传递过来的额外参数
        instance.username = validated_data.get('username', instance.username)
        instance.email = validated_data.get('email', instance.email)
        instance.set_password(validated_data.get('password', instance.password))
        instance.save()

        return instance

    def save(self, **kwargs):
        """
        覆盖 save() 方法，在使用 serializer.save() 时不调用 create 或 update 方法一般用于不保存对象，而是做其他的事
        """
        pass
```

{{< /tab >}}
{{< /tabs >}}

# 3. 视图和 Mixin

{{< tabs "视图" >}}
{{< tab "函数视图" >}}

```python
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from .models import User
from .serializers import UserSerializer

@api_view(['GET', 'POST'])
def user_list_view(request):
    """
    显示所有用户，或创建新用户
    """
    if request.method == 'GET':
        users = User.objects.all()
        serializer = UserSerializer(users, many=True, context={"request": request})
        return Response(serializer.data)

    elif request.method == 'POST':
        serializer = UserSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

@api_view(['GET', 'PUT', 'DELETE'])
def user_detail_view(request, pk):
    """
    获取、更新、删除一个用户
    """
    try:
        user = User.objects.get(pk=pk)
    except User.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
        serializer = UserSerializer(user)
        return Response(serializer.data)

    elif request.method == 'PUT':
        serializer = UserSerializer(user, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'DELETE':
        user.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)

urlpatterns = [
    path('users/', views.user_list_view, name='user-list'),
    path('users/<int:pk>/', views.user_detail_view, name='user-detail'),
]
```

{{< /tab >}}
{{< tab "类视图" >}}

```python
from django.http import Http404
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

from .serializers import PostSerializer
from .models import Post

class PostListView(APIView):
    """
    获取所有文章、或创建新文章
    """

    def get(self, request, format=None):
        posts = Post.objects.all()
        serializer = PostSerializer(posts, many=True, context={'request': request})
        return Response(serializer.data)

    def post(self, request, format=None):
        serializer = PostSerializer(data=request.data)
        if serializer.is_valid():
            # 在保存文章的时候传递额外参数给 create 方法
            serializer.save(author=self.request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

class PostDetailView(APIView):
    """
    获取、更新、删除一篇文章
    """

    def get_object(self, pk):
        try:
            return Post.objects.get(pk=pk)
        except Post.DoesNotExist:
            raise Http404

    def get(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = PostSerializer(snippet, context={'request': request})
        return Response(serializer.data)

    def put(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = PostSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk, format=None):
        snippet = self.get_object(pk)
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)

urlpatterns = [
    path('posts/', views.PostListView.as_view(), name='post-list'),
    path('posts/<int:pk>/', views.PostDetailView.as_view(), name='post-detail'),
]
```

{{< /tab >}}
{{< tab "通用类视图" >}}

```python
from rest_framework import mixins
from rest_framework import generics

from .models import Tag
from .serializers import TagSerializer

class TagListView(mixins.ListModelMixin,
                mixins.CreateModelMixin,
                generics.GenericAPIView):
    queryset = Tag.objects.all()
    serializer_class = TagSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)

class TagDetailView(mixins.RetrieveModelMixin,
                    mixins.UpdateModelMixin,
                    mixins.DestroyModelMixin,
                    generics.GenericAPIView):
    queryset = Tag.objects.all()
    serializer_class = TagSerializer

    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)

urlpatterns = [
    path('tags/', views.TagListView.as_view(), name='tag-list'),
    path('tags/<int:pk>/', views.TagDetailView.as_view(), name='tag-detail'),
]
```

{{< /tab >}}
{{< /tabs >}}

## 内置 Mixin

{{< tabs "Mixin" >}}
{{< tab "ListModelMixin" >}}

列表视图扩展类，提供 `list(request, *args, **kwargs)` 方法快速实现列表视图，返回 200 状态码。该 `Mixin` 的 `list` 方法会对数据进行过滤和分页。

```python
class ListModelMixin(object):
    """
    List a queryset.
    """
    def list(self, request, *args, **kwargs):
        # 过滤
        queryset = self.filter_queryset(self.get_queryset())
        # 分页
        page = self.paginate_queryset(queryset)
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)
        # 序列化
        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)

# Example：
from rest_framework.mixins import ListModelMixin

class UserListView(ListModelMixin, GenericAPIView):
    queryset = User.objects.all()
    serializer_class = UserInfoModelSerializer

    def get(self, request):
        return self.list(request)
```

{{< /tab >}}
{{< tab "CreateModelMixin" >}}

创建视图扩展类，提供 `create(request, *args, **kwargs)` 方法快速实现创建资源的视图，成功返回 201 状态码。如果序列化器对前端发送的数据验证失败，返回 400 错误。

```python
class CreateModelMixin(object):
    """
    Create a model instance.
    """
    def create(self, request, *args, **kwargs):
        # 获取序列化器
        serializer = self.get_serializer(data=request.data)
        # 验证
        serializer.is_valid(raise_exception=True)
        # 保存
        self.perform_create(serializer)
        headers = self.get_success_headers(serializer.data)
        return Response(serializer.data, status=status.HTTP_201_CREATED, headers=headers)

    def perform_create(self, serializer):
        serializer.save()

    def get_success_headers(self, data):
        try:
            return {'Location': str(data[api_settings.URL_FIELD_NAME])}
        except (TypeError, KeyError):
            return {}

# Example：
from rest_framework.mixins import CreateModelMixin

class UserListView(CreateModelMixin, GenericAPIView):
    serializer_class = UserInfoModelSerializer

    def post(self, request):
        return self.create(request)
```

{{< /tab >}}
{{< tab "RetrieveModelMixin" >}}

详情视图扩展类，提供 `retrieve(request, *args, **kwargs)` 方法，可以快速实现返回一个存在的数据对象。如果存在，返回200， 否则返回 404。

```python
class RetrieveModelMixin(object):
    """
    Retrieve a model instance.
    """
    def retrieve(self, request, *args, **kwargs):
        # 获取对象，会检查对象的权限
        instance = self.get_object()
        # 序列化
        serializer = self.get_serializer(instance)
        return Response(serializer.data)

# Example：
class UserDetailView(RetrieveModelMixin, GenericAPIView):
    queryset = User.objects.all()
    serializer_class = UserInfoModelSerializer

    def get(self, request, pk):
        return self.retrieve(request)
```

{{< /tab >}}
{{< tab "UpdateModelMixin" >}}

更新视图扩展类，提供 `update(request, *args, **kwargs)` 方法，可以快速实现更新一个存在的数据对象。同时也提供 `partial_update(request, *args, **kwargs)` 方法，可以实现局部更新。成功返回 200，序列化器校验数据失败时，返回 400 错误。

```python
class UpdateModelMixin(object):
    """
    Update a model instance.
    """
    def update(self, request, *args, **kwargs):
        partial = kwargs.pop('partial', False)
        instance = self.get_object()
        serializer = self.get_serializer(instance, data=request.data, partial=partial)
        serializer.is_valid(raise_exception=True)
        self.perform_update(serializer)

        if getattr(instance, '_prefetched_objects_cache', None):
            # If 'prefetch_related' has been applied to a queryset, we need to
            # forcibly invalidate the prefetch cache on the instance.
            instance._prefetched_objects_cache = {}

        return Response(serializer.data)

    def perform_update(self, serializer):
        serializer.save()

    def partial_update(self, request, *args, **kwargs):
        kwargs['partial'] = True
        return self.update(request, *args, **kwargs)

# Example：
from rest_framework.mixins import UpdateModelMixin

class UserListView(UpdateModelMixin, GenericAPIView):
    serializer_class = UserInfoModelSerializer

    def put(self, request, pk):
        return self.update(request)
        # return self.partial_update(request)
```

{{< /tab >}}
{{< tab "DestroyModelMixin" >}}

删除视图扩展类，提供 `destroy(request, *args, **kwargs)` 方法，可以快速实现删除一个存在的数据对象。成功返回 204，不存在返回 404。

```python
class DestroyModelMixin(object):
    """
    Destroy a model instance.
    """
    def destroy(self, request, *args, **kwargs):
        instance = self.get_object()
        self.perform_destroy(instance)
        return Response(status=status.HTTP_204_NO_CONTENT)

    def perform_destroy(self, instance):
        instance.delete()

# Example：
from rest_framework.mixins import DestroyModelMixin

class UserListView(DestroyModelMixin, GenericAPIView):
    serializer_class = UserInfoModelSerializer

    def delete(self, request, pk):
        return self.destroy(request)
```

{{< /tab >}}
{{< /tabs >}}

## 具体视图类

![具体视图类](具体视图类.png)

## 自定义 Mixin

例如，如果需要根据 URL conf 中的多个字段查找对象，则可以创建一个 `mixin` 类

```python
class MultipleFieldLookupMixin:
    """
    Apply this mixin to any view or viewset to get multiple field filtering
    based on a `lookup_fields` attribute, instead of the default single field filtering.
    """
    def get_object(self):
        queryset = self.get_queryset()             # Get the base queryset
        queryset = self.filter_queryset(queryset)  # Apply any filter backends
        filter = {}
        for field in self.lookup_fields:
            if self.kwargs[field]: # Ignore empty fields.
                filter[field] = self.kwargs[field]
        obj = get_object_or_404(queryset, **filter)  # Lookup the object
        self.check_object_permissions(self.request, obj)
        return obj
```

随后可以在需要应用自定义行为的任​​何时候，将该 `mixin` 应用于视图或视图集。

```python
class RetrieveUserView(MultipleFieldLookupMixin, generics.RetrieveAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    lookup_fields = ['pk', 'username']

urlpatterns = [
    path('users/<pk>/<username>/', views.RetrieveUserView.as_view(), name='user-detail'),
]
```

# 4. 视图集

视图集 `ViewSet` 可以将一系列逻辑相关的动作放到一个类中，该类不再实现 `get()` 、 `post()` 等方法，而是实现动作 `action` ，如下：

* `list()`：提供一组数据
* `retrieve()`：提供单个数据
* `create()`：创建数据
* `update()`：保存数据
* `destory()`：删除数据

{{< tabs "视图集" >}}
{{< tab "ViewSet" >}}

```python
from rest_framework import viewsets

class UserViewSet(viewsets.ViewSet):

    def list(self, request):
        queryset = User.objects.all()
        serializer = UserSerializer(queryset, many=True)
        return Response(serializer.data)

    def retrieve(self, request, pk=None):
        queryset = User.objects.all()
        user = get_object_or_404(queryset, pk=pk)
        serializer = UserSerializer(user)
        return Response(serializer.data)

    def create(self, request, *args, **kwargs):
        pass

    def update(self, request, *args, **kwargs):
        pass

    def partial_update(self, request, *args, **kwargs):
        pass

    def destroy(self, request, *args, **kwargs):
        pass

urlpatterns = [
    path('users/', views.UserViewSet.as_view({'get': 'list', 'post': 'create'}), name='user-list'),
    path('users/<int:pk>/', views.UserViewSet.as_view({'get': 'retrieve', 'delete': 'destroy', 'put': 'update'}), name='user-detail'),
]
```

{{< /tab >}}
{{< tab "GenericViewSet" >}}

```python
from rest_framework.viewsets import GenericViewSet
from rest_framework import mixins

class UserGenericViewSet(mixins.ListModelMixin, mixins.RetrieveModelMixin,
                         mixins.UpdateModelMixin, mixins.DestroyModelMixin,
                         mixins.CreateModelMixin,
                         GenericViewSet):
    """GenericViewSet 结合 mixins"""
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

{{< /tab >}}
{{< tab "ModelViewSet" >}}

```python
from rest_framework import viewsets

class UserModelViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

{{< /tab >}}
{{< tab "ReadOnlyModelViewSet" >}}

`ReadOnlyModelViewSet` 与 `ModelViewSet` 一样，但不同的是它只提供 “只读” 操作： `list() 和 ` retrieve()`。

```python
from rest_framework import viewsets

class UserReadOnlyModelViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

{{< /tab >}}
{{< /tabs >}}

## 视图集中定义附加 action 动作

```python
from django.contrib.auth.models import User
from rest_framework import status, viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from .serializers import UserSerializer

class UserModelViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer

    # detail 为 True，表示路径名格式应该为 users/{pk}/set_password/
    @action(detail=True, methods=['post'])
    def set_password(self, request, pk=None):
        ......

    # detail为 False 表示路径名格式应该为 users/recent_users/
    @action(detail=False)
    def recent_users(self, request):
        ......

    # 装饰器允许覆盖任何视图集级别的配置，例如 permission_classes, serializer_class, filter_backends
    @action(detail=True, methods=['post'], permission_classes=[IsAdminOrIsSelf])
    def unset_password(self, request, pk=None):
        ......

    # 相当于给 set_password action 添加一个 HTTP delete 操作
    @set_password.mapping.delete
    def delete_password(self, request, pk=None):
        """Delete the user's password."""
        ......
```

## 视图集中获取反向解析的 URL

```python
class UserModelViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer

    @action(detail=True, methods=['post'])
    def set_password(self, request, pk=None):
        ......

    @action(detail=True)
    def set_password_url(self, request, pk=None):
        # 反向解析，自动添加路由器配置的 basename 路由名称的前缀
        return Response({"set_password_url": self.reverse_action('set_password', args=pk)})
```

# 5. 路由器 Routers

> https://www.django-rest-framework.org/api-guide/routers/#routers

对于视图集 `ViewSet` ，我们除了可以自己手动指明请求方式与动作 `action` 之间的对应关系外，还可以使用 `Routers` 来帮助我们快速实现路由信息。 `REST framework` 提供了 `SimpleRouter` 和 `DefaultRouter` 。其中 `DefaultRouter` 的区别是会多附带一个默认的 API 根视图，返回一个包含所有列表视图的超链接响应数据，下面的演示以 `SimpleRouter` 为例：

```python
from rest_framework import routers

router = routers.SimpleRouter()
router.register(prefix='users', viewset=views.UserModelViewSet, basename='user')

# 最终会形成：
# path('users/', views.UserModelViewSet, name='user-list')
# path('users/<pk>/', views.UserModelViewSet, name='user-detail')

# 添加路由数据
urlpatterns = [
    ...
]

urlpatterns += router.urls

# 或者

urlpatterns = [
    ...
    path('', include(router.urls)),
    path('', include((router.urls, 'app_name'), namespace='instance_name')),
]

```

* `prefix`：该视图集的路由前缀
* `viewset`：视图集
* `basename`：路由名称的前缀

默认情况下，创建的 `SimpleRouter` 会在视图集的路由前缀附加一个尾斜杠。可以通过在实例化路由器时将 `trailing_slash` 参数设置为 False 来修改此行为。例如：

```python
router = SimpleRouter(trailing_slash=False)
```

# 6. 解析器

> https://www.django-rest-framework.org/api-guide/parsers/

* `JSONParser`：解析 JSON 请求数据，`request.data` 得到一个数据字典
* `FormParser`：解析多 HTML Form 表单请求数据，`request.data` 得到一个 `QueryDict`
* `MultiPartParser`：解析多 HTML Form 表单内容，支持文件上传，`request.data` 得到一个 `QueryDict`
* `FileUploadParser`：解析原始文件上传内容。`request.data` 属性将是一个字典，其中包含一个包含上传文件的键 'file'

{{< tabs "解析器" >}}
{{< tab "全局配置" >}}

```python
'DEFAULT_PARSER_CLASSES': [
        # 允许带有 JSON 内容的请求，而不是默认的 JSON 或表单数据。
        'rest_framework.parsers.JSONParser',

        # pip install djangorestframework-yaml
        'rest_framework_yaml.parsers.YAMLParser',

        # pip install djangorestframework-xml
        'rest_framework_xml.parsers.XMLParser',
    ],
```

{{< /tab >}}
{{< tab "类视图属性配置" >}}

```python
from rest_framework.parsers import JSONParser
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
    """接受带有 JSON 内容的 POST 请求的视图"""
    parser_classes = [JSONParser]
    ......
```

{{< /tab >}}
{{< tab "函数视图装饰器配置" >}}

```python
from rest_framework.decorators import api_view
from rest_framework.decorators import parser_classes
from rest_framework.parsers import JSONParser

@api_view(['POST'])
@parser_classes([JSONParser])
def example_view(request, format=None):
    """接受带有 JSON 内容的 POST 请求的视图"""
    ......
```

{{< /tab >}}
{{< /tabs >}}

# 7. 渲染器

> https://www.django-rest-framework.org/api-guide/renderers/

{{< tabs "渲染器" >}}
{{< tab "全局配置" >}}

```python
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',
        'rest_framework_yaml.renderers.YAMLRenderer',
        'rest_framework_xml.renderers.XMLRenderer',
    ]
}
```

{{< /tab >}}
{{< tab "类视图属性配置" >}}

```python
from rest_framework.renderers import JSONRenderer
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
    """以 JSON 形式返回数据"""
    renderer_classes = [JSONRenderer]
    ......
```

{{< /tab >}}
{{< tab "函数视图装饰器配置" >}}

```python
@api_view(['GET'])
@renderer_classes([JSONRenderer])
def example_view(request, format=None):
    """以 JSON 形式返回数据"""
    ......
```

* **模板渲染**

```python
class UserDetailTemplateHTMLRenderer(generics.RetrieveAPIView):
    """返回模板"""

    queryset = User.objects.all()
    renderer_classes = [TemplateHTMLRenderer]
    serializer_class = UserSerializer

    def get(self, request, *args, **kwargs):
        self.object = self.get_object()
        return Response({'user': self.object}, template_name='user_detail.html')
```

* **静态 HTML 渲染**

```python
@api_view(['GET'])
@renderer_classes([StaticHTMLRenderer])
def simple_html_view(request):
    data = '<html><body><h1>Hello, world</h1></body></html>'
    return Response(data)
```

* **不同类型选择不同的渲染器**

```python
@api_view(['GET'])
@renderer_classes([TemplateHTMLRenderer, JSONRenderer])
def list_users(request):
    """返回 JSON 或者 Template"""

    queryset = Users.objects.filter(active=True)
    if request.accepted_renderer.format == 'html':
        data = {'users': queryset}
        return Response(data, template_name='list_users.html')
    else:
        serializer = UserSerializer(instance=queryset)
        data = serializer.data
        return Response(data)
```

{{< /tab >}}
{{< /tabs >}}

# 8. 验证

## 验证配置

{{< tabs "验证配置" >}}
{{< tab "全局配置" >}}

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework.authentication.BasicAuthentication',   # 基本认证
        'rest_framework.authentication.SessionAuthentication',  # session认证
    )
}
```

{{< /tab >}}
{{< tab "类视图属性配置" >}}

```python
from rest_framework.authentication import SessionAuthentication, BasicAuthentication
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
    authentication_classes = [SessionAuthentication, BasicAuthentication]
    permission_classes = [IsAuthenticated]

    def get(self, request, format=None):
        content = {
            'user': str(request.user),  # `django.contrib.auth.User` instance.
            'auth': str(request.auth),  # None
        }
        return Response(content)
```

{{< /tab >}}
{{< tab "函数视图装饰器配置" >}}

```python
from rest_framework.authentication import SessionAuthentication, BasicAuthentication
from rest_framework.decorators import api_view, authentication_classes, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response

@api_view(['GET'])
@authentication_classes([SessionAuthentication, BasicAuthentication])
@permission_classes([IsAuthenticated])
def example_view(request, format=None):
    content = {
        'user': str(request.user),  # `django.contrib.auth.User` instance.
        'auth': str(request.auth),  # None
    }
    return Response(content)
```

{{< /tab >}}
{{< /tabs >}}

## 内置认证方案

{{< tabs "认证方案" >}}
{{< tab "BasicAuthentication" >}}

如果请求头中包含用户名和密码的签名( `Authorization: Basic xxx` )，然后通过 `base64.b64decode("xxx").decode('utf8')` 解析签名得到用户名和密码，以此来认证用户名和密码的正确性，从而决定用户是否认证通过。

* 如果身份验证通过，则 `request.user` 为用户实例，`request.auth` 为 None
* 如果身份验证不通过，返回 `401 Unauthorized` 响应且带有 `WWW-Authenticate: Basic realm="api"` 的响应头

如果请求头中没有包含签名，则 `request.user` 为 AnonymousUser 实例， `request.auth` 为 None

{{< /tab >}}
{{< tab "TokenAuthentication" >}}

使用该认证方案需要在 `INSTALLED_APPS` 注册 `'rest_framework.authtoken'` 应用并且进行数据库迁移，完成之后，会创建一张与用户一对一关系的 Token 表。如果请求中包含用户 Token（ `Authorization: Token 401f7ac837da42b97f613d789819ff93537bee6a` ），然后查询数据库进行比对，以此来决定用户是否认证通过。

* 如果成功通过身份验证，则`request.user` 为用户实例，`request.auth` 是一个 `rest_framework.authtoken.models.Token` 实例
* 如果身份验证不通过，返回 `401 Unauthorized` 响应且带有 `WWW-Authenticate: Token` 的响应头

如果请求头中没有包含签名，则 `request.user` 为 AnonymousUser 实例， `request.auth` 为 None

```python
# 方式一：根据用户对象生成 Token
from rest_framework.authtoken.models import Token
token = Token.objects.create(user=...)

# 方式二：通过信号，在创建用户时，自动生成 Token
from django.conf import settings
from django.db.models.signals import post_save
from django.dispatch import receiver
from rest_framework.authtoken.models import Token

@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def create_auth_token(sender, instance=None, created=False, **kwargs):
    if created:
        Token.objects.create(user=instance)

# 方式三：为所有现有用户生成 Token
from django.contrib.auth.models import User
from rest_framework.authtoken.models import Token

for user in User.objects.all():
    Token.objects.get_or_create(user=user)

# 方式四：管理员后台选择用户生成，配置 your_app/admin.py
from rest_framework.authtoken.admin import TokenAdmin
TokenAdmin.raw_id_fields = ['user']

# 方式五：使用命令生成用户 Token
python manage.py drf_create_token <username>    # 创建 Token
python manage.py drf_create_token -r <username> # 重新生成 Token
```

{{< /tab >}}
{{< tab "SessionAuthentication" >}}

`SessionAuthentication` 使用 Django 的默认会话进行身份验证

* 如果请求会话当前有一个登录用户，则`request.user` 为用户实例，`request.auth` 为 None
* 如果请求会话没有用户登录，则 `request.user` 为 AnonymousUser 实例， `request.auth` 为 None

{{< /tab >}}
{{< /tabs >}}

## 自定义身份验证

要实现自定义身份验证方案，需要继承 `BaseAuthentication` 并重写 `authenticate()` 方法，该方法应该返回一个二元组，如果身份验证成功返回 `(user, auth)` ，这两个值会添加到 `request` 对象上，否则返回 None。

如果需要在验证失败的时候，抛出 `AuthenticationFailed` 异常，返回 `HTTP 401 Unauthorized` 相应时，还可以重写 `authenticate_header()` 方法，该方法返回一个字符串，作为响应头 `WWW-Authenticate` 的值。

```python
from django.contrib.auth.models import User
from rest_framework import authentication
from rest_framework import exceptions

class ExampleAuthentication(authentication.BaseAuthentication):
    """根据用户请求头是否携带 X-USERNAME:username 从而认证用户"""

    def authenticate(self, request):
        username = request.META.get('HTTP_X_USERNAME')
        if not username:
            return None

        try:
            user = User.objects.get(username=username)
        except User.DoesNotExist:
            raise exceptions.AuthenticationFailed('No such user')

        return (user, None)
```

# 9. 权限

## 权限配置

{{< tabs "权限配置" >}}
{{< tab "全局配置" >}}

```python
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.AllowAny', # 默认允许无限制访问
        'rest_framework.permissions.IsAuthenticated',
    ]
}
```

{{< /tab >}}
{{< tab "类视图属性配置" >}}

```python
from rest_framework.permissions import BasePermission, IsAuthenticated, SAFE_METHODS
from rest_framework.response import Response
from rest_framework.views import APIView

class ReadOnly(BasePermission):
    def has_permission(self, request, view):
        return request.method in SAFE_METHODS

class ExampleView(APIView):
    permission_classes = [IsAuthenticated]
    # permission_classes = [IsAuthenticated|ReadOnly] 按位运算符组合权限

    def get(self, request, format=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)
```

{{< /tab >}}
{{< tab "函数视图装饰器配置" >}}

```python
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response

@api_view(['GET'])
@permission_classes([IsAuthenticated])
def example_view(request, format=None):
    content = {
        'status': 'request was permitted'
    }
    return Response(content)
```

{{< /tab >}}
{{< /tabs >}}

## 内置权限方案

{{< tabs "内置权限类" >}}
{{< tab "AllowAny" >}}

```python
class AllowAny(BasePermission):
    """直接返回 True，允许任何访问"""

    def has_permission(self, request, view):
        return True
```

{{< /tab >}}
{{< tab "IsAuthenticated" >}}

```python
class IsAuthenticated(BasePermission):
    """仅允许经过身份验证的用户访问"""
    def has_permission(self, request, view):
        return bool(request.user and request.user.is_authenticated)
```

{{< /tab >}}
{{< tab "IsAdminUser" >}}

```python

class IsAdminUser(BasePermission):
    """仅允许管理员用户访问"""
    def has_permission(self, request, view):
        return bool(request.user and request.user.is_staff)
```

{{< /tab >}}
{{< tab "IsAuthenticatedOrReadOnly" >}}

```python
class IsAuthenticatedOrReadOnly(BasePermission):
    """登录的用户可以随意访问，其他用户只能通过 ('GET', 'HEAD', 'OPTIONS') 方法访问"""

    def has_permission(self, request, view):
        return bool(
            request.method in SAFE_METHODS or
            request.user and
            request.user.is_authenticated
        )
```

{{< /tab >}}
{{< /tabs >}}

## 自定义权限

要实现自定义权限，需要继承 `BasePermission` 并实现下面两个方法中的任意一个：

* `.has_permission(self, request, view)`
* `.has_object_permission(self, request, view, obj)`

如果授予访问权限，则返回 True，否则返回 False

```python
from rest_framework import permissions

class BlocklistPermission(permissions.BasePermission):
    """只允许 IP 地址没有在黑名单的访问"""

    def has_permission(self, request, view):
        ip_addr = request.META['REMOTE_ADDR']
        blocked = Blocklist.objects.filter(ip_addr=ip_addr).exists()
        return not blocked

class IsOwnerOrReadOnly(permissions.BasePermission):
    """只允许对象的所有者进行任何操作，其他用户只能 通过 ('GET', 'HEAD', 'OPTIONS') 方法访问"""

    def has_object_permission(self, request, view, obj):
        # obj 为当前模型实例
        if request.method in permissions.SAFE_METHODS:
            return True

        # 假设该模型有一个关联用户的 owner 字段
        return obj.owner == request.user
```

# 10. 限流

## 限流配置

{{< tabs "限流配置" >}}
{{< tab "全局配置" >}}

```python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day', # second，minute，hour,day
        'user': '1000/day'
    }
}
```

{{< /tab >}}
{{< tab "类视图属性配置" >}}

```python
from rest_framework.response import Response
from rest_framework.throttling import UserRateThrottle
from rest_framework.views import APIView

class ExampleView(APIView):
    throttle_classes = [UserRateThrottle]

    def get(self, request, format=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)
```

{{< /tab >}}
{{< tab "函数视图装饰器配置" >}}

```python
@api_view(['GET'])
@throttle_classes([UserRateThrottle])
def example_view(request, format=None):
    content = {
        'status': 'request was permitted'
    }
    return Response(content)

# 使用 @action 装饰器创建的路由设置
@action(detail=True, methods=["post"], throttle_classes=[UserRateThrottle])
def example_adhoc_method(request, pk=None):
    content = {
        'status': 'request was permitted'
    }
    return Response(content)
```

{{< /tab >}}
{{< /tabs >}}

## 内置限流策略

限流策略需要用到缓存，用来存储用户请求的唯一标识，例如：对于匿名用户来说可以缓存 IP 地址，登陆的用户可以缓存用户名或 ID，而 REST framework 默认使用 Django 设置的的缓存后端 `detault` ，不过可以通过创建自定义限制类并设置 `cache` 属性来实现

```python
class CustomAnonRateThrottle(AnonRateThrottle):
    cache = caches['throttle']
```

{{< tabs "内置限流策略" >}}
{{< tab "AnonRateThrottle" >}}

```python
class AnonRateThrottle(SimpleRateThrottle):
    """缓存匿名用户的 IP，通过获取 IP 次数从而判断用户请求的次数，以此达到限流"""

    scope = 'anon' # 配置 DEFAULT_THROTTLE_RATES 时的键名

    def get_cache_key(self, request, view):
        if request.user.is_authenticated:
            return None  # Only throttle unauthenticated requests.

        return self.cache_format % {
            'scope': self.scope,
            'ident': self.get_ident(request)
        }
```

{{< /tab >}}
{{< tab "UserRateThrottle" >}}

```python
class UserRateThrottle(SimpleRateThrottle):
    """
    如果是匿名用户，缓存匿名用户的 IP，通过获取 IP 次数从而判断用户请求的次数，以此达到限流
    如果是登录用户，缓存匿名用户的 ID，通过获取 ID 次数从而判断用户请求的次数，以此达到限流
    """
    scope = 'user'

    def get_cache_key(self, request, view):
        if request.user.is_authenticated:
            ident = request.user.pk
        else:
            ident = self.get_ident(request)

        return self.cache_format % {
            'scope': self.scope,
            'ident': ident
        }
```

{{< /tab >}}
{{< tab "ScopedRateThrottle" >}}

`ScopedRateThrottle` 类可用于限制访问 API 的特定部分，也就限制访问某个视图的次数

```python
class ContactListView(APIView):
    throttle_scope = 'contacts'
    ...

class ContactDetailView(APIView):
    throttle_scope = 'contacts'
    ...

class UploadView(APIView):
    throttle_scope = 'uploads'
    ...

REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.ScopedRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'contacts': '1000/day', # 限制用户对 ContactListView 或 ContactDetailView 每天总共 1000 个请求
        'uploads': '20/day' # 限制用户对 UploadView 每天 20 个请求
    }
}
```

{{< /tab >}}
{{< /tabs >}}

## 自定义限流

要创建自定义限流类，需要继承 `BaseThrottle` 并重写 `allow_request` 方法，如果请求被允许，该方法应该返回 True，否则返回 False。

```python
import random

class RandomRateThrottle(throttling.BaseThrottle):
    """将在每 10 个请求中随机限制 1 个"""

    def allow_request(self, request, view):
        return random.randint(1, 10) != 1

    def wait(self):
        """当 allow_request 返回 False 时，该方法返回下一个请求等待的秒数"""
        return 10
```

# 11. 过滤

`django-filter` 库包括一个支持 REST 框架高度可定制的字段过滤的类 `DjangoFilterBackend` ，要使用它需要先安装 `django-filter` 。

* **下载**

```bash
$ pip install django-filter
```

* **安装应用**

```python
INSTALLED_APPS = [
    ...
    'django_filters',
    ...
]
```

* **配置过滤后端**

```python
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': ['django_filters.rest_framework.DjangoFilterBackend']
}
```

* **在类视图中配置过滤**

```python
from django_filters.rest_framework import DjangoFilterBackend

class PostListView(generics.ListAPIView):
    ...
    filter_backends = [DjangoFilterBackend]
    filterset_fields = ['title', 'status'] # 希望过滤的字段集

    # ---------------- 搜索过滤器 ------------------------

    filter_backends = [filters.SearchFilter]
    search_fields = ['title', 'content', 'author__username'] # 搜索，双下划线符号对 ForeignKey 或 ManyToManyField 执行相关查找

    search_fields = ['^username', '=email','@contnet'] # $' 正则表达式搜索 '^' 以搜索开头 '=' 完全匹配 '@' 全文搜索(只支持 Django 的 PostgreSQL 后端)

    # ---------------- 排序过滤器 ------------------------

    filter_backends = [filters.OrderingFilter]
    ordering_fields = ['username', 'email']
    ordering_fields = '__all__'
    ordering = ['username'] # 指定默认排序
```

* **自定义通用过滤器**

```python
class IsOwnerFilterBackend(filters.BaseFilterBackend):
    """仅允许用户查看自己对象的过滤器"""

    def filter_queryset(self, request, queryset, view):
        return queryset.filter(owner=request.user)
```

# 12. 分页
* **配置**

```python
REST_FRAMEWORK = {
    # 使用内置的偏移量分页：https://api.example.org/accounts/?limit=100&offset=400
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.LimitOffsetPagination',

    # 使用页码分页："https://api.example.org/accounts/?page=5",
    # 'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',

    'PAGE_SIZE': 100
}
```

* **页码分页配置**

```python
class StandardResultsSetPagination(PageNumberPagination):
    # 指示页面大小的数值，如果设置了，将覆盖 PAGE_SIZE
    page_size = 100 

    # 分页查询参数的名称
    page_query_param = 'page'

    # 允许客户端基于每个请求，设置页面大小的查询参数的名称
    page_size_query_param = 'page_size'

    # 最大允许请求的页面大小，此属性仅在 page_size_query_param 也设置时有效
    max_page_size = 1000

    # 指示可与 page_query_param 一起使用，以请求集合中的最后一页的值
    last_page_strings = ('last',)

    # 呈现分页控件时使用的模板名称，可以覆盖以修改呈现样式
    template = "rest_framework/pagination/numbers.html"

REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'apps.core.pagination.StandardResultsSetPagination'
}
```

* **自定义分页样式**

要创建自定义分页序列化程序类，需要继承 `pagination.BasePagination` 且重写以下两个方法：

* `paginate_queryset(self, queryset, request, view=None)`：传递给初始查询集，并应返回一个可迭代对象。该对象仅包含请求页面中的数据
* `get_paginated_response(self, data)`：该方法接收传递给序列化的页面数据，返回一个 Response 实例

```python
class CustomPagination(pagination.PageNumberPagination):
    def get_paginated_response(self, data):
        return Response({
            'links': {
                'next': self.get_next_link(),
                'previous': self.get_previous_link()
            },
            'count': self.page.paginator.count,
            'results': data
        })

REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'my_project.apps.core.pagination.CustomPagination',
    'PAGE_SIZE': 100
}
```
