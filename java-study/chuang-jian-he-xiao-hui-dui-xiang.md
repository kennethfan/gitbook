# 创建和销毁对象

## 考虑用静态工厂方法代替构造器

对于类而言，为了让客户端获取它的每一个实例，最常用的方法就是提供一个公有的构造器。还有一种方法，也应该在每个程序员的工具箱中占有一席之地。类可以提供一个公有的静态工厂方法，它只是一个返回类的实例的静态方法。下面是一个来自Boolean的简单示例。这个方法将boolean基本类型值转换成了一个Boolean对象引用：

```java
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```

注意，静态工厂方法与设计模式中的工厂方法模式不同。

类可以通过静态工厂方法来提供它的客户端，而不是通过构造器。提供静态方法而不是公有的构造器，这样做有几大优势。

### 静态工厂方法与构造器不同的第一大优势在于，它们有名称

如果构造器的参数本身没有确切的描述正被返回的对象，那么具有适当名称的静态工厂会更容易使用，产生的客户端代码也更容易阅读。

### 静态工厂方法与构造器不同的第二大优势在于，不必每次调用它们的时候都创建一个对象

静态工厂方法能够为重复的调用返回相同对象，这样有助于类总能够严格控制在某个时刻哪些实例应该存在。这种类被称作实例受控的类（instance-controlled）。

### 静态工厂方法与构造器不同的第三大优势在于，他们可以返回原返回类型的任何子类型对象

这种灵活性的一种应用是，API可以返回对象，同时又不会使对象的类变成公有的。以这种方式隐藏实现类会使API变的非常简洁。这项技术适用于基于接口的框架，因为在这种框架中，接口为静态工厂方法提供了自然返回类型。接口不能有静态方法，因此按照惯例，接口Type的静态工厂方法被放在一个名为Types的不可实例化的类中。例如，Java Collections Framework的集合接口有32个便利实现，分别提供了不可修改的集合，同步集合等等。

### 静态工厂方法的第四大优势在于，在创建参数化类型实例的时候，它们使代码变得更加简历。

遗憾的是，在调用参数化类的构造器时，即使类型参数很明显，也必须指明。这通常要求你接连两次提供类型参数：

```java
Map<String, List<String>> m = new HashMap<String, List<String>>();
```

随着类型参数变得越来越长，越来越复杂，这一冗长的说明也很快变得痛苦起来。但是有了静态工厂方法，编译器就可以替你找到类型参数。这被称作类型推导（Type inference）。例如，假设HashMap提供了这个静态工厂：

```java
public static <K, V> HashMap<K, V> newInstance() {
    return new HashMap<K, V>();
}
```

你就可以用下面这句简介的代码代替上面这段繁琐的声明：

```java
Map<String, List<String>> m = HashMap.newInstance();
```

### 静态工厂的主要缺点在于，类如果不含公有的或者受保护的构造器，就不能被实例化

对于公有的静态工厂所返回的非公有类，也同样如此。例如，要想将Collections Framework中的任何方便的实现类子类化，这是不可能的。但是这样也许会因祸得福，因为它鼓励程序员使用复合（composition），而不是继承。

### 静态工厂方法的第二个缺点在于，它们与其他的静态方法实际上没有任何区别

## 遇到多个构造器参数时要考虑用构造器

静态工厂和构造器有个共同的局限性：它们都不能很好的扩展到大量的可选参数。

### 重叠构（telescoping constructor）造器模式

```java
// Telescoping constructor pattern - does not scale well!
public class NutritionFacts {
    private final int servingSize; // (ml)    required
    private final int servings;    // (per container)    required
    private final int calories;    //         optional
    private final int fat;         // (g)     optional
    private final int sodium;      // (mg)    optional
    private final int carbohydrate; // (g)     optional
    
    public NutritionFacts(int servingSize, int servings) {
        this(servingSize, servings, 0);
    }
    
    public NutritionFacts(int servingSize, int servings,
            int calories) {
      	this(servingSize, servings, calories, 0);                       
    }
    
    public NutritionFacts(int servingSize, int servings,
            int calories, int fat) {
      	this(servingSize, servings, calories, fat, 0);                    
    }
    
    public NutritionFacts(int servingSize, int servings,
            int calories, int fat, int sodium) {
      	this(servingSize, servings, calories, fat, sodium);                   
    }
  
    public NutritionFacts(int servingSize, int servings,
            int calories, int fat, int sodium, int carbohydrate) {
      	this.servingSize = servingSize;
        this.servings = servings;
        this.calories = calories;
        this.fat = fat;
        this.sodium = sodium;
        this.carbohydrate = carbohydrate;
    }
}
```

### JavaBeans模式

```java
// JavaBeans Pattern -allows inconsistency, mandates mutability
public class NutritionFacts {
    private int servingSize = -1;
    private int servings = -1;
    private int calories = 0;
    private int fat = 0;
    private int sodium = 0;
    private carbohydrate = 0;
  
    public NutritionFacts() {
    }
    
    public void setServingSize(int servingSize) {
        this.servingSize = servingSize;
    }
  
    public void setServings(int servings) {
       this.servings = servings;
    }
  
    public void setCalories(int calories) {
        this.calories = calories;
    }
  
    public void setFat(int fat) {
        this.fat = fat;
    }
    
    public void setSodium(int sodium) {
        this.sodium = sodium;
    }
  
    public void setCarbohydrate(int carbohydrate) {
        this.carbohydrate = carbohydrate;
    }
}
```

