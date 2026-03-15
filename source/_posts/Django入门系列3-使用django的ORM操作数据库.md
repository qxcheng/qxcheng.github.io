---
categories:
- - django
date: 2024-11-12 15:12:13
description: 本文旨在介绍如何利用Django框架的ORM（对象关系映射）系统来定义数据库表结构，以及如何执行数据的增加、删除、更新和查询等基本操作。
id: '326'
tags:
- web框架
title: Django入门系列3-使用django的ORM操作数据库
---


本文旨在介绍如何利用Django框架的ORM（对象关系映射）系统来定义数据库表结构，以及如何执行数据的增加、删除、更新和查询等基本操作。

## 1.myapp/models.py

这里介绍了常用的一对多和多对多关系的创建方法，此处使用了django提供的外键，也可以使用逻辑外键来完成。

```python
from django.db import models


# 作者 和文章是一对多关系，和组是多对多关系
class Author(models.Model):
    name = models.CharField(max_length=20, db_index=True)
    mobile = models.CharField(max_length=11, unique=True)
    # null=True表示数据库可存储null值，blank=True表示表单验证时可以不填
    age = models.IntegerField(blank=True, null=True, default=0)
    birth_date = models.DateField()
    # 每次保存时重置为当前时间
    mod_date = models.DateTimeField(auto_now=True)


# 文章 一对多
class Article(models.Model):
    title = models.CharField(max_length=90, db_index=True)
    body = models.TextField(blank=True)
    # 第一次创建时设置为当前时间
    create_date = models.DateTimeField(auto_now_add=True)  
    # 外键关联
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name='author_articles')  

    def __str__(self):
        return self.title

    class Meta:
        # 按'create_date'降序排列，升序排列：['create_date']，多字段排列：['create_date', 'title']
        ordering = ['-create_date']        
        verbose_name = "Article"
        verbose_name_plural = "Articles"
        # 联合约束：一旦二者都相同，则会被Django拒绝创建
        unique_together = (('title', 'create_date'),)


# 组 多对多
class Group(models.Model):
    name = models.CharField(max_length=50)
    members = models.ManyToManyField(
        Author,
        through='Membership',               # 中间表
        through_fields=('group', 'author'), # 显示指明关系连接字段
        related_name="author_groups"
    )


# 中间表
class Membership(models.Model):
    group = models.ForeignKey(Group, on_delete=models.CASCADE)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    inviter = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="membership_invites")
    invite_reason = models.CharField(max_length=64)
```

### on\_delete参数

*    `CASCADE`：级联删除。当你删除Author记录时，与之关联的所有 Article 都会被删除。
*    `PROTECT`: 保护模式。如果有外键关联，就不允许删除，删除的时候会抛出ProtectedError错误，除非先把关联了外键的记录删除掉。例如想要删除Author，那你要把所有关联了该Author的Article全部删除才可能删Author。
*    `SET_NULL`: 置空模式。删除的时候，外键字段会被设置为空。删除Author后，Article 记录里面的author\_id 就置为null了。
*    `SET_DEFAULT`: 置默认值，删除的时候，外键字段设置为默认值。
*    `SET`: 自定义一个值。
*    `DO_NOTHING`：什么也不做。删除不报任何错，外键值依然保留，但是无法用这个外键去做查询。

### related\_name参数

`related_name`参数用于`ForeignKey`、`OneToOneField`和`ManyToManyField`字段中，它定义了反向关系的名称。这个参数允许你从关联的对象访问当前模型的实例。如Author对象通过`author_articles`属性访问Article对象列表。

### 其他内容

*    一对一关系类型`OneToOneFiled`的使用场景相对其他两种关系要少，经常用于对已有模型的扩展。
*    在Django ORM中，模型的元数据设置是通过在模型内部定义一个名为`Meta`的类来完成的。这个类提供了多种选项，允许你控制模型的行为和数据库表的属性。
*    在Django中，模型继承是一个强大功能，它允许你通过继承一个模型来创建新的模型。Django支持三种类型的模型继承：抽象继承、单表继承、多表继承。

## 2.myapp/serializers.py

在Django REST Framework中，涉及到复杂的嵌套模型关系时，一般需要重写序列化器的`create`和`update`方法。其在创建或更新数据时触发，若初始化序列化器实例时传递了instance参数，则在调用`save`方法时触发`update`，否则触发`create`方法。下一篇文章将继续介绍序列化类的使用和编写方法。

```python
from rest_framework import serializers

from .models import Article, Author, Group, Membership


class AuthorSerializer(serializers.ModelSerializer):
    name = serializers.CharField(max_length=20, required=True)
    mobile = serializers.CharField(max_length=11, required=True)
    age = serializers.IntegerField(required=False)
    birth_date = serializers.DateField(format="%Y-%m-%d", required=True)
    mod_date = serializers.DateTimeField(format="%Y-%m-%d %H:%M:%S", required=False, read_only=True)

    class Meta:
        model = Author
        fields = '__all__'


class ArticleSerializer(serializers.ModelSerializer):
    title = serializers.CharField(max_length=90, required=True)
    body = serializers.CharField(max_length=2048, required=True)
    create_date = serializers.DateTimeField(format="%Y-%m-%d %H:%M:%S", required=False, read_only=True)

    author = AuthorSerializer()

    class Meta:
        model = Article
        fields = '__all__'

    def create(self, validated_data):
        author_data = validated_data.pop('author')
        author = Author.objects.create(**author_data)
        article = Article.objects.create(author=author, **validated_data)
        return article
    
    def update(self, instance, validated_data):
        author_data = validated_data.pop('author')
        author = Author.objects.create(**author_data)
        instance.author = author
        instance.save()
        return instance


class GroupSerializer(serializers.ModelSerializer):
    name = serializers.CharField(max_length=50, required=True)
    members = AuthorSerializer(many=True)
    
    class Meta:
        model = Group
        fields = '__all__'

    def create(self, validated_data):
        authors_data = validated_data.pop('members')
        group = Group.objects.create(**validated_data)

        # 处理多对多关系
        for author_data in authors_data:
            author, _ = Author.objects.get_or_create(**author_data)
            #group.members.add(author)
            Membership.objects.create(group=group, author=author, inviter=author)

        return group
```

