---
title: ArrayList.subList
categories: 
- 源码分析
---

首发在公众号！！！

# 前言

《阿里巴巴Java开发手册》中提出了以下几点建议和规则：

**「规则1:」** ![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c3b067887994b99895080dd22e8cf3f~tplv-k3u1fbpfcp-zoom-1.image)

**「规则2:」** ![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af11a1a592c3438181d782668dffb69e~tplv-k3u1fbpfcp-zoom-1.image) 本文通过源码分析，给大家讲清楚《手册》为什么这么规定

# ArrayList的subList分析

首先通过 IDEA 的提供的类图工具，我们可以查看下该类的继承体系。

具体步骤：在 SubList 类中 右键，选择 “Diagrams” -> “Show Diagram” 。 ![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a6e48b93e26440a6a423226ae5181fc0~tplv-k3u1fbpfcp-zoom-1.image) 可以看到 SubList 和 ArrayList 的继承体系非常类似，都实现了 RandomAccess 接口 继承自 AbstarctList。

但是SubList 和 ArrayList 并没有继承关系，因此 ArrayList 的 SubList 并不能强转为 ArrayList 。

**「下面我们写一个简单的测试代码片段来验证转换异常问题：」**

```
public static void main(String[] args) {
    List<Integer> integerList = new ArrayList<>();
    integerList.add(0);
    integerList.add(1);
    integerList.add(2);
    List<Integer> subList = integerList.subList(0, 1);

    ArrayList<Integer> castList = (ArrayList<Integer>) subList;
  }
```

输出：

```
Exception in thread "main" java.lang.ClassCastException: java.util.ArrayList$SubList cannot be cast to java.util.ArrayList
```

从上面的结果也可以清晰地看出，subList 并不是 ArrayList 类型的实例，不能强转为 ArrayList 。

**「我们再来写一个代码片段：」**

```
public static void main(String[] args) {
    List<String> stringList = new ArrayList<>();
    stringList.add("月");
    stringList.add("伴");
    stringList.add("小");
    stringList.add("飞");
    stringList.add("鱼");
    stringList.add("哈");
    stringList.add("哈");

    List<String> subList = stringList.subList(2, 4);
    System.out.println("子列表：" + subList.toString());
    System.out.println("子列表长度：" + subList.size());

    subList.set(1, "周星星");
    System.out.println("子列表：" + subList.toString());
    System.out.println("原始列表：" + stringList.toString());
  }
```

输出：

```
子列表：[小, 飞]
子列表长度：2
子列表：[小, 周星星]
原始列表：[月, 伴, 小, 周星星, 鱼, 哈, 哈]
```

可以观察到，对子列表的修改最终对原始列表产生了影响。

**「那么为啥修改子序列的索引为 1 的值影响的是原始列表的第 4 个元素呢？」**

下面将进行源码分析和解读。 ![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c097d0fbd67a4402a4a0fecbc8773487~tplv-k3u1fbpfcp-zoom-1.image)

## 源码分析

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/df472aeb576946a7bcc34ae442ed7355~tplv-k3u1fbpfcp-zoom-1.image) 可以看到SubList是ArrayList的一个内部类

这个 SubList 中的 parent 字段就是原始的 List。我们 可以认为 SubList 是原始 List 的视图

看看`java.util.ArrayList#subList` 源码： ![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8725801d72d7405682a54892cb53bfdf~tplv-k3u1fbpfcp-zoom-1.image) 这里在构造子列表对象的时候传入了 this，创建了SubList类

同时从上面注释中我们可以学到以下知识点：

- 该方法返回本列表中 fromIndex （包含）和 toIndex （不包含）之间的元素视图。如果两个索引相等会返回一个空列表。
- 如果需要对 list 的某个范围的元素进行操作，可以用 subList，如：`list.subList(from, to).clear();`
- 任何对子列表的操作最终都会反映到原列表中。

我们再来看下SubList的构造函数

```
SubList(AbstractList<E> parent,
                int offset, int fromIndex, int toIndex) {
            this.parent = parent;
            this.parentOffset = fromIndex;
            this.offset = offset + fromIndex;
            this.size = toIndex - fromIndex;
            this.modCount = ArrayList.this.modCount;
        }
```

通过子列表的构造函数我们知道，这里的偏移量 ( offset ) 的值为 fromIndex 参数，因为参数传的offset等于0

接下来我们再看下函数 `java.util.ArrayList.SubList#set` 源码：

