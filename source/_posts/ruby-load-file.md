---
title: 浅淡Ruby的文件加载与继承
date: 2016-08-29
tags: ["Ruby", "文件加载"]
---
## 文件加载

Ruby中load一个文件有四种方式，`require`、`require_relative`、`load`、`autoload`。

### require和require_relative

`require`是Ruby中最常见的加载一个文件的方式，如调用`require 'rails'`时，会`$LOAD_PATH`下寻找名为`rails`的`gem`包，然后将其`lib`文件夹下的同名文件加载到Ruby虚拟机中来。`多次require同一个包，只会加载一次。`

`require_relative`与`require`类似，它只会在第一次调用时加载。区别是`require_relative`的调用是相对路径。如当前文件夹下存在一个名为`foo.rb`的文件时，调用的方式为`require_relative 'foo'`。`它不能调用$LOAD_PATH中的包`。

大约是Ruby2.0以后，`require`也支持了相对路径的加载。比如上面的`foo.rb`在当前目录时，通过`require './foo'`也能达到`require_relative 'foo'`的效果。

### load

`load`也是加载一个文件，它与`require_relative`的区别是：
- `require_relative`多次加载同一文件时，只会加载一次；`load`每一次调用都会重加载该文件。

### autoload

`autoload`是一种重要的加载方式，与`require`的区别是`require是即时加载`，`autoload`的加载是`懒加载`，即在需要它的时候才会被加载。`autoload在某一个作用域内多次加载也只会被加载一次，因此不要以为它与load方法相似。`

如果当前路径下有`a.rb`、`b.rb`两个文件：
```ruby
class A
  autoload :B, './b'
  # 与 require './b' 等价，但autoload只有在调用 A::B 的时候才会去加载
end
```

`autoload`第一个参数是类名的符号，第二个参数是加载的路径。它同时支持加载`$LOAD_PATH`里的文件和`相对路径`、`绝对路径`的文件。

`Rails`中重定义了`autoload`方法，补充了一下path为空的情况下的常量寻找方法：

```ruby
class Foo
  require "active_support/all"
  extend ActiveSupport::Autoload
  autoload :B, "b"
end

class Bar
  require "active_support/all"
  extend ActiveSupport::Autoload
  autoload :B
end
```

`Bar`中的`autoload`没有指定加载文件的路径，`Rails`会自动生成加载路径`bar/b`，而`Foo`中的路径已指明，加载的路径是`b`，因此两者加载的路径是不一样的。这是`Rails`中`autoload`与`Ruby`中`autoload`的区别。

## 变量与继承

关于对象模型，已经在元编程-对象模型篇讲过，此处不作详细说明。

### 类变量

类变量是以`@@`开头命名的变量，在Ruby中，类变量是单例，`在整个祖先链中是唯一的。`

```ruby
class A
  @@a = 1
  def self.a
    @@a
  end

  def self.a=(a)
    @@a = a
  end
end

class B < A
end

B.a = 2
p A.a  # 这里输出的值是2
```

代码中定义了一个类`B`继承自`A`,改变了`B.a`的值，`A.a`的值也跟着变化了。`说明类变量是祖先链中唯一的。`这个专门问题被提出来讲，主要是前段时间面试的时候这个问题的理解上出了问题。之前以为两个类的`self`不一样因此调用的不是同一个类变量对象。

由于`类方法`和`实例方法`都是可以被继承的，因此调用`B.a`的时候，实质上`B`中并不存在`a`方法，因此调用的还是祖先链上游的`A.a`方法，这与外部调用`A.a`实际上效果一样。因此`类变量是可以被继承的类所共享的。`

### 类的实例变量

实例变量是以`@`开头的变量，大部分情况下使用实例变量都是为`类的实例`服务的。然而类本身，也是一个实例对象，因此`类也可以有实例变量`。

```ruby
class A
  @a = 1
  def self.a
    @a
  end

  def self.a=(a)
    @a = a
  end
end

class B < A
end

p A.a # 1
p B.a # nil
B.a = 2
p A.a # 1
p B.a # 2
```

可以看到`@a`对象是不共享的，`A`和`B`两个类都有自己独立的实例变量，因此修改任一个都不会影响另一个。

## 总结

require系与autoload在`同一个命名空间下`只会加载一次，load每调用一次便会重加载一次；require与load是实时加载，autoload是懒加载。类方法和类变量是可以被子类继承的，在本身的类中找不到时会在祖先链中去寻找，而`类的实例变量是不能被继承的`，因此实例变量是不会在祖先链中去查找的。