## 3.myapp/views.py

此处是视图函数代码，drf还提供了视图类和视图集来简化视图函数的代码，但是是以牺牲可读性为代价的，这里不作介绍。

```python
from rest_framework import status
from rest_framework.views import APIView
from rest_framework.response import Response
from django.http import Http404

from .serializers import ArticleSerializer, AuthorSerializer, GroupSerializer
from .models import Article, Author, Group


class ArticleList(APIView):
    # 获取所有文章
    def get(self, request, format=None):
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True)
        return Response(serializer.data)
    
    # 新增文章
    def post(self, request, format=None):
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    
class ArticleDetail(APIView):
    def get_object(self, pk):
        try:
            return Article.objects.get(pk=pk)
        except Article.DoesNotExist:
            raise Http404
    
    # 获取一篇文章
    def get(self, request, pk, format=None):
        article = self.get_object(pk)
        serializer = ArticleSerializer(article)
        return Response(serializer.data)
    
    # 修改一篇文章
    def put(self, request, pk, format=None):
        article = self.get_object(pk)
        serializer = ArticleSerializer(instance=article, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    # 删除一篇文章
    def delete(self, request, pk, format=None):
        article = self.get_object(pk)
        article.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)


class AuthorDetail(APIView):
    def get_object(self, pk):
        try:
            return Author.objects.get(pk=pk)
        except Author.DoesNotExist:
            raise Http404
    
    # 获取一个作者
    def get(self, request, pk, format=None):
        author = self.get_object(pk)
        serializer = AuthorSerializer(author)
        return Response(serializer.data)
    
    # 新增一个作者
    def post(self, request, format=None):
        serializer = AuthorSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    

class GroupDetail(APIView):
    def get_object(self, pk):
        try:
            return Group.objects.get(pk=pk)
        except Group.DoesNotExist:
            raise Http404
    
    # 获取一个组
    def get(self, request, pk, format=None):
        group = self.get_object(pk)
        serializer = GroupSerializer(group)
        return Response(serializer.data)
    
    # 创建一个组
    def post(self, request, format=None):
        serializer = GroupSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

## 4.myapp/urls.py

将视图与 URL 路由关联：

```python
urlpatterns = [
    re_path(r'^articles/$', views.ArticleList.as_view()),
    re_path(r'^articles/(?P<pk>[0-9]+)$', views.ArticleDetail.as_view()),
    re_path(r'^authors/$', views.AuthorDetail.as_view()),
    re_path(r'^authors/(?P<pk>[0-9]+)$', views.AuthorDetail.as_view()),
    re_path(r'^groups/$', views.GroupDetail.as_view()),
    re_path(r'^groups/(?P<pk>[0-9]+)$', views.GroupDetail.as_view()),
]
```

## 5.测试

```python
# docker-compose重新构建web
docker-compose up -d --build web

# 生成迁移文件
docker exec -it django-tutorial-web-1 /bin/sh
python manage.py makemigrations myapp
# 这里可以根据提示为article中的author设置默认值为None

# docker-compose重启web
docker-compose restart web 

# 新增author
curl -H "Content-Type: application/json" -X POST -d '{"name":"author101", "mobile":"101", "birth_date":"2001-01-01"}' 127.0.0.1:5000/api/v1/authors/

curl -H "Content-Type: application/json" -X POST -d '{"name":"author102", "mobile":"102", "birth_date":"2001-01-01"}' 127.0.0.1:5000/api/v1/authors/

# 新增group
curl -H "Content-Type: application/json" -X POST -d '{"name":"group01", "members":[{"name":"author103", "mobile":"103", "birth_date":"2001-01-01"},{"name":"author104", "mobile":"104", "birth_date":"2001-01-01"}]}' 127.0.0.1:5000/api/v1/groups/

# 增
curl -H "Content-Type: application/json" -X POST -d '{"title":"mytitle", "body":"mybody", "author": {"name":"author105", "mobile":"105", "birth_date":"2001-01-01"}}' 127.0.0.1:5000/api/v1/articles/

# 查列表
curl 127.0.0.1:5000/api/v1/articles/
[{"id":1,"title":"mytitle","body":"mybody","create_date":"2024-10-23 13:49:31","author":{"id":8,"name":"author105","mobile":"105","age":0,"birth_date":"2001-01-01","mod_date":"2024-10-23 13:49:31"}}]

# 查
curl 127.0.0.1:5000/api/v1/articles/1
{"id":1,"title":"mytitle","body":"mybody","create_date":"2024-10-23 13:49:31","author":{"id":8,"name":"author105","mobile":"105","age":0,"birth_date":"2001-01-01","mod_date":"2024-10-23 13:49:31"}}

# 改
curl -H "Content-Type: application/json" -X PUT -d '{"title":"notitle", "body":"nobody", "author": {"name":"author106", "mobile":"106", "birth_date":"2001-01-01"}}' 127.0.0.1:5000/api/v1/articles/1

# 删
curl -X DELETE 127.0.0.1:5000/api/v1/articles/1
```