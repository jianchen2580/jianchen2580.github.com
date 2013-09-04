---
layout: post
title: "Python Design"
description: ""
category: 
tags: [Python, Design Pattern]
---
{% include JB/setup %}

#python设计模式之享元模式#
享元模式
Flyweight模式，顾名思义，就是共享元数据，在python这个动态语言中可以提高程序性能和效率，比如从一个文件读取很多的字符串，而这些字符串必定重复，所以可以使用这个模式将其存在一个pool中

python的例子(我将提出一系列的例子不同方式实现这个功能)
普通青年版，看了Gof，可能就会有这样基础的一段代码:

    class Spam(object):
        def __init__(self, a, b):
            self.a = a
            self.b = b

    class SpamFactory(object):
        def  __init__(self):
            self.__instances = dict()

        def get_instance(self, a, b):
            # 在实例化后生成一个字典，当新的元祖对不存在就缓存起来
            if (a, b) not in self.__instances:
                self.__instances[(a, b)] = Spam(a, b)
            return self.__instances[(a, b)]

    # 这个和上面的意思完全一样， 这是为了在最后断言下2个工厂缓存的数据是不是能共享
    class Egg(object):
        def __init__(self, x, y):
            self.x = x
            self.y = y

    class EggFactory(object):
        def __init__(self):
            self.__instances = dict()

        def get_instance(self, x, y):
            if (x, y) not in self.__instances:
                self.__instances[(x, y)] = Egg(x, y)
            return self.__instances[(x, y)]

    spamFactory = SpamFactory()
    eggFactory = EggFactory()

    assert spamFactory.get_instance(1, 2) is spamFactory.get_instance(1, 2)
    assert eggFactory.get_instance('a', 'b') is eggFactory.get_instance('a', 'b')
    # spamFactory存储的这个元祖和eggFactory存储的不是一个东西
    assert spamFactory.get_instance(1, 2) is not eggFactory.get_instance(1, 2)

上面的是最简单能想到的，也是我见过用得最多的一种， 可是还有梦想青年版， 因为上面的东西是可以抽象出来的, 比如上面只能传入a,b2个，再多了就不行了，其他的风格也不行，比如我想缓存键值对，所以想起来了args和\*kwargs上面的是最简单能想到的，也是我见过用得最多的一种， 可是还有梦想青年版， 因为上面的东西是可以抽象出来的,
比如上面只能传入a,b2个，再多了就不行了，其他的风格也不行，比如我想缓存键值对，所以想起来了args和\*kwargs

    class FlyweightFactory(object):
        def __init__(self, cls):
            self._cls = cls
            self._instances = dict()
        # 使用*args, **kargs这样的方式抽象实现的cls的参数数量和类型
        def get_instance(self, *args, **kargs):
            # 如果键在字典中，返回这个键所对应的值。如果键不在字典中，向字典 中插入这个键
            return self._instances.setdefault(
                                    (args, tuple(kargs.items())),
                                    self._cls(*args, **kargs))


    class Spam(object):
        # 我这里实现的是3个参数，这个随意
        def __init__(self, a, b, c):
            self.a = a
            self.b = b
            self.c = c

    class Egg(object):
        def __init__(self, x, y, z):
            self.x = x
            self.y = y
            self.z = z


    SpamFactory = FlyweightFactory(Spam)
    EggFactory = FlyweightFactory(Egg)

    assert SpamFactory.get_instance(1, 2, 3) is SpamFactory.get_instance(1, 2, 3)
    assert EggFactory.get_instance('a', 'b', 'c') is EggFactory.get_instance('a', 'b', 'c')
    assert SpamFactory.get_instance(1, 2, 3) is not EggFactory.get_instance(1, 2, 3)

然后就是我最喜欢的风格，装饰器解决这个问题，算是老土的文艺青年吧

    # 这个是装饰器，主要用来将操作工厂当参数传入，拦截操作工厂的调用

    class flyweight(object):
        def __init__(self, cls):
            self._cls = cls
            self._instances = dict()
        # 重载括号操作符， 你想啊，加了装饰器就会调用，也就会触发__call__
        def __call__(self, *args, **kargs):
            return self._instances.setdefault(
                                        (args, tuple(kargs.items())),
                                        self._cls(*args, **kargs))


    @flyweight
    class Spam(object):
        def __init__(self, a, b):
            self.a = a
            self.b = b


    @flyweight
    class Egg(object):
        def __init__(self, x, y):
            self.x = x
            self.y = y


    assert Spam(1, 2) is Spam(1, 2)
    assert Egg('a', 'b') is Egg('a', 'b')
    assert Spam(1, 2) is not Egg(1, 2)

但是我们实在态out了，首先，没必要把装饰器搞成一个类，完全可以使用函数式编程

    # instances是闭包，好懂吧

    def flyweight(cls):
        instances = dict()
        return lambda *args, **kargs: instances.setdefault(
                                                (args, tuple(kargs.items())),
                                                cls(*args, **kargs))


    @flyweight
    class Spam(object):
        def __init__(self, a, b):
            self.a = a
            self.b = b


    @flyweight
    class Egg(object):
        def __init__(self, x, y):
            self.x = x
            self.y = y


    assert Spam(1, 2) is Spam(1, 2)
    assert Egg('a', 'b') is Egg('a', 'b')
    assert Spam(1, 2) is not Egg(1, 2)

