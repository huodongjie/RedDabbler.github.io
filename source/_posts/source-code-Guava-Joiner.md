---
title: 源码Guava之Joiner
date: 2017-08-04 20:20:49
permalink: Source-code-Guava-Joiner
tags: [Guava]
categories:
- 源码
- Guava
---
Joiner 可用于将分隔符 和 集合 拼接，不用写大段类似的循环逻辑，极大的简洁了代码。为实现Appendable的接口的对象和实现Iterable接口的对象提供了用分隔符拼接的解决方案。

<!--more-->

[toc]
### Joiner的使用

#### 集合元素拼接
- 用字符串拼接，返回字符串
``` java
//1#2#3#4
Joiner.on('#').join(Lists.newArrayList('1','2','3','4'));   
```
- 用StringBuilder 拼接
```java
Joiner.on('#').appendTo(new StringBuilder(),Lists.newArrayList('1','2','3','4'));
```
- 其实只要实现了Appendable的类都可以如上使用,传入什么类型，就返回什么类型
```java
Joiner.on('#').appendTo(new FileWriter("app"),Lists.newArrayList('1','2','3','4'));
```
-  集合元素可以是个对象，只不过返回的字符串，若要指定返回的格式，需要重写集合元素toString()方法
```java
Joiner.on('#').appendTo(new StringBuilder(),Lists.newArrayList( obj1，obj2));
```
- 其实集合只是接口Iterable的实现，只要实现Iterable的类都可以拼接
```java
Joiner.on('#').appendTo(Iterable Iterable)
```
- 集合元素null处理
```java
Joiner.on('-').skipNulls().join(1, null, 3);//1-3
Joiner.on('-').useForNull("None").join(1, null, 3);//1-None-3
```
#### Map 的key，value拼接
```java
//如 map('"1,"2"),('"3,"2")  ==> 1=2#3=2
Joiner.on('#').withKeyValueSeparator('=').appendTo(Map map);

```
### 源码解析

Joiner的构造器是私有的，调用静态方法on，才能获取Joiner对象，通过join或者appendTo的实例方法来连接集合元素。MapJoiner与其类似。

- 最底层的实现
```java
public <A extends Appendable> A appendTo(A appendable, Iterator<?> parts) throws IOException {
    checkNotNull(appendable);
    if (parts.hasNext()) {
      appendable.append(toString(parts.next()));
      while (parts.hasNext()) {
        appendable.append(separator);
        appendable.append(toString(parts.next()));
      }
    }
    return appendable;
  }
```
其他的方法可以说都由这个方法抽象及重载，比如StringBuilder 是Appendable的实现，集合或数组可以认为是Iterable的实现，因此尽可能面向抽象编程，让代码看起来简洁。

这个代码实现 巧妙的利用迭代器 hasNext 的逻辑判断，来获知迭代器中是否还有元素，在进入while循环，而不是利用Iterable的特点，for循环遍历，再在最后补救去掉多余的separator。遇到类似的处理，可以考虑参考下。

```java
CharSequence toString(Object part) {
  checkNotNull(part); // checkNotNull for GWT (do not optimize).
  return (part instanceof CharSequence) ? (CharSequence) part : part.toString();
}
```
这个是toString方法的实现，返回的结果也没有直接用String, 而是String的接口CharSequence，这样就巧妙的使用了Appendable的原生接口，append(CharSequence c)，这样的话任何实现CharSequence的对象，可以直接返回对象本身（其实最终还是会调用toString方法）

- 对空的处理
  Joiner对Iterable元素进行拼接时，遇到**集合内元素**为null的情况下，若不处理方法会直接抛出NPE，Joiner 提供了skipNulls和useForNull 方法来处理避免NPE。

Joiner对这种null的实现方案返回一个新的Joiner子类，子类重写已实现的底层appendTo方法，而不是另外实现一套专门处理空的appendTo方法，如下：
```java
@CheckReturnValue
  public Joiner skipNulls() {
    return new Joiner(this) {
      @Override
      public <A extends Appendable> A appendTo(A appendable, Iterator<?> parts) throws IOException {
        checkNotNull(appendable, "appendable");
        checkNotNull(parts, "parts");
        while (parts.hasNext()) {
          Object part = parts.next();
          if (part != null) {
            appendable.append(Joiner.this.toString(part));
            break;
          }
        }
        while (parts.hasNext()) {
          Object part = parts.next();
          if (part != null) {
            appendable.append(separator);
            appendable.append(Joiner.this.toString(part));
          }
        }
        return appendable;
      }

      @Override
      public Joiner useForNull(String nullText) {
        throw new UnsupportedOperationException("already specified skipNulls");
      }

      @Override
      public MapJoiner withKeyValueSeparator(String kvs) {
        throw new UnsupportedOperationException("can't use .skipNulls() with maps");
      }
    };
  }
  ```
  skipNulls返回的Joiner子类，除了重写要实现的appendTo方法外，还重写了**子类不支持的方法**, skipNulls 和userForNull都是处理null的方案，不存在两种方法同时存在的情况，直接覆盖方法。
  useForNull 方法类似，具体可以看下源码。

  - MapJoiner
  Joiner有个public的静态内部类MapJoiner, 专门处理键值对的情况，
  下面这方法是它最底层的方法，与Joiner类似
```java
   @Beta
   public <A extends Appendable> A appendTo(A appendable, Iterator<? extends Entry<?, ?>> parts)
       throws IOException {
     checkNotNull(appendable);
     if (parts.hasNext()) {
       Entry<?, ?> entry = parts.next();
       appendable.append(joiner.toString(entry.getKey()));
       appendable.append(keyValueSeparator);
       appendable.append(joiner.toString(entry.getValue()));
       while (parts.hasNext()) {
         appendable.append(joiner.separator);
         Entry<?, ?> e = parts.next();
         appendable.append(joiner.toString(e.getKey()));
         appendable.append(keyValueSeparator);
         appendable.append(joiner.toString(e.getValue()));
       }
     }
     return appendable;
   }
```

[参考文档](http://www.importnew.com/15221.html)
