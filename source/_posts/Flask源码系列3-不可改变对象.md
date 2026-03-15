---
categories:
- - flask
date: 2024-04-19 10:07:12
description: "本篇文章介绍flask引用的werkzeug包中的不可改变对象的使用，其具体思路是通过继承基础类型，并通过重写将其能改变自身的方法覆盖掉，直接抛出异常，从而实现了对象不可改变的特性。"
id: '170'
tags:
- python
- web框架
title: Flask源码系列3-不可改变对象
---


本篇文章介绍flask引用的werkzeug包中的不可改变对象的使用，其具体思路是通过继承基础类型，并通过重写将其能改变自身的方法覆盖掉，直接抛出异常，从而实现了对象不可改变的特性。

flask源码：

```python
# app.py
default_config = ImmutableDict(
        {
            "DEBUG": None,
            "TESTING": False,
            "PROPAGATE_EXCEPTIONS": None,
            "SECRET_KEY": None,
            ...
        }
    )
```

## 1.ImmutableDict

以下是不可改变的字典，这些函数可以通过 `dir(dict)` 查询。

```python

from itertools import repeat

def is_immutable(self):
    raise TypeError(f"{type(self).__name__!r} objects are immutable")

class ImmutableDictMixin:
    """Makes a :class:`dict` immutable.

    .. versionadded:: 0.5

    :private:
    """

    _hash_cache = None

    @classmethod
    def fromkeys(cls, keys, value=None):
        instance = super().__new__(cls)
        instance.__init__(zip(keys, repeat(value)))
        return instance

    def __reduce_ex__(self, protocol):
        return type(self), (dict(self),)

    def _iter_hashitems(self):
        return self.items()

    def __hash__(self):
        if self._hash_cache is not None:
            return self._hash_cache
        rv = self._hash_cache = hash(frozenset(self._iter_hashitems()))
        return rv

    def setdefault(self, key, default=None):
        is_immutable(self)

    def update(self, *args, **kwargs):
        is_immutable(self)

    def pop(self, key, default=None):
        is_immutable(self)

    def popitem(self):
        is_immutable(self)

    def __setitem__(self, key, value):
        is_immutable(self)

    def __delitem__(self, key):
        is_immutable(self)

    def clear(self):
        is_immutable(self)

class ImmutableDict(ImmutableDictMixin, dict):
    """An immutable :class:`dict`.

    .. versionadded:: 0.5
    """

    def __repr__(self):
        return f"{type(self).__name__}({dict.__repr__(self)})"

    def copy(self):
        """Return a shallow mutable copy of this object.  Keep in mind that
        the standard library's :func:`copy` function is a no-op for this class
        like for any other python immutable type (eg: :class:`tuple`).
        """
        return dict(self)

    def __copy__(self):
        return self

if __name__ == '__main__':
    default_config = ImmutableDict(
        {
            "DEBUG": None,
            "TESTING": False,
            "PROPAGATE_EXCEPTIONS": None,
            "SECRET_KEY": None,
        }
    )

    default_config["DEBUG"] = True

$
Traceback (most recent call last):
  File "C:\test\main.py", line 88, in <module>
    default_config["DEBUG"] = True
  File "C:\test\main.py", line 49, in __setitem__
    is_immutable(self)
  File "C:\test\main.py", line 5, in is_immutable
    raise TypeError(f"{type(self).__name__!r} objects are immutable")
TypeError: 'ImmutableDict' objects are immutable
```

## 2.ImmutableList

同理，以下是不可改变的列表。

```python
def is_immutable(self):
    raise TypeError(f"{type(self).__name__!r} objects are immutable")

class ImmutableListMixin:
    """Makes a :class:`list` immutable.

    .. versionadded:: 0.5

    :private:
    """

    _hash_cache = None

    def __hash__(self):
        if self._hash_cache is not None:
            return self._hash_cache
        rv = self._hash_cache = hash(tuple(self))
        return rv

    def __reduce_ex__(self, protocol):
        return type(self), (list(self),)

    def __delitem__(self, key):
        is_immutable(self)

    def __iadd__(self, other):
        is_immutable(self)

    def __imul__(self, other):
        is_immutable(self)

    def __setitem__(self, key, value):
        is_immutable(self)

    def append(self, item):
        is_immutable(self)

    def remove(self, item):
        is_immutable(self)

    def extend(self, iterable):
        is_immutable(self)

    def insert(self, pos, value):
        is_immutable(self)

    def pop(self, index=-1):
        is_immutable(self)

    def reverse(self):
        is_immutable(self)

    def sort(self, key=None, reverse=False):
        is_immutable(self)

class ImmutableList(ImmutableListMixin, list):
    """An immutable :class:`list`.

    .. versionadded:: 0.5

    :private:
    """

    def __repr__(self):
        return f"{type(self).__name__}({list.__repr__(self)})"

if __name__ == '__main__':
    default_list = ImmutableList([1, 2, 3])

    default_list.append(4)

$
Traceback (most recent call last):
  File "C:\test\main.py", line 73, in <module>
    default_list.append(4)
  File "C:\test\main.py", line 37, in append
    is_immutable(self)
  File "C:\test\main.py", line 2, in is_immutable
    raise TypeError(f"{type(self).__name__!r} objects are immutable")
TypeError: 'ImmutableList' objects are immutable
```

## 3.Mixin

Mixin 类是一种在面向对象编程中用于组合类功能的技术。Mixin 类通常是具有一组特定功能的类，可以被其他类包含或混合使用，以增强这些类的功能。

在flask orm中使用Mixin类增强模型功能的实例：

```python
from contextlib import contextmanager
from flask_sqlalchemy import SQLAlchemy as _SQLAlchemy

class SQLAlchemy(_SQLAlchemy):
    @contextmanager
    def auto_commit(self):
        try:
            yield
            self.session.commit()
        except Exception as e:
            self.session.rollback()
            raise e

db = SQLAlchemy()

# Mixin类
class CRUDMixin(object):
    __table_args__ = {'extend_existing': True}

    id = db.Column(db.Integer, primary_key=True)

    @classmethod
    def create(cls, **kwargs):
        instance = cls(**kwargs)
        return instance.save()

    @classmethod
    def get_by_id(cls, id):
        if any((isinstance(id, str) and id.isdigit(), isinstance(id, (int, float))), ):
            return cls.query.get(int(id))
        return None

    @classmethod
    def filter_first(cls, *args):
        return cls.query.filter(*args).first()

    @classmethod
    def filter_all(cls, *args):
        return cls.query.filter(*args).all()

    def update(self, **kwargs):
        with db.auto_commit():
            for attr, value in kwargs.items():
                setattr(self, attr, value)

    def save(self):
        with db.auto_commit():
            db.session.add(self)
        return self

    def delete(self):
        with db.auto_commit():
            db.session.delete(self)

    # 将模型转换为字典形式
    def to_dict(self, fileds=None, int_bool=False):
        if fileds is None:
            return {
                column.name: int(getattr(self, column.name))
                if int_bool and isinstance(getattr(self, column.name), bool) else getattr(self, column.name)
                for column in self.__table__.columns
            }
        else:
            return {
                filed: int(getattr(self, filed))
                if int_bool and isinstance(getattr(self, filed), bool) else getattr(self, filed)
                for filed in fileds
            }

# 模型继承Mixin类
class User(CRUDMixin, db.Model):
    """
    用户表
    """
    id = db.Column(db.Integer, primary_key=True)
    ...
```