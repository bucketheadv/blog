---
title: Ruby元编程-对象模型
date: 2016-08-29
---
作为一个Ruby开发者，让人又爱又恨的便是元编程了。

### 【前言】元编程是什么
  简单地说，元编程就是对语言本身的进行操作的一种编程手段，最常见的就是`代码生成代码`。对于Ruby这门语言而言，不会元编程，等于不会这门语言，因为这是它的核心能力与魅力。本文是基于阅读《Ruby元编程》后记录的一些自己的理解和看法。

### 元编程示例

#### 【示例1】

```ruby
module Kernel
  def attr_access(*args)
    args.each do |arg|
      define_method(arg) do
         instance_variable_get("@#{arg}")
      end
      define_method("#{arg}=") do |val|
        instance_variable_set("@#{arg}", val)
      end
    end
  end
end

class Student
  attr_access :name, :age
end

stu = Student.new
stu.age = 20
stu.name = 'Rapheal'

p stu.inspect
```
`【示例1】`一个典型元编程的例子，它实现了Ruby中自带的`attr_accessor`相同的功能，作用是动态的为传入的参数（上面代码中是`:name`和`:age`）添加`setter`和`getter`方法（stu.age=xxx为其`setter`方法, stu.age为其`getter`方法）。这样的方法避免了类似Java中的长篇`setter`和`getter`定义。

##【主题】对象模型
Ruby作为一种完全面向对象的编程语言，即使是一个数字、类、甚至一个方法都是一个对象。所谓对象，就是能对它进行一系列操作的一个集合。

### 打开类
对象模型篇第一讲就是`打开类`。在`【示例1】`代码中其实就已经包含了`打开类`的一种具体实现方法。`打开类`，即打开一个已经存在的类或对象，为其`添加新方法`、`修改已存在的方法`或`删除不需要的方法`的一种技术。在`【示例1】`中，`Kernel`是Ruby库中已经存在的一个模块，使用`module Kernel`将其重新打开，并添加了一个新方法`attr_access`。于是`Kernel`模块便在原来的基础上新增了一个方法`attr_access`。

#### 修改一个已经存在的方法`【示例2】`
```ruby
str = "abc"
p str.to_s # 这里会输出"abc"
class String
  def to_s
    "Nothing"
  end
end
str = "abc"
p str.to_s # 这里输出的就是"Nothing"了
```
`String`也是Ruby库自带的类，`to_s`是`String`类已存在的方法，当重新打开它并重写了`to_s`方法之后，原来方法的作用便不复存在了，取而代之的是新方法的作用。（这种修改已经存在的方法又被称为`猴子补丁`）

#### 打开类的利与弊
通过`【示例1】`与`【示例2】`的代码可以知道，打开类技术可以很好的对已经存在的类或方法进行修改，使之更符合个人的使用需求。然而，若不加以思考随意使用，带来的问题也是很严重的。比如`String`类的`to_s`方法，作用就是要返回本身这个字符串，结果被别人修改了这个定义，导致了所有引用这个方法的代码全部失去了它本来的功能与意义。因此在使用打开类定义一个方法时，需要谨慎，`尽量取一个当前不存在的方法名来新定义一个方法`。

#### 对象中有什么
首先，`实例变量`，如`【示例1】`中的`:name`和`:age`，当调用`stu.name = 'rapheal'`之后，`stu`对象便产生了一个实例变量`@name`。`实例变量必须是以【一个@符号】开头的变量名`。这时可以通过调用`stu.instance_variables`来查看已经存在的实例变量，可以看到输出中有`:@name`这一条。

其次，`方法`。通过`stu.methods`可以查看`stu`对象能调用的所有方法。`Ruby对象共享方法，但不共享实例变量，共享的方法被称为【实例方法】。【实例方法】定义在对象的类中，这样可以使得同一类对象可以调用相同的方法。`

`类也是对象。类对象所属的类是Class类。类的方法即为Class类中定义的【实例方法】。`比如，所有类都有一个方法`new`,而`new`方法的定义就在`Class`类中。我们甚至可以简单的认为：`ClassA = Class.new`和`class ClassA; end`是等价的。它们都是在定义一个新的类`ClassA`。

### 方法查找
提到方法查找，那么首先要知道的就是`祖先链`。`祖先链`其实就是记录的一个类的`继承关系`的一个列表，可以通过调用`ancestors`方法来查看。比如`String.ancestors`返回的是`[String, Comparable, Object, Kernel, BasicObject]`，于是我们可以判断，`String`类继承自`Object`，（`Comparable`和`Kernel`是两个`module`,它被包含在了其中的某个类中，也会出现在祖先链中来，此处我们不讨论祖先链中的`module`），`Object`又继承自`BasicObject`。

理解了方法链，再回头来看方法查找。Ruby中的方法查找有个原则叫作`向右，再向上。`比如，有一个`String`类的对象`str`，调用方法`str.test_call_method`，这时Ruby解释器会：
- 1、`【向右】`来到`str`所属的`String`类查看`String`类是否定义了`test_call_method`这个方法，若定义了则直接调用
- 2、`【向上】`否则查看`Comparable`这个`module`中是否定义这个方法（因为祖先链中有这个`module`，并且排在了第二个，即`String`类和`Object`类中间）
- 3、`【向上】`若还未定义，则来到父类`Object`类查找
- 4、重复上述2、3步骤直到`BasicObject`类

上述步骤中，`步骤1`称为`向右`，`步骤2、3`称为`向上`。整个流程中，可以看出，方法查找是`优先向右（所属类）`查找，`再向上（优先是自身包含的模块然后是父类)`查找。因此称为`向右，再向上`原则。

对于类所包含的模块会在方法查找时定义为一个`匿名类`并插入到祖先链中该类的`直接上方`。

### 关于self
在某个特定时刻，一定会有一个指定的对象在执行，这个对象就是`self`对象。最开始接触这个的时候，会有一个误区认为`self`是当前调用方法的`执行者`，然而事实上`self`是当前方法执行的`接收者`。简单说即是，`当前方法调用的结果会传递返回给这个self对象。`

谈到`self`，那么就应该顺便说一下`private`。Ruby中的private是和`self`相关的，在Ruby类的定义的`private`方法是不能被显式调用的。

#### private示例`【示例3】`

```ruby
class A
   def print_self
      self.t_pri
   end
   def print_self_2
      t_pri
   end
   private
      def t_pri
        p "hello world"
      end
end

obj1 = A.new
obj1.print_self_2 # 输出 "hello world"
obj1.print_self     # 报错, NoMethodError: private method 't_pri' called
obj1.t_pri            # 报错，同上
```

从`【示例3】`，可以看出私有方法`t_pri`只能由`self`隐式调用，即`私有方法只能在定义的内部以直接调用方式调用，而不能在任何地方以 xxx.yyyy的方式调用。同时，若没有显式指定方法接收者，那么调用方法的接收都将隐式指定为self对象。`

### 小结

第一章对象模型基本上就这些内容，了解对象基本模型对于以后编写Gem包等扩展有相当重要的作用，尤其是self对象。以后的章节讲到扁平作用域的时候会对self对象有更加深刻的描述，如果这些技能不熟练，要编写Gem扩展真的是举步维艰。
