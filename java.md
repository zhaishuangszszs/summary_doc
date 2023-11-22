## java简介
[一文彻底搞懂令人疑惑的Java和JDK的版本命名！](https://blog.csdn.net/sinat_33921105/article/details/117513645)
* Java Oracle 公司的产品
* JavaSE（J2SE）（Java2 Platform Standard Edition，java平台标准版）
* JavaEE(J2EE)(Java 2 Platform,Enterprise Edition，java平台企业版)
* JavaME(J2ME)(Java 2 Platform Micro Edition，java平台微型版)。


## 文件结构
* 1.**文件名与类名**
[一个Java文件可以有多个类（外部类、内部类）](https://blog.csdn.net/qq_43783527/article/details/125831011   )
> 一个".java"源文件中是否可以包括多个类（不是内部类）？有什么限制？
* 可以有多个类，但只能有一个public的类，并且**public的类名必须与文件名相一致**。
* 编译单元内完全不带public类也是可能的。在这种情况下，可以随意对文件命名。
* 2.**idea文件结构**
project->module->package->java编译单元->class
* 如果一个类定义在某个包中，那么 package 语句应该在源文件的首行。
* 包名一般小写，类名首字母大写(以示区分)

### 类与对象
* java,c++ 当父类没有定义默认构造函数，定义了了自定义的构造函数时，子类定义构造函数时需要使用父类定义的构造函数，否则编译报错。


## java特性注意
* [主要特性](https://www.runoob.com/java/java-intro.html)
* 没有**操作符重载**(有**方法的重载**)、**多继承**、java 语言**不使用指针**，而是引用。
* 只支持类之间的单继承，但支持接口之间的多继承，并支持类与接口之间的实现机制（关键字为 implements）。Java 语言全面支持动态绑定，而 C++语言只对虚函数使用动态绑定。
* 在一些其它语言中方法指过程和函数。一个返回非void类型返回值的方法称为函数；一个返回void类型返回值的方法叫做过程。

