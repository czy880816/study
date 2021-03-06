在程序世界中，经常会存在现有程序无法直接使用，需要做适当的变换后才能使用的情况。这种用于填补“现有程序”和“所需程序”之间差异的设计模式就是Adapter模式

Adapter模式也被称为Wrapper模式。Wrapper有“包装器”的意思，就像用精美的包装纸将普通商品包装成礼物那样，替我们把某样东西包起来，使其能够用于其他用途的东西就被称为“包装器”或是“适配器”

Adapter模式有以下两种：

- **类适配器模式（使用继承的适配器）**

  如果要适配的是一个接口，则可以使用该方式

- **对象适配器模式（使用委托的适配器）**

  如果要适配的是一个类或者抽象类，则可以使用该方

**Adapter模式中的相关角色：**

- Target（对象）

  该角色定义所需的方法。

- Client（请求者）

  该角色使用Target角色所定义的方法进行具体处理

- Adaptee（被适配者）

  Adaptee是一个持有既定方法的类，适配器模式的任务就是将其与Target适配。如果Adaptee角色中的方法与Target角色的方法相同，就不需要接下来的Adapter角色了

- Adapter（适配者）

  使用Adaptee角色的方法来满足Target角色的需求，这是Adapter模式的目的，也是Adapter角色的作用

**什么时候使用Adapter模式？**

很多时候，我们并非从零开始编程，经常会用到现有的类。特别是当现有的类已经被充分测试过了，Bug很少，而且已经被用于其它软件之中时，我们更愿意将这些类作为组件重复利用。这就是Adapter模式出现的目的，Adapter模式会对现有的类进行适配，生成新的类。通过该模式可以很方便地创建我们所需要的方法群