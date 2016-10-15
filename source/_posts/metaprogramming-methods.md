---
title: Ruby元编程-方法
date: 2016-08-29
---
Ruby是一门动态语言，动态创建与调用方法是其中一个体现。

### 动态方法
#### 动态调用方法（动态派发）
动态调用方法，是指在代码中不通过硬编码而是在程序运行时自动去决定要调用的方法的一种行为。

#### 示例代码1
```ruby
class Student
   attr_accessor :name, :age, :birthday
   def initialize(args = {})
     name = args[:name]
     age    = args[:age]
     birthday = args[:birthday]
   end
end
```
`【示例代码1】`中`initialize`方法中，给三个字段赋值的方式就是一种典型的硬编码方式，假如这三个字段的名称有改动，抑或添加、去掉字段的时候，不得不同时修改这个方法。为了避免这种情况，这里可以考虑使用动态调用的方式来重构它。

#### 示例代码2
```ruby
class Student
  attr_accessor :name, :age, :birthday
  def initialize(args = {})
    args.each do |key, value|
       method_name = "#{key}="
       self.send("#{key}=", value) if self.respond_to?(method_name)
    end
  end
end
```
通过`【示例代码2】`的重构，但凡`attr_accessor`后面的字段有变动时，`initialize`方法都会自动进行适配。那么实现的原理是什么呢？

在Ruby中，方法调用其实是向一个对象发送了一条消息，当接收方接收消息后，会在对象的祖先链中去寻找这个方法，找到之后调用它并返回给`self`对象（[详细见【对象模型篇】](http://www.jianshu.com/p/324fc76e68a2)）。也就是说，当调用`str.method`的时候，本质上就是发送了一条方法调用的消息，接收者是`str`对象，它等价于`str.send(:method)`。因此示例代码便很好理解了，它是将`args`这个hash中的值进行遍历，动态调用`attr_accessor`生成的`setter`和`getter`方法。但有一个问题，如果参数中有在`attr_accessor`未定义的字段怎么办？比如`Student.new({ year: 2016 })`，`year`字段是未在`attr_accessor`中定义的，如果调用`self.year = `这个方法，是会拋异常的。所以这里添加了`respond_to?`来判断这个方法是否是存在的，存在再对它进行调用赋值。

#### 动态定义方法
关于动态定义方法，其实在第一章[对象模型篇](http://www.jianshu.com/p/324fc76e68a2)的`【示例代码1】`已经在使用了，就是对`define_method`的使用。在此基础之上，此处实现一个更加具有可用性的案例：

#### 示例代码3
```ruby
module Kernel
   def attr_access(*args)
      args.each do |arg|
         define_method(arg) do
            instance_variable_get("@#{arg}")
         end
         define_method("#{arg}=") do |value|
            instance_variable_set("@{arg}=", value)
         end
      end
   end

  def cattr_access(*args)
     args.each do |arg|
        define_singleton_method(arg) do
           self.class_variable_get("@@#{arg}")
        end
        define_singleton_method("#{arg}=") do |value|
           self.class_variable_set("@@#{arg}", value)
        end
     end
  end
end

class A
  cattr_access :a
end
A.a = 1
p A.a  # 输出1
```
此处不再说明`define_method`和`attr_access`的使用，重点说明一下`define_singleton_method`和`cattr_access`的实现。

`define_singleton_method`和`define_method`的区别是，前者定义的是`单例方法`（这里可称为类方法），后者定义的是`实例方法`。从用法来看，`cattr_access`声明的变量直接在类（这里是`A`）上调用，而`attr_access`声明的变量需要在`A`类对象实例化（`A.new`）之后调用。同理，`class_variable_set`和`class_variable_get`定义的是单例变量（这里指类变量），而`instance_variable_set`和`instance_variable_get`定义的是实例变量。由于Ruby的语法约定，以`@`开头的为实例变量，以`@@`开头的为类变量，因此，在定义变量时尤其要注意变量的全名，否则会拋异常。

#### 幽灵方法

还记得之前在方法查找中，如果找不到方法时，会触发一个`NoMethodError`的异常拋出。然而它来源于向对象发送了一个消息调用了一个方法叫做`method_missing`。

假如对一个`String`类对象`str`调用`test_method_a`，即`str.test_method_a`，由于这个方法未定义，因此在祖先链中找不到这个方法。此时会发送一个消息`str.send(:method_missing, :test_method_a)`，从而拋出`NoMethodError`的异常。也就是说，当找不到要调用的方法时，会自动触发调用`method_missing`方法。那么如果重写了某个类的`method_missing`方法会是什么样的结果呢？

#### 示例代码4
```ruby
class XmlGen
  def method_missing(name, *args, &block)
     if %W(html head title body).include?(name.to_s)
        define_singleton_method(name) do |arg = nil, &blk|
           str  = "<#{name}>"
           str += arg if arg
           str += blk.call if blk
           str += "</#{name}>"
           str
        end
        self.send(name, *args, &block)
     end
  end
end

xml = XmlGen.new
str = xml.html do
   xml.head do
      xml.title "Test"
   end
end
p str  # 输出<html><head><title>Test</title></head></html>
```
由于在`method_missing`中对调用方法的名字做了限制，必须是`html`、`head`、`title`、`body`其中之一才会生成代码，因此无需担心其它额外正常调用不存在方法的时候不能正常拋出`NoMethodError`异常的情况。由于在调用不存在的方法时就会调用`method_missing`这个方法，因此如果要重写这个方法一定要格外小心，`能力越大，责任越大`。

#### 幽灵方法与普通动态方法的优劣
普通动态方法是指，在类初始化时便使用`define_method`等手段将需要的所有方法定义好。幽灵方法本质是在调用时，如果发现不存在方法时，那么即时定义这个方法并产生一次调用，从示例可以看出幽灵方法在定义方法时也是调用的`define_method`等行为来定义动态方法。与普通定义动态方法的区别是，如果一个对象永远没有调用一个方法，那么这个方法永远不会被定义，只有调用过一次时它才会被定义，因此使用幽灵方法时，对象所占用的内存空间比普通动态方法要少，反之付出的代价是第一次在祖先链中查找该方法的时间变长。`这可以认为是一种以时间换取空间的策略。`

### 动态代理
动态代理的原理是，对`a`对象的操作转移到`b`对象上来，Ruby中使用`delegate`库来实现动态代理。

#### 示例代码5
```ruby
class UserProfile
    def initialize(name)
       @name = name
    end
    def hello
       "#{@name} says hello."
    end
end

class User < DelegateClass(UserProfile)
   def initialize(user_profile)
      super(user_profile)
   end
end
user_profile = UserProfile.new("Rapheal")
user = User.new(user_profile)
p user.hello  # 输出 "Rapheal says hello."
```

#### 关于respond_to?
`respond_to?`是Ruby中用于判断一个方法是否存在的一个方法。比如，`Class.respond_to?(:new) #返回true`，说明`Class`这个类可以调用`new`方法。这个方法通常与`define_method`、`method_missing`等方法一起使用，与`method_missing`一样，不到万不得已，不要修改这个方法。

### 小结

本章主要讲了使用define_method来定义动态方法，使用method_missing来处理`NoMethodError`的情况。依然是那句话，能力越大，责任越大。如果能加以善用，那么这些特征能使的代码的灵活度越来越高，反之只能使之晦涩难懂甚至导致难以追踪的BUG。慎之！慎之！
