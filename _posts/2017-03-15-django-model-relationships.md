---
layout: post
title: "Django 的 Model 与关系"
tags: Django
category: 计算机
---



# Model

Django 默认只有一个数据库，名字在 `settings.py` 中设定：

```json
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    }
}
```



每个 `Model` 对应一张表，表名为 `AppName_ModelName`，其中的 `Field` 对应表中的一列：

```python
from django.db import models

class Reporter(models.Model):
    first_name = models.CharField(max_length=30)
    last_name = models.CharField(max_length=30)
    email = models.EmailField()

    def __str__(self):              # __unicode__ on Python 2
        return "%s %s" % (self.first_name, self.last_name)
```

上面的定义表示 AppName_reporter 这个表中有 first_name、last_name、email 三列。

ID 字段可以省略，默认是自增的整数。

每种 `Field` 在前端页面上的显示都不一样，比如 CharField 被显示为输入条，TextField 被显示为编辑框等等。

在模板中直接输出 Reporter 对象，会调用它的 `__str__` 方法。



可以在内部类 `Meta` 里面定义 Model 的属性，常用的如下：

```python
class Article(models.Model):
    class Meta:
        ordering = ('field1', 'field2')            # 默认排序
        unique_together = ('field1', 'field2')     # 组合键唯一
        default_permissions = ()                   # 禁用默认的权限类型 (就是对每个表的 CURD 4种权限)
        permissions = (                            # 自定义权限类型
            ("perm1", "perm1 description"),
            ("perm2", "perm2 description"),
        )
```



# 关系

关系数据库中的几种关系：

* 一对一：一个酒店只能有一个地址，一个地址也只能有一个酒店。
* 一对多：一个记者可以写多篇稿子，但一个稿子只能有一个作者。
* 多对多：一本杂志可以包含多篇文章，一篇文章也可以在多本杂志上刊登。




# 一对多

一对多的关系用 `ForeignKey` ：

```python
class Reporter(models.Model):
    first_name = models.CharField(max_length=30)
    last_name = models.CharField(max_length=30)
    email = models.EmailField()

    def __str__(self):              # __unicode__ on Python 2
        return "%s %s" % (self.first_name, self.last_name)

class Article(models.Model):
    headline = models.CharField(max_length=100)
    pub_date = models.DateField()
    reporter = models.ForeignKey(Reporter, on_delete=models.CASCADE)

    def __str__(self):              # __unicode__ on Python 2
        return self.headline
```

`on_delete=models.CASCADE` 表示 Reporter 对象被删除后，Ta 的文章也会被全部清除。



使用方式如下：

```python
>>> r = Reporter(first_name='John', last_name='Smith', email='john@example.com')
>>> r.save()
>>> a = Article(id=None, headline="This is a test", pub_date=date(2005, 7, 27), reporter=r)
>>> a.save()
>>> a.reporter.id
1
>>> r.article_set.all()
<QuerySet [<Article: xxx>]>      # 返回可迭代的 QuerySet 对象，其中的元素是 Article 对象
```



其它的一些用法：

```python
>>> r.article_set.count()
>>> r.article_set.add(new_article)
>>> r.article_set.filter(headline__startswith='This')
>>> a2 = r.article_set.create(headline="xxx", pub_date=date(2005, 7, 29))

>>> Article.objects.filter(reporter__first_name='John')
>>> Article.objects.filter(reporter__first_name='John', reporter__last_name='Smith')
>>> Article.objects.filter(reporter__pk=1)    # pk 默认为 id
>>> Article.objects.filter(reporter=1)
>>> Article.objects.filter(reporter=r)
>>> Article.objects.filter(reporter__in=[1,2]).distinct()
>>> Article.objects.filter(reporter__in=Reporter.objects.filter(first_name='John')).distinct()

>>> Reporter.objects.filter(article__pk=1)
>>> Reporter.objects.filter(article=1)
>>> Reporter.objects.filter(article=a)
>>> Reporter.objects.filter(article__headline__startswith='This').distinct()
>>> Reporter.objects.order_by('first_name')

>>> Reporter.objects.filter(article__reporter__first_name__startswith='John').distinct()
>>> Reporter.objects.filter(article__reporter=r).distinct()
```



# 多对多

多对多的关系用 `ManyToManyField` ，且对应的字段名应该用复数形式，比如 publications：

```python
class Publication(models.Model):
    title = models.CharField(max_length=30)

    def __str__(self):              # __unicode__ on Python 2
        return self.title

    class Meta:
        ordering = ('title',)

class Article(models.Model):
    headline = models.CharField(max_length=100)
    publications = models.ManyToManyField(Publication)

    def __str__(self):              # __unicode__ on Python 2
        return self.headline

    class Meta:
        ordering = ('headline',)
```



在多对多关系表中增加额外的字段：

```python
class Person(models.Model):
    name = models.CharField(max_length=128)

    def __str__(self):              # __unicode__ on Python 2
        return self.name

class Group(models.Model):
    name = models.CharField(max_length=128)
    members = models.ManyToManyField(Person, through='Membership')

    def __str__(self):              # __unicode__ on Python 2
        return self.name

class Membership(models.Model):
    person = models.ForeignKey(Person, on_delete=models.CASCADE)
    group = models.ForeignKey(Group, on_delete=models.CASCADE)
    date_joined = models.DateField()
    invite_reason = models.CharField(max_length=64)
```

如果不用 `through` 参数，Django 会自动生成一张名为 `AppName_person_group` 的关系表。或者可以通过 `through` 手动指定关系表，然后在这个 Model 里面添加额外的字段。



# 一对一

一对一的关系用 `OneToOneField` ：

```python
from django.contrib.auth.models import User

class Employee(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    department = models.CharField(max_length=100)
```

Django 默认已经有 User 表了，可以直接使用。如果还想加入其它字段，这是一种简单的办法。