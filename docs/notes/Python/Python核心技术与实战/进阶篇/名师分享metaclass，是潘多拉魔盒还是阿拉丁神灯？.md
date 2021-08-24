# [名师分享] metaclass，是潘多拉魔盒还是阿拉丁神灯？

你好，我是蔡元楠，是极客时间《大规模数据处理实战》的作者。很高兴受邀来我们专栏分享，今天我分享的主题是：metaclass，是潘多拉魔盒还是阿拉丁神灯？

Python 中有很多黑魔法，比如今天我将分享的 metaclass。我认识许多人，对于这些语言特性有两种极端的观点。

- 一种人觉得这些语言特性太牛逼了，简直是无所不能的阿拉丁神灯，必须找机会用上才能显示自己的 Python 实力。
- 另一种观点则是认为这些语言特性太危险了，会蛊惑人心去滥用，一旦打开就会释放“恶魔”，让整个代码库变得难以维护。

其实这两种看法都有道理，却又都浅尝辄止。今天，我就带你来看看，metaclass 到底是潘多拉魔盒还是阿拉丁神灯？

市面上的很多中文书，都把 metaclass 译为“元类”。我一直认为这个翻译很糟糕，所以也不想在这里称 metaclass 为元类。因为如果仅从字面理解，“元”是“本源”“基本”的意思，“元类”会让人以为是“基本类”。难道 Python 的 metaclass，指的是 Python 2 的 Object 吗？这就让人一头雾水了。

事实上，meta-class 的 meta 这个词根，起源于希腊语词汇 meta，包含下面两种意思：

1. “Beyond”，例如技术词汇 metadata，意思是描述数据的超越数据；
2. “Change”，例如技术词汇 metamorphosis，意思是改变的形态。

metaclass，一如其名，实际上同时包含了“超越类”和“变形类”的含义，完全不是“基本类”的意思。所以，要深入理解 metaclass，我们就要围绕它的<strong>超越变形</strong>特性。接下来，我将为你展开 metaclass 的超越变形能力，讲清楚 metaclass 究竟有什么用？怎么应用？Python 语言设计层面是如何实现 metaclass 的 ？以及使用 metaclass 的风险。

## metaclass 的超越变形特性有什么用？

<a href="https://pyyaml.org/wiki/PyYAMLDocumentation">YAML</a>是一个家喻户晓的 Python 工具，可以方便地序列化 / 逆序列化结构数据。YAMLObject 的一个<strong>超越变形能力</strong>，就是它的任意子类支持序列化和反序列化（serialization &amp; deserialization）。比如说下面这段代码：

```python
class Monster(yaml.YAMLObject):
  yaml_tag = u'!Monster'
  def __init__(self, name, hp, ac, attacks):
    self.name = name
    self.hp = hp
    self.ac = ac
    self.attacks = attacks
  def __repr__(self):
    return "%s(name=%r, hp=%r, ac=%r, attacks=%r)" % (
       self.__class__.__name__, self.name, self.hp, self.ac,      
       self.attacks)
 
yaml.load("""
--- !Monster
name: Cave spider
hp: [2,6]    # 2d6
ac: 16
attacks: [BITE, HURT]
""")
 
Monster(name='Cave spider', hp=[2, 6], ac=16, attacks=['BITE', 'HURT'])
 
print yaml.dump(Monster(
    name='Cave lizard', hp=[3,6], ac=16, attacks=['BITE','HURT']))
 
# 输出
!Monster
ac: 16
attacks: [BITE, HURT]
hp: [3, 6]
name: Cave lizard

```

这里 YAMLObject 的特异功能体现在哪里呢？

你看，调用统一的 yaml.load()，就能把任意一个 yaml 序列载入成一个 Python Object；而调用统一的 yaml.dump()，就能把一个 YAMLObject 子类序列化。对于 load() 和 dump() 的使用者来说，他们完全不需要提前知道任何类型信息，这让超动态配置编程成了可能。在我的实战经验中，许多大型项目都需要应用这种超动态配置的理念。

比方说，在一个智能语音助手的大型项目中，我们有 1 万个语音对话场景，每一个场景都是不同团队开发的。作为智能语音助手的核心团队成员，我不可能去了解每个子场景的实现细节。

