---
categories:
- - django
date: 2024-11-12 15:16:57
description: 在现代 web 应用中，缓存机制可以显著提高数据访问效率，减轻数据库压力。同时，信号机制则为模型的变化提供了响应机制，确保数据一致性。本文将介绍如何在 Django 中结合缓存与信号，以优化文章详情的访问和管理。
id: '334'
tags:
- web框架
title: Django入门系列6-使用缓存和信号
---


在现代 web 应用中，缓存机制可以显著提高数据访问效率，减轻数据库压力。同时，信号机制则为模型的变化提供了响应机制，确保数据一致性。本文将介绍如何在 Django 中结合缓存与信号，以优化文章详情的访问和管理。

## 1.myproject/settings.py

在 `myproject/settings.py` 中，我们配置了 Redis 作为缓存后端。通过设置 `BACKEND` 和 `LOCATION`，Django 将使用指定的 Redis 实例存储缓存数据。

```python
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://redis:6379', # redis所在服务器或容器ip地址
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
            # "PASSWORD": "your_pwd", # 你设置的密码
        },
    },
}
```

## 2.myapp/views.py

在 `myapp/views.py` 中，我们定义了一个 `ArticleDetail` 视图类。该视图实现了缓存机制：首先尝试从缓存中获取文章数据，如果缓存未命中，则从数据库查询并存储到缓存中。这种方法有效减少了数据库的查询次数。

```python
from django.core.cache import cache

class ArticleDetail(APIView):
    authentication_classes = []  
    permission_classes = [AllowAny]

    def get_object(self, pk):
        try:
            article_cache_key = f'myapp-article-{pk}'
            article = cache.get(article_cache_key)
            if not article:
                print('使用数据库查询...', flush=True)
                article = Article.objects.get(pk=pk)
                cache.set(article_cache_key, article)
            else:
                print('使用缓存查询...', flush=True)
            return article
        except Article.DoesNotExist:
            raise Http404
```

## 3.myapp/signals.py

在 `myapp/signals.py` 中，我们使用 Django 的信号机制，响应文章模型的创建、更新和删除事件。通过信号处理器，我们能够在模型变化时自动更新缓存，保持数据的实时性和一致性。

```python
from django.core.cache import cache
from django.db.models.signals import post_delete, post_save
from django.dispatch import receiver

from .models import Article


@receiver(post_save, sender=Article)
def my_save_handler(sender, instance, created, **kwargs):
    if created:
        print(f'新建模型的 ID: {instance.id}', flush=True)
    else:
        print(f'更新模型的 ID: {instance.id}', flush=True)
        article_cache_key = f'myapp-article-{instance.id}'
        cache.delete(article_cache_key)


@receiver(post_delete, sender=Article, dispatch_uid="delte_article_item")
def my_delete_handler(sender, instance, **kwargs):
    print(f'已删除模型的 ID: {instance.id}', flush=True)
    article_cache_key = f'myapp-article-{instance.id}'
    cache.delete(article_cache_key)
```

## 4.myapp/apps.py

在 `myapp/apps.py` 中，我们确保信号处理器在应用启动时被注册。通过重写 `ready` 方法，我们导入信号模块，使其在应用加载时生效。

```python
class MyappConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'myapp'

    def ready(self):
        from . import signals
```

## 5.测试

通过使用 `curl` 命令，我们可以测试缓存机制和信号处理的效果。首次请求文章时，数据从数据库获取；随后请求则从缓存中返回，确保了高效的数据访问。在删除操作中，信号处理器被触发，自动清理相应的缓存。

```python
# docker-compose重新构建web
docker-compose up -d --build web

# 查看服务端docker日志
docker-compose logs web -f

# 获取一篇文章，此时查询数据库
curl -X GET 127.0.0.1:5000/api/v1/articles/1
服务端打印：
使用数据库查询...

# 再次获取这篇文章，此时使用缓存
curl -X GET 127.0.0.1:5000/api/v1/articles/1
服务端打印：
使用缓存查询...

# 删除一篇文章，触发删除信号
curl -X DELETE 127.0.0.1:5000/api/v1/articles/1
服务端打印：
使用缓存查询...
已删除模型的 ID: 1
```

## 6.关于信号

在 Django 中，信号机制主要由以下三个关键要素构成：

*   **发送者（sender）**：指发出信号的实体，既可以是模型，也可以是视图。当特定操作发生时，发送者会触发信号。
*   **信号（signal）**：这是实际发送的信号。Django 提供了多个内置信号，例如在模型保存后触发的 `post_save` 信号。
*   **接收者（receiver）**：信号的接收者，实质上是一个回调函数。当特定事件发生时，这个函数会被注册并执行，以响应信号的发出。

Django 提供了一些内置信号（此外，还支持自定义信号）：

*    `pre_save` 和 `post_save`：在模型的 `save()` 方法调用前后触发。
*    `pre_init` 和 `post_init`：在模型的 `__init__` 方法调用前后触发。
*    `pre_delete` 和 `post_delete`：在模型的 `delete()` 方法或查询集的 `delete()` 方法调用前后触发。
*    `m2m_changed`：在模型的多对多关系发生变化时触发。
*    `request_started` 和 `request_finished`：在 Django 开始或结束 HTTP 请求时触发。

需要注意的是，监听 `pre_save` 和 `post_save` 信号的回调函数不应再次调用 `save()` 方法，以免造成死循环。此外，Django 的 `update` 方法不会触发 `pre_save` 和 `post_save` 信号。 最后，Django 的信号监听函数是同步执行的，因此如果需要异步处理耗时的任务（如发送电子邮件或写入文件），不推荐使用 Django 自带的信号机制。