该是刚进城的文艺小青版版了，这里用了一个东西Mixin: 给某个具体的类一些它需要的具体功能

    # 这是Mixin类，它提供了get_instance， 谁需要这个方法谁继承，不需要不继承
    class FlyweightMixin(object):

        _instances = dict()

        @classmethod
        def get_instance(cls, *args, **kargs):
            return cls._instances.setdefault(
                                    (cls, args, tuple(kargs.items())), 
                                    cls(*args, **kargs))


    class Spam(FlyweightMixin):

        def __init__(self, a, b):
            self.a = a
            self.b = b


    class Egg(FlyweightMixin):

        def __init__(self, x, y):
            self.x = x
            self.y = y


    assert Spam.get_instance(1, 2) is Spam.get_instance(1, 2)
    assert Egg.get_instance('a', 'b') is Egg.get_instance('a', 'b')
    assert Spam.get_instance(1, 2) is not Egg.get_instance(1, 2)

    唯一不爽的是调用方式：XX.get_instance(A， B)，不够高端

    class FlyweightMixin(object):
        _instances = dict()
        def __init__(self, *args, **kargs):
            # 只想被继承不想被初始化
            raise NotImplementedException
        # 重载实例化触发的__new__
        def __new__(cls, *args, **kargs):
            return cls._instances.setdefault(
                        (cls, args, tuple(kargs.items())),
                        super(type(cls), cls).__new__(cls, *args, **kargs))


    class Spam(FlyweightMixin):

        def __init__(self, a, b):
            self.a = a
            self.b = b


    class Egg(FlyweightMixin):

        def __init__(self, x, y):
            self.x = x
            self.y = y


    assert Spam(1, 2) is Spam(1, 2)
    assert Egg('a', 'b') is Egg('a', 'b')
    assert Spam(1, 2) is not Egg(1, 2)

嗯， 这样就和以前一样好看了，可是 什么文艺青年？ 差得很远， 一个想成为文艺青年的炫技版:

    @classmethod
    def _get_instance(cls, *args, **kargs):
        return cls.__instances.setdefault(
                                    (args, tuple(kargs.items())),
                                    super(type(cls), cls).__new__(*args, **kargs))
    # 其实绕了个圈子在操作工厂实例化的时候拦截执行上面的类方法
    def flyweight(decoree):
        decoree.__instances = dict()
        decoree.__new__ = _get_instance
        return decoree


    @flyweight
    class Spam(object):
        def __init__(self, a, b):
            self.a = a
            self.b = b


    @flyweight
    class Egg(object):
        def __init__(self, x, y):
            self.x = x
            self.y = y


    assert Spam(1, 2) is Spam(1, 2)
    assert Egg('a', 'b') is Egg('a', 'b')
    assert Spam(1, 2) is not Egg(1, 2)

无语的文艺，好吧，现在就大众经常看见的文艺青年(穿着文艺，其实骨子里不文艺)版

    # 赋予创建类的控制权,也就是python的元类metaclass
    class MetaFlyweight(type):
        def __init__(cls, *args, **kargs):
            type.__init__(cls, *args, **kargs)
            cls.__instances = dict()
            # 当你实例化cls的时候，实例的结果其实是执行_get_instance方法
            cls.__new__ = cls._get_instance

        def _get_instance(cls, *args, **kargs):
            return cls.__instances.setdefault(
                                        (args, tuple(kargs.items())),
                                        super(cls, cls).__new__(*args, **kargs))


    class Spam(object):
        # 提供你生成这个类的模板
        __metaclass__ = MetaFlyweight

        def __init__(self, a, b):
            self.a = a
            self.b = b


    class Egg(object):
        __metaclass__ = MetaFlyweight

        def __init__(self, x, y):
            self.x = x
            self.y = y


    assert Spam(1, 2) is Spam(1, 2)
    assert Egg('a', 'b') is Egg('a', 'b')
    assert Spam(1, 2) is not Egg(1, 2)

这个当然也可以搞成类方法的风格，我只列出关键的代码

    smethod
    def _get_instance(cls, *args, **kargs):
        return cls.__instances.setdefault(
                                    (args, tuple(kargs.items())),
                                    super(type(cls), cls).__new__(*args, **kargs))

    def metaflyweight(name, parents, attrs):
        cls = type(name, parents, attrs)
        cls.__instances = dict()
        cls.__new__ = _get_instance
        return cls

好吧，终极的文艺青年版- 这是一个纯函数式使用元类的方法：

    metaflyweight = lambda name, parents, attrs: type(
            name,
            parents,
            dict(attrs.items() + [
                ('__instances', dict()),
                ('__new__', classmethod(
                    lambda cls, *args, **kargs: cls.__instances.setdefault(
                                    tuple(args),
                                    super(type(cls), cls).__new__(*args, **kargs))
                    )
                )
            ])
        )


    class Spam(object):
        __metaclass__ = metaflyweight

        def __init__(self, a, b):
            self.a = a
            self.b = b


    class Egg(object):
        __metaclass__ = metaflyweight

        def __init__(self, x, y):
            self.x = x
            self.y = y


    assert Spam(1, 2) is Spam(1, 2)
    assert Egg('a', 'b') is Egg('a', 'b')
    assert Spam(1, 2) is not Egg(1, 2)

你到了那个境界呢？