在动态配置实验不同场景时，经常是今天我要实验场景 A 和 B 的配置，明天实验 B 和 C 的配置，光配置文件就有几万行量级，工作量不可谓不小。而应用这样的动态配置理念，我就可以让引擎根据我的文本配置文件，动态加载所需要的 Python 类。

对于 YAML 的使用者，这一点也很方便，你只要简单地继承 yaml.YAMLObject，就能让你的 Python Object 具有序列化和逆序列化能力。是不是相比普通 Python 类，有一点“变态”，有一点“超越”？

事实上，我在 Google 见过很多 Python 开发者，发现能深入解释 YAML 这种设计模式优点的人，大概只有 10%。而能知道类似 YAML 的这种动态序列化 / 逆序列化功能正是用 metaclass 实现的人，更是凤毛麟角，可能只有 1% 了。

## metaclass 的超越变形特性怎么用？

刚刚提到，估计只有 1% 的 Python 开发者，知道 YAML 的动态序列化 / 逆序列化是由 metaclass 实现的。如果你追问，YAML 怎样用 metaclass 实现动态序列化 / 逆序列化功能，可能只有 0.1% 的人能说得出一二了。

因为篇幅原因，我们这里只看 YAMLObject 的 load() 功能。简单来说，我们需要一个全局的注册器，让 YAML 知道，序列化文本中的 <code>!Monster</code> 需要载入成 Monster 这个 Python 类型。

一个很自然的想法就是，那我们建立一个全局变量叫 registry，把所有需要逆序列化的 YAMLObject，都注册进去。比如下面这样：

```python
registry = {}
 
def add_constructor(target_class):
    registry[target_class.yaml_tag] = target_class

```

然后，在 Monster 类定义后面加上下面这行代码：

```python
add_constructor(Monster)

```

但这样的缺点也很明显，对于 YAML 的使用者来说，每一个 YAML 的可逆序列化的类 Foo 定义后，都需要加上一句话，<code>add_constructor(Foo)</code>。这无疑给开发者增加了麻烦，也更容易出错，毕竟开发者很容易忘了这一点。

那么，更优的实现方式是什么样呢？如果你看过 YAML 的源码，就会发现，正是 metaclass 解决了这个问题。

```python
# Python 2/3 相同部分
class YAMLObjectMetaclass(type):
  def __init__(cls, name, bases, kwds):
    super(YAMLObjectMetaclass, cls).__init__(name, bases, kwds)
    if 'yaml_tag' in kwds and kwds['yaml_tag'] is not None:
      cls.yaml_loader.add_constructor(cls.yaml_tag, cls.from_yaml)
  # 省略其余定义
 
# Python 3
class YAMLObject(metaclass=YAMLObjectMetaclass):
  yaml_loader = Loader
  # 省略其余定义
 
# Python 2
class YAMLObject(object):
  __metaclass__ = YAMLObjectMetaclass
  yaml_loader = Loader
  # 省略其余定义

```

你可以发现，YAMLObject 把 metaclass 都声明成了 YAMLObjectMetaclass，尽管声明方式在 Python 2 和 3 中略有不同。在 YAMLObjectMetaclass 中， 下面这行代码就是魔法发生的地方：

```python
cls.yaml_loader.add_constructor(cls.yaml_tag, cls.from_yaml) 

```

YAML 应用 metaclass，拦截了所有 YAMLObject 子类的定义。也就说说，在你定义任何 YAMLObject 子类时，Python 会强行插入运行下面这段代码，把我们之前想要的<code>add_constructor(Foo)</code>给自动加上。

```python
cls.yaml_loader.add_constructor(cls.yaml_tag, cls.from_yaml)

```

所以 YAML 的使用者，无需自己去手写<code>add_constructor(Foo)</code> 。怎么样，是不是其实并不复杂？

看到这里，我们已经掌握了 metaclass 的使用方法，超越了世界上 99.9% 的 Python 开发者。更进一步，如果你能够深入理解，Python 的语言设计层面是怎样实现 metaclass 的，你就是世间罕见的“Python 大师”了。

## <strong>Python 底层语言设计层面是如何实现 metaclass 的？</strong>

刚才我们提到，metaclass 能够拦截 Python 类的定义。它是怎么做到的？

要理解 metaclass 的底层原理，你需要深入理解 Python 类型模型。下面，我将分三点来说明。

