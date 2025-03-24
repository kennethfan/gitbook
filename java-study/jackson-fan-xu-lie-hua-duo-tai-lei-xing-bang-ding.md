---
description: Jackson反序列化多态类型绑定
---

# Jackson反序列化多态类型绑定

## 背景

在实际项目中，有时候会遇到，同一个实体对应多个不同的实体类的情况，因此在restful中经常需要定义不同的接口，因为json本身不携带类型信息，但是实际上多个接口从功能层面上来说是一样的，只是返回的结构略有不同，因为这种方式不是很好的处理方式

## 目标

能否将多个接口合并成一个？尽快他们的返回结构有所区别

## 方案

在常用的语言中，其实都是可以的，比如javascript/php/python天然就支持的，最终都是反序列为object/array等

在java中，稍微有些区别，默认也可以反序列为Map及其扩展类(JSONObject, ObjectNode等）但是表意性不好，我们期望的是尽可能的从接口定义上获取到完成的类型信息，反序列化时也尽可能的解析成具体的对象

实际上Jackson本身支持这种操作。

具体操作步骤如下

首先，我们需要定义一个接口，将其实现类信息确定

```java
@JsonTypeInfo(
    use = JsonTypeInfo.Id.NAME,
    include = JsonTypeInfo.As.PROPERTY,
    property = "type"
)
@JsonSubTypes({
    @JsonSubTypes.Type(value = AUserSessionDTO.class, name = UserSession.TYPE_A),
    @JsonSubTypes.Type(value = BUserSessionDTO.class, name = UserSession.TYPE_B),
})
public interface UserSession {

  String TYPE_A = "A";

  String TYPE_B = "B";
} 
```

JsonTypeInfo中property用于确定具体根据json中的哪个字段来确定对应实现类

JsonSubTypes是一组规则，确定了type的取值范围以及和具体实现类之间的映射关系

接下来定义实现类信息

```java
public class AUserSessionDTO implements UserSession {

  private final String type = UserSession.TYPE_A;

  private String propertyA1;

  private String propertyA2;

  private String propertyA3;

  // ...
  private Integer propertyAN;
}
```

```java
public class BUserSessionDTO implements UserSession {

  private final String type = UserSession.TYPE_B;

  private String propertyB1;

  private String propertyB2;

  private String propertyB3;

  // ...
  private Integer propertyBN;
}
```

至此，所有步骤完成，当使用jackson做反序列化时，会自动根据type来反序列化为对应的实现类