### Builder模式

```java
// Builder Pattern
public class NutritionFacts {
    private int servingSize;
    private int servings;
    private int calories;
    private int fat;
    private int sodium;
    private carbohydrate;
  
    public static class Builder {
        // required parameters;
        private final int servingSize;
        private final int servings;
      
        // optional parameters;
        private int calories = 0;
        private int fat = 0;
        private int sodium = 0;
        private int carbohydrate = 0;
      
        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.serving = servings;
        }
      
        public Builder calories(int calories) {
            this.calories = calories;
            return this;
        }
      
        public Builder fat(int fat) {
            this.fat = fat;
            return this;
        }
      
        public Builder sodium(int sodium) {
            this.sodium = sodium;
            return this;
        }
        
        public Builder carbohydrate(int carbohydrate) {
            this.carbohydrate = carbohydrate;
        }
      
        public NutritionFacts build() {
            return new NutritionFacts(this);
        }
    }
  
    private NutritionFacts(Builder builder) {
        servingSize = builder.servingSize;
        servings = builder.servings;
        calories = builder.calories;
        fat = builder.fat;
        sodium = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }
}
```

## 用私有构造器或者枚举类型强化Singleton属性

Singleton指仅仅被实例化一次的类。在Java 1.5发行版本之前，实现Singleton有两种方法。

### 第一种：公有静态域有个final域

```java
// Singleton with public final field
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() {
        // do sth;
    }
    
    public void leaveTheBuilding() {
        // do sth;
    }
}
```

### 第二种：公有的成员是个静态工厂方法

```java
// Singleton with static factory
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() {
        // do sth;
    }
  
    public static Elvis getInstance() {
        return INSTANCE;
    }
  
    public void leaveTheBuilding() {
        // do sth;
    }
}
```

从Java 1.5发行版本起，实现Singleton还有第三种方法。

### 第三种：写一个包含单个元素的枚举类型

```java
// Enum singleton - the preferred approach
public enum Elvis {
    INSTANCE;
  
  	public void leaveTheBuilding() {
        // do sth;
  	}
}
```

## 通过私有构造器强化不和实例化的能力

有些工具类不希望被实例化，实例对它没有任何意义。然而，在缺少显示构造器的情况下，编译器回自动提供一个公有的、无参的缺省构造器（default constructor）。对于用户而言，这个构造器与其他构造器没有任何区别。**企图通过将类做成抽象类来强制该类不可被实例化，这是行不通的**。该类可以被子类化，并且该子类也可以被实例化。这样做甚至会误导用户，以为这种类是专门为了集成而设计的。为类生成一个私有的构造器就可以保证它不能被实例化了

```java
// Noninstantiable utility class
public class UtilityClass {
    // Suppress default constructor for noninstantiablity
    private UtilityClass() {
        throw new AssertionError();
    }
}
```

## 避免创建不必要的对象

一般来说，最好能重用对象而不是在每次需要的时候就创建一个相同功能的对象。重用方式既快速又流行。如果对象是不可变得（immutable），它就是始终可以被重用。

## 消除过期的对象引用

消除过期的对象引用主要是方便内存回收。

```java
// Can you spot the "memory leak"
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INTIAL_CAPACITY = 16;
  
    public Stack() {
        elements = new Object[DEFAULT_INTIAL_CAPACITY];
    }
     
    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }
   
    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        return elements[--size];
    }
   
    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

这段程序中并没有很明显的错误。无论如何测试，它都回成功地通过每一项测试，但是这个程序中隐藏着一个问题。不严格的讲，这段程序有一个"内存泄露"，随着垃圾回收器活动的增加，或者由于内存占用的不断增加，程序性能的降低会逐渐表现出来。

在支持垃圾回收的语言中，内存泄露是很隐蔽的。如果一个对象引用被无意识地保留起来了，那么，垃圾回收机制不近不会处理这个对象，而且也不会处理这个对象所引用的所有其他对象。这类问题的修复方法很简单：一旦对象引用已经过期，只需清空这些引用即可。

```java
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null;
    return result;
}
```

一般而言，**只要类是自己管理内存，程序员就应该警惕内存泄露问题**。一旦元素被释放掉，则该元素中包含的任何对象引用都应该被清空。

**内存泄露的另一个常见来源是缓存**。一旦你把对象引用放到缓存中，它就很容易被遗忘掉，从而使得它不再有用之后很长一段时间内仍然保留在缓存中。

**内存泄露的第三个常见来源是监听器和其他回调**。如果你实现了一个API，客户端在这个API中注册回调，去没有显式的取消注册，那么除非你采取某些动作，否则他们就会积聚。确保回调立即被当做垃圾回收的最佳方法是只保存它们的弱引用（weak reference）。

## 避免使用终结方法

**终结方法通常是不可预测的，也是很危险的，一般情况下是不必要的**。使用终结方法会导致行为不稳定、降低性能，以及可移植性问题。

**终结方法的缺点在于不能保证会被及时地执行**。从一个对象变得不可达开始，到它的终结方法被执行，所花费的这段时间是任意长的。

**Java语言规范不仅不保证终结方法会被及时地执行，而且根本就不保证它们会被执行**。不应该依赖终结方法来更新重要的持久状态。例如，依赖终结方法来释放共享资源上的永久锁。

还有一点：**终结方法有一个非常严重的性能损失**。
