#  关于Comparable和Comparator(一)

> 假如你想排序请告诉我比较的规则

## Comparable和Comparator的介绍

如果是数字的话，排序对应的比较的规则是显而易见的，假如a > b, 则a - b > 0 。字符串是另一种形式的数字，字符串还是字符数组，字符又可以当做数字来看，所以字符串的排序规则也可以参照着数字来。那么对于对象和Map这种类型的数据 , 那么排序的规则就需要自己制定了。java中我们在排序的时候也就是三个函数:
-   Collections.sort(List<T> list)
-  Collections.sort(List<T> list, Comparator<? super T> c)
-  Arrays.sort() 重载不再列出

那么该如何制定排序规则呢? 通过Comparable和Comparator制定排序规则。
- Comparable用于两个对象之间的比较。
- Comparator则是对集合中的所有对象起作用，用于集合的排序。

也就是说当你对集合中进行排序的时候，也就是调用 Collections.sort的时候，集合中的数据对应的类一定要实现Comparable接口或者给定一个Comparator的实现类的实例。

仔细想想这么设计也是合理的，不论怎么样，排序是一定要给定比较规则的。

那么比较规则应该怎么制定呢? 让我们从数字身上寻找答案。 假如 a > b, 那么 a - b > 0 , 如果 a  =  b, 那么 a - b = 0。a < b , 那么 a - b < 0。这是很自然的事情，我们都习以为常。对于
Comparable和Comparator也是一样，我们可以认为Comparable和Comparator是一个对象的运算符，像是减号一样。

- 假如 Comparable 返回的大于0，则说明a 对象 在 Comparable 下大于 b。
- 假如 Comparable 返回的值小于0，则说明a对象在 Comparable 下小于 b对象。
- 假如 Comparable 返回的值等于0，则说明a 对象 在Comparable 下等于 b对象。

这种关系事实上可以有更简洁的描述，不过我们需要引入一个数学函数sgn(x)，
-  x > 0   sgn(x) = 1
-  x < 0   sgn(x) = -1
-  x = 0   sgn(x) = 0


## 一定要确保

- Comparator的实现类一定要确保: 对任意的y ， sgn(compare(x,y)) == -sgn(compare(y,x)) , 
- Comparator的实现类一定要确保: 对任意的z ，如果有 ((compare(x, y)>0) && (compare(y, z)>0)) 则 compare(x, z)>0.
- Comparator的实现类一定要确保:  对任意的z ，如果有 compare(x, y)==  0  则  sgn(compare(x, z))==sgn(compare(y, z))

对任意的y, sgn(compare(x,y)) == -sgn(compare(y,x)).

 上文我们提到可以将compare()理解为减号，那么 compare(x,y) 就可以理解为x - y。如果x-y > 0 , 那么 y - x < 0。于是对任意的y, sgn(compare(x,y)) == -sgn(compare(y,x))。不可以违反此协定，违反此协定，排序的时候就会失灵。

对于第二条的话，对于学习过《离散数学》的同学来说估计会想到: 关系的传递性。第二条在我们将Comparator理解为减号的情况下，也是十分自然的。即如果 x - y > 0 且 y - z > 0 ，则 x - z > 0.

第三条的话，也是让人联想到关系，x - y = 0 => x = y => x - z = y - z。

谈到比较的话，各位有没有想到equals方法呢，equals方法表达的也是比较，Comparator中的源码中建议但是不严格要求: 
>  如果 compare(x,y) = 0, 那么equlas方法返回true。
>  It is generally the case, but <i>not</i> strictly required that <tt>(compare(x, y)==0) == (x.equals(y))</tt>

## 总结
- Comparable和Comparator主要用于排序，假如集合中的对象已经实现了Comparable接口，那么就不需要在指定外部比较器了。
- Comparable通常的用法是两个对象之间，比较两个对象的大小。
- Comparator则作用于集合中全体的元素。这也是内部排序器和外部排序器的由来。
- Comparator也被用于控制一定数据结构的顺序，像SortedSet和SortedMap, 这是两个接口，你可能不是很熟，但这两个接口有两个实现类，你一定很熟: TreeMap,TreeSet。那HashMap就是无序的Map了吗? 这是另一个话题了。
建议维持比较的一致性， 即(compare(x, y)==0) == (x.equals(y))

参考资料:  Comparable和Comparator 的注释