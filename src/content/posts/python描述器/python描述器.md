---
title: 详解python描述器
published: 2022-11-13
tags: [Python]
category: Python
draft: false
---


Python中我们常常使用A.B这样的表达式对对象（包括类）的成员变量或函数进行访问。我们一般假定这样的访问将返回成员本身，但有多少人考虑过这样语法结构对应的底层过程呢？今天，我们就从这个.说开去，聊聊Load_Attr的运作流程，与描述符、@staticmethod、@classmethod以及@property等描述符的实现机制。

## 描述符

### 描述符概述

在聊后续内容之前，我们需要先介绍一下Python中的描述符协议。我们都知道，Python中除了显示继承来实现接口外，只要实现了特定的魔法方法就算是实现了某种协议，可以适配某些语法结构与外部类。如实现了__next__与__iter__方法的类就算是实现了迭代器协议，其就成为一个迭代器，可以用于for循环进行迭代。**描述符**也是一个协议，其包含__get__、__set__与__del__三个方法。

### 描述符的函数签名

```python
def __get__(self, instance, owner):
  """Parameter:
  self: the descriptor instance
  instance: the instance of the owner class
  owner: the class of the instance
  """
  pass

def __set__(self, instance, value):
  """Parameter:
  self: the descriptor instance
  instance: the instance of the owner class
  value: the value to be set
  """
  pass

def __delete__(self, instance):
  """Parameter:
  self: the descriptor instance
  instance: the instance of the owner class
  """
  pass

## 对于类的补充，类中找到描述符时，instance为NULL，type为自身
```

### 描述符分类

虽然实现其中任意一个就能称为描述符，但是根据实现程度的不同亦有不同的分类。我们将描述符分为两类：

1. 数据描述符：实现了__set__与__get__方法的描述符。
2. 非数据描述符：未实现__set__方法的描述符。

### 描述符用途

迭代器协议用于迭代，那么描述符协议用于什么呢？没错，正是A.B结构的访问机制。当A.B来获取B时，若B实现__get__就回访会B的__get__函数的返回值，而不一定是B本身；同样，使用A.B=C来进行赋值以及del进行删除时，将分别对应__set__与__del__方法。当然，A.B的机制绝非如此三言二语能描述清楚，在Load_Attr中将进行详细描述。

## Load_Attr

A.B这样的语法结构，将经由A.__getattribute\__(B)这样一个魔法方法来执行，其在Python字节码层面对应的是Load_Attr这条指令。其行为取决于A的类型（对象或类），但也十分相似。大体上，对象的访问流程如下：

1. 在对象的type，也就是类的中查找数据描述符，若找到则
   + **访问**:返回其__get__方法的返回值
   + **赋值**:调用__set__方法。
2. 在对象的__dict__里查找实例属性，也就是实例生成后附上的属性
   + **访问**:返回属性
   + **赋值**:修改属性
3. 在对象type里查找非数据描述符，若找到则
   + **访问**:返回其__get__方法的返回值
   + **赋值**:不进行这一步
4. 在对象type里查找类属性，也就是直接定义在类中的属性
   + **访问**:返回属性
   + **赋值**:为实例添加这一属性，并赋值
5. 若以上都未找到，则抛出AttributeError；但若是类实现了__getattr__方法，则调用其并获得返回值

类的访问流程则较为简单，直接在自己中查找数据描述符、非数据描述符、类属性即可。

## 使用场景

### 隔离访问

面向对象语言强调的是对象的自治，对于对象的成员，我们理应使用对应的方法来访问，如Java中一般借由给成员属性设置private，再提供对应getter、setter方法来避免直接访问。但在python中缺缺乏访问描述符，因此我们只能通过其他手段来实现，描述符就很好地提供了这一机制。

当然，为每一个属性定义一个类是十分繁琐的，因此我们可以使用内置的property类来实现。property类的构造函数接受三个参数，分别是：fget,fset,fdel，对应描述器的三个方法。

Property类实现如下：

```python
class Property:
    def __init__(self, fget=None, fset=None, fdel=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel

    def __get__(self, instance, owner):
        if instance is None:
            return self
        if self.fget is None:
            raise AttributeError("unreadable attribute")
        return self.fget(instance)

    def __set__(self, instance, value):
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(instance, value)

    def __delete__(self, instance):
        if self.fdel is None:
            raise AttributeError("can't delete attribute")
        self.fdel(instance)

    def getter(self, fget):
        return type(self)(fget, self.fset, self.fdel)

    def setter(self, fset):
        return type(self)(self.fget, fset, self.fdel)

    def deleter(self, fdel):
        return type(self)(self.fget, self.fset, fdel)
```

结合装饰器机制，我们可以简便地将某个属性实现为描述器,仅需提供描述器所需的三个方法即可。

``` python

class A:
  def __init__(self):
    self.a = 1
  @property
  def a(self):  # 对应fget，但一般不称为get_a，而是直接称为a，便于外部访问
    return self.a
  @get_a.setter
  def a(self, value):
    self.a = value
  @get_a.deleter
  def a(self):
    del self.a

```

### 类方法与静态方法

@classmethod以及@staticmethod装饰器的实现原理就是描述符。

### 函数绑定

方法调用时，我们不需要传递self参数，这是因为python会自动将self传递给方法。这是因为python会将方法转换为描述符，然后调用描述符的__get__方法，将self作为参数传递给方法。