```
public E set(int index, E e) {
    rangeCheck(index);
    checkForComodification();
    E oldValue = ArrayList.this.elementData(offset + index);
    ArrayList.this.elementData[offset + index] = e;
    return oldValue;
}
```

可以看到替换值的时候，获取索引是通过 offset + index 计算得来的。

这里的 `java.util.ArrayList#elementData` 即为原始列表存储元素的数组。

因此上面提到的：

> ❝
>
> 为啥子序列的索引为 1 的值影响的是原始列表的第 4 个元素呢？
>
> ❞

这个问题就解答了

**「到现在完成了规则1的解释了」** ![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3db5357c568b4bb290618f731d66ee98~tplv-k3u1fbpfcp-zoom-1.image)

另外在 SubList 的构造函数中，我们发现会将 ArrayList 的 modCount 赋值给 SubList 的 modCount 。

**「所以这个就引出了另外一个问题」**

我们先看 `java.util.ArrayList#add(E)`的源码：

```
public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```

可以发现新增元素和删除元素，都会对 modCount 进行修改。

我们来看下 SubList 的 核心的函数 `java.util.ArrayList.SubList#get`

```
public E get(int index) {
            rangeCheck(index);
            checkForComodification();
            return ArrayList.this.elementData(offset + index);
        }
```

会进行修改检查：

```
private void checkForComodification() {
            if (ArrayList.this.modCount != this.modCount)
                throw new ConcurrentModificationException();
        }
```

我们再来看下 SubList 的 核心的函数 `java.util.ArrayList.SubList#add`

```
public void add(int index, E e) {
            rangeCheckForAdd(index);
            checkForComodification();
            parent.add(parentOffset + index, e);
            this.modCount = parent.modCount;
            this.size++;
        }
```

也会调用checkForComodification进行检查

从上面的 SubList 的构造函数我们可以知道，SubList 复制了 ArrayList 的 modCount，因此对原函数的新增或删除都会导致 ArrayList 的 modCount 的变化。

而子列表的遍历、增加、删除时又会检查创建 SubList 时的 modCount 是否一致，显然此时两者会不一致，导致抛出 ConcurrentModificationException (并发修改异常)。

**「至此上面的规则2的原因我们也清楚了。」** ![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e815dafa8cb745a7ab2cbfcd9f996080~tplv-k3u1fbpfcp-zoom-1.image)

**「既然 SubList 相当于原始 List 的视图，那么避免相互影响的修复方式有两种：」**

一种是，不直接使用 subList 方法返回的 SubList，而是重新使用 new ArrayList，在构 造方法传入 SubList，来构建一个独立的 ArrayList；

```
List<Integer> subList = new ArrayList<>(list.subList(1, 4));
```

另一种是，对于 Java 8 使用 Stream 的 skip 和 limit API 来跳过流中的元素，以及限制 流中元素的个数，同样可以达到 SubList 切片的目的。

```
List<Integer> subList = list.stream().skip(1).limit(3).collect(Collectors.toList());
```

# 总结

本文通过类图分析、源码分析以及的方式对 ArrayList 的 SubList 问题进行分析

**「要点：」**

- ArrayList 内部类 SubList 和 ArrayList 没有继承关系，因此无法将其强转为 ArrayList 。
- subList()返回的是 ArrayList 的内部类 SubList，并不是 ArrayList 本身，而是 ArrayList 的一个视 图，对于 SubList 的所有操作最终会反映到原列表上
- ArrayList 的 SubList 构造时传入 ArrayList 的 modCount，因此对原列表的修改将会导致子列表的遍历、增加、删除产生 ConcurrentModificationException 异常。

**「学习建议：」**

学习不能浮于表面，不能知其然而不知其所以然。而看源码是掌握深度知识最好的方法。

希望大家从现在开始学习和开发中能够偶尔到感兴趣的类中查看源码，这样学的更快，更扎实。通过进入源码中自主研究，这样印象更加深刻，掌握的程度更深。

**「觉得有收获，可以帮忙点赞，在看哈，您的点赞，转发对小飞鱼非常重要，谢谢」**

# 最后

微信搜索：月伴飞鱼，交个朋友

> ❝
>
> 1.日常分享一篇实用的技术文章，对面试，工作都有帮助
>
> ❞

> ❝
>
> 2.后台回复666，获得免费电子书籍，会持续更新
>
> ❞

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/06253c9a53694aa1b554526eec448679~tplv-k3u1fbpfcp-zoom-1.image)