可能会让你惊讶，事实上，类本身不过是一个名为 type 类的实例。在 Python 的类型世界里，type 这个类就是造物的上帝。这可以在代码中验证：

```python
# Python 3 和 Python 2 类似
class MyClass:
  pass
 
instance = MyClass()
 
type(instance)
# 输出
<class '__main__.C'>
 
type(MyClass)
# 输出
<class 'type'>

```

你可以看到，instance 是 MyClass 的实例，而 MyClass 不过是“上帝”type 的实例。

当我们定义一个类的语句结束时，真正发生的情况，是 Python 调用 type 的<code>__call__</code>运算符。简单来说，当你定义一个类时，写成下面这样时：

```python
class MyClass:
  data = 1

```

Python 真正执行的是下面这段代码：

```python
class = type(classname, superclasses, attributedict)

```

这里等号右边的<code>type(classname, superclasses, attributedict)</code>，就是 type 的<code>__call__</code>运算符重载，它会进一步调用：

```python
type.__new__(typeclass, classname, superclasses, attributedict)
type.__init__(class, classname, superclasses, attributedict)

```

当然，这一切都可以通过代码验证，比如下面这段代码示例：

```python
class MyClass:
  data = 1
  
instance = MyClass()
MyClass, instance
# 输出
(__main__.MyClass, <__main__.MyClass instance at 0x7fe4f0b00ab8>)
instance.data
# 输出
1
 
MyClass = type('MyClass', (), {'data': 1})
instance = MyClass()
MyClass, instance
# 输出
(__main__.MyClass, <__main__.MyClass at 0x7fe4f0aea5d0>)
 
instance.data
# 输出
1

```

由此可见，正常的 MyClass 定义，和你手工去调用 type 运算符的结果是完全一样的。

其实，理解了以上几点，我们就会明白，正是 Python 的类创建机制，给了 metaclass 大展身手的机会。

一旦你把一个类型 MyClass 的 metaclass 设置成 MyMeta，MyClass 就不再由原生的 type 创建，而是会调用 MyMeta 的<code>\_\_call\_\_</code>运算符重载。

```python
class = type(classname, superclasses, attributedict) 
# 变为了
class = MyMeta(classname, superclasses, attributedict)

```

所以，我们才能在上面 YAML 的例子中，利用 YAMLObjectMetaclass 的<code>\_\_init\_\_</code>方法，为所有 YAMLObject 子类偷偷执行<code>add_constructor()</code>。

## <strong>使用 metaclass 的风险</strong>

前面的篇幅，我都是在讲 metaclass 的原理和优点。的的确确，只有深入理解 metaclass 的本质，你才能用好 metaclass。而不幸的是，正如我开头所说，深入理解 metaclass 的 Python 开发者，只占了 0.1% 不到。

不过，凡事有利必有弊，尤其是 metaclass 这样“逆天”的存在。正如你所看到的那样，metaclass 会"扭曲变形"正常的 Python 类型模型。所以，如果使用不慎，对于整个代码库造成的风险是不可估量的。

换句话说，metaclass 仅仅是给小部分 Python 开发者，在开发框架层面的 Python 库时使用的。而在应用层，metaclass 往往不是很好的选择。

也正因为这样，据我所知，在很多硅谷一线大厂，使用 Python metaclass 需要特例特批。

## 总结

这节课，我们通过解读 YAML 的源码，围绕 metaclass 的设计本意“超越变形”，解析了 metaclass 的使用场景和使用方法。接着，我们又进一步深入到 Python 语言设计层面，搞明白了 metaclass 的实现机制。

正如我取的标题那样，metaclass 是 Python 黑魔法级别的语言特性。天堂和地狱只有一步之遥，你使用好 metaclass，可以实现像 YAML 那样神奇的特性；而使用不好，可能就会打开潘多拉魔盒了。

所以，今天的内容，一方面是帮助有需要的同学，深入理解 metaclass，更好地掌握和应用；另一方面，也是对初学者的科普和警告：不要轻易尝试 mateclass。

## 思考题

学完了上节课的 Python 装饰器和这节课的 metaclass，你知道了，它们都能干预正常的 Python 类型机制。那么，你觉得装饰器和 metaclass 有什么区别呢？欢迎留言和我讨论。
