---
categories:
- - django
date: 2024-11-12 15:14:06
description: 本文将介绍如何在 Django REST Framework (DRF) 中使用序列化器进行数据验证和输出定制。序列化器不仅用于将复杂的数据类型（如模型实例）转换为可读的格式（如
  JSON），还可以帮助验证输入数据的有效性，并根据需要调整输出格式。通过序列化器，我们可以更加简洁地处理复杂数据的读写操作。
id: '328'
tags:
- web框架
title: Django入门系列4-使用序列化和参数校验
---


本文将介绍如何在 Django REST Framework (DRF) 中使用序列化器进行数据验证和输出定制。序列化器不仅用于将复杂的数据类型（如模型实例）转换为可读的格式（如 JSON），还可以帮助验证输入数据的有效性，并根据需要调整输出格式。通过序列化器，我们可以更加简洁地处理复杂数据的读写操作。

## 1.myapp/serializers.py

1.  `read_only_fields`用于设定仅可读的字段，这些字段在反序列化（接收数据）时不会被修改。
2.  `SerializerMethodField` 允许我们在序列化器中添加自定义字段，通常用于返回计算得出的数据或者从关联数据中提取特定信息。
3.   在序列化器中，字段不仅可以是基本类型，还可以是其他序列化器。这允许我们表示更加复杂的关系，如在 `ArticleSerializer` 中添加 `AuthorSerializer`。
4.   DRF 提供了多种字段来表示与其他模型的关系，常用的包括：
    *    `PrimaryKeyRelatedField`：只返回关联对象的主键。
    *    `StringRelatedField`：返回模型中定义的 `__str__` 方法返回的字符串。
    *    `HyperlinkedRelatedField`：返回一个包含链接的字段。
5.  5\. 介绍了两种字段验的方法：字段级别验证、对象级别验证，django还提供了验证器和定义好的验证器
    *    字段级别验证通过在序列化器中定义 `validate_<field_name>` 方法来实现。当字段的值不符合预期时，可以抛出 `ValidationError` 异常。
    *    对象级别的验证涉及整个序列化对象的数据，通常用于检查多个字段之间的依赖关系。对象级别验证通过定义 `validate` 方法实现。

```python
from rest_framework import serializers

from .models import Article, Author, Group, Membership


class AuthorSerializer(serializers.ModelSerializer):
    name = serializers.CharField(max_length=20, required=True)
    mobile = serializers.CharField(max_length=11, required=True)
    age = serializers.IntegerField(required=False)
    birth_date = serializers.DateField(format="%Y-%m-%d", required=True)
    mod_date = serializers.DateTimeField(format="%Y-%m-%d %H:%M:%S", required=False, read_only=True)

    # 关系序列化，只提供主键
    author_articles = serializers.PrimaryKeyRelatedField(many=True, read_only=True)

    class Meta:
        model = Author
        fields = '__all__'
        # 设定仅可读的字段
        read_only_fields = ('id', 'mod_date', 'articles')


class ArticleSerializer(serializers.ModelSerializer):
    title = serializers.CharField(max_length=90, required=True)
    body = serializers.CharField(max_length=2048, required=True)
    create_date = serializers.DateTimeField(format="%Y-%m-%d %H:%M:%S", required=False, read_only=True)

    author = AuthorSerializer()

    # 添加自定义的输出字段，需提供 get_article_type 方法
    article_type = serializers.SerializerMethodField()

    class Meta:
        model = Article
        fields = '__all__'
        read_only_fields = ('id', 'create_date')

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
    
    # 自定义的输出字段，方法名为：get_字段名
    def get_article_type(self, obj):
        length = len(obj.body)
        if length > 5000:
            return "长篇"
        elif length > 1000:
            return "中篇"
        else:
            return '短篇'
    
    # 字段级别验证，方法名为：validate_字段名
    def validate_title(self, value):
        if len(value) < 3:
            raise serializers.ValidationError("Title is too short")
        return value
    
    # 对象级别验证，方法名固定
    def validate(self, data):
        if len(data['title']) > len(data['body']):
            raise serializers.ValidationError("Title is longer than body")
        return data


class GroupSerializer(serializers.ModelSerializer):
    name = serializers.CharField(max_length=50, required=True)
    members = AuthorSerializer(many=True)
    
    class Meta:
        model = Group
        fields = '__all__'
        read_only_fields = ('id', )

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

## 2.测试

```bash
# docker-compose重新构建web
docker-compose up -d --build web

# 添加article失败示例
curl -H "Content-Type: application/json" -X POST -d '{"title":"my", "body":"mybody", "author": {"name":"author107", "mobile":"107", "birth_date":"2001-01-01"}}' 127.0.0.1:5000/api/v1/articles/

{"title":["Title is too short"]}

# 添加article
curl -H "Content-Type: application/json" -X POST -d '{"title":"mytitle", "body":"mybody++", "author": {"name":"author107", "mobile":"107", "birth_date":"2001-01-01"}}' 127.0.0.1:5000/api/v1/articles/

# 查article
curl 127.0.0.1:5000/api/v1/articles/1

{"id":1,"title":"mytitle","body":"mybody++","create_date":"2024-10-25 15:15:36","author":{"id":10,"name":"author107","mobile":"107","age":0,"birth_date":"2001-01-01","mod_date":"2024-10-25 15:15:36","author_articles":[1]},"article_type":"短篇"}
```