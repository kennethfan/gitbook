# 对于所有对象都通用的方法

## 覆盖equals时请遵守通用约定

不覆盖equals方法的几种情况

* 类的每个实例本质上都是唯一的；例如Thead。
* 不关心类是否提供了"逻辑相等（logical equality）"的测试功能；例如Rondom。
* 超类已经覆盖了equals，从超类继承过来的行为对于子类也是合适的；例如大多数Set都实现从AbtractSet继承的equals实现。
* 类是私有的或者包级私有的，可以确定它的equals方法永远不会调用。

什么时候应该覆盖Object.equals呢？**如果类具有自己特有的"逻辑相等"概念，而且超类还没有覆盖equals以实现期望的行为，这时候我们就需要覆盖equals方法**。

覆盖eqauls方法需要遵守的通用约定：

* 自反性（reflexive）。对于任何非null的引用值x，x.equals(x)必须返回true
* 对称性（symmetric）。对于任何非null的引用值x和y，当且仅当y.equas(x)返回true时，x.equals(y)必须返回true。
* 传递性（transitive）。对于任何非null的引用值x、y和z，如果x.equals(y)返回true且y.equals(z)也返回true，那么x.equals(z)也必须返回true。
* 一致性（consistent）。对于任何非null的引用值x和y，只要equals的比较操作在对象所用的信息没有被修改，多次调用x.equals(y)就会一致地返回true或者false。

一些诀窍：

* 使用==操作符检查"参数是否为这个对象的引用"。如果是则返回 true。
* 使用instanceof操作符检查"参数是否为正确的类型"。如果不是则返回 false。
* 把参数转换成正确的类型。
* 对于该类种的每个"关键（significant）"域，检查参数中的域是否与该对象中对应的域相匹配。如果这些测试全部成功，则返回true；否则返回false。
* 编写完成了equals方法之后，应该确认三个问题：是否对称的、传递的、一致的？

一些告诫：

* 覆盖equals时总要覆盖hashCode
* 不要企图让equals方法过于智能
* 不要将equals声明中的Object对象替换为其他的类型。

## 覆盖equals时总要覆盖hashCode

相等的对象必须具有相等的散列码（hash code）

## 始终要覆盖toString

提供好的toString实现可以使类用起来更加舒适。

## 谨慎的覆盖clone

考虑使用拷贝构造器替代clone

## 考虑实现Comparable接口

约定sng（表达式）会根据(expression)的值为负值、零和正直分别返回-1、0或1。

实现comparable的一些约定

* 必须确保所有的x和y都满足sgn(x.compareTo(y)) == -sgn(y.compareTo(x))
* 比较关系是可传递的：（x.compareTo(y) >0 && y.compareTo(x) > 0）则x.compareTo(z) > 0
* 确保x.compareTo(y) == 0则sgn(x.compareTo(z)) == sgn(y.compareTo(z))
* 强烈建议(x.compareTo(y) == 0) == (x.equals(y))
