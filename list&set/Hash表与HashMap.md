## 一、什么是哈希表？
在回答这个问题之前我们先来思考一个问题：**如何在一个无序的线性表中查找一个数据元素？**

注意，这是一个无序的线性表，也就是说要查找的这个元素在线性表中的位置是随机的。对于这样的情况，想要找到这个元素就必须对这个线性表进行遍历，然后与要查找的这个元素进行比较。这就意味着查找这个元素的时间复杂度为o(n)。对于o(n)的时间复杂度，在查找海量数据的时候也是一个非常消耗性能的操作。那么有没有一种数据结构，这个数据结构中的元素与它所在的位置存在一个对应关系，这样的话我们就可以通过这个元素直接找到它所在的位置，而此时查找这个元素的时间复杂度就变成了o(1),可以大大节省程序的查找效率。当然，这种数据结构是存在的，它就是我们今天要讲的**哈希表**。

我们先来看一下哈希表的定义：

> **哈希表**又叫**散列表**，是一种根据设定的映射函数f(key)将一组关键字映射到一个有限且连续的地址区间上，并以关键字在地址区间中的“像”作为元素在表中的存储位置的一种数据结构。这个映射过程称为**哈希造表**或者**散列**，这个映射函数f(key)即为**哈希函数**也叫**散列函数**，通过哈希函数得到的存储位置称为**哈希地址**或**散列地址**

定义总是这么的拗口且难以理解。简单来说，哈希表就是通过一个映射函数f(key)将一组数据散列存储在数组中的一种数据结构。在这哈希表中，每一个元素的key和它的存储位置都存在一个f(key)的映射关系，我们可以通过f(key)快速的查找到这个元素在表中的位置。

举个例子，有一组数据：[19,24,6,33,51,15]，我们用散列存储的方式将其存储在一个长度为11的数组中。采用**除留取余法**，将这组数据分别模上数组的长度（即f(key)=key % 11），以余数作为该元素在数组中的存储的位置。则会得到一个如下图所示的哈希表：

![哈希表示例](https://img-blog.csdnimg.cn/20200922233650707.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz,size_16,color_FFFFFF,t_70#pic_center)


此时，如果我们想从这个表中找到值为15的元素，只需要将15模上11即可得到15在数组中的存储位置。可见哈希表对于查找元素的效率是非常高的。

## 二、什么是哈希冲突
上一节中我们举了一个很简单的例子来解释什么是哈希表，例子中的这组数据只有6个元素。假如我们向这组数据中再插入一些元素，插入后的数据为：[19,24,6,33,51,15,25,72]，新元素25模11后得到3，存储到3的位置没有问题。而接下来我们对72模11之后得到了6，而此时在数组中6的位置已经被其他元素给占据了。“72“只能很无奈的表示我放哪呢？
![哈希冲突](https://img-blog.csdnimg.cn/20200922233756827.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz,size_16,color_FFFFFF,t_70#pic_center)
对于上述情况我们将其称之为哈希冲突。哈希冲突比较官方的定义为：

> 对于不同的关键字，可能得到同一个哈希地址，即key1≠key2,而 f(key1)=f(key2)，对于这种现象我们称之为**哈希冲突**，也叫**哈希碰撞**

一般情况下，哈希冲突只能尽可能的减少，但不可能完全避免。因为哈希函数是从关键字集合到地址集合的映射，通常来说关键字集合比较大，它的元素理论上包括所有可能的关键字，而地址集合的元素仅为哈希表中的地址值。这就导致了哈希冲突的必然性。

### 1.如何减少哈希冲突？
尽管哈希冲突不可避免，但是我们也要尽可能的减少哈希冲突的出现。一个好的哈希函数可以有效的减少哈希冲突的出现。那什么样的哈希函数才是一个好的哈希函数呢？通常来说，一个好的哈希函数对于关键字集合中的任意一个关键字，经过这个函数映射到地址集合中任何一个集合的概率是相等的。 

常用的构造哈希函数的方法有以下几种：
**（1）除留取余法**

这个方法我们在上边已经有接触过了。取关键字被某个不大于哈希表长m的数p除后所得余数为哈希地址。即：f(key)=key % p, p≤m;

**（2）直接定址法**

直接定址法是指取关键字或关键字的某个线性函数值为哈希地址。即： f(key)=key 或者 f(key)=a*key+b、

**（3）数字分析法**

假设关键字是以为基的数（如以10为基的十进制数），并且哈希表中可能出现的关键字都是事先知道的，则可以选取关键字的若干位数组成哈希表。

当然，除了上边列举的几种方法，还有很多种选取哈希函数的方法，就不一一列举了。我们只要知道，选取合适的哈希函数可以有效减少哈希冲突即可。

### 2.如何处理哈希冲突？

虽然我们可以通过选取好的哈希函数来减少哈希冲突，但是哈希冲突终究是避免不了的。那么，碰到哈希冲突应该怎么处理呢？接下来我们来介绍几种处理哈希冲突的方法。

**（1）开放定址法**

开放定址法是指当发生地址冲突时，按照某种方法继续探测哈希表中的其他存储单元，直到找到空位置为止。

我们以本节开头的例子来讲解开放定址法是如何处理冲突的。72模11后得到6，而此时6的位置已经被其他元素占用了，那么将6加1得到7，
此时发现7的位置也被占用了，那就再加1得到下一个地址为8，而此时8仍然被占用，再接着加1得到9，此时9处为空，则将72存入其中，即得到如下哈希表：

![线性探测再散列](https://img-blog.csdnimg.cn/20200923005618516.png#pic_center)
像上边的这种探测方法称为**线性探测再散列**。当然除了线性探测再散列之外还有二次探测再散列，探测地址的方式为原哈希地址加上d (d= $±1^2$、$±2^2$、$±3^2$......$±m^2$)，经过二次探测再散列后会得到求得72的哈希地址为5，存储如下图所示：

![二次探测再散列](https://img-blog.csdnimg.cn/20200923010619960.png#pic_center)

**（2）再哈希法**

再哈希法即选取若干个不同的哈希函数，在产生哈希冲突的时候计算另一个哈希函数，直到不再发生冲突为止。

**（3）建立公共溢出区**

专门维护一个溢出表，当发生哈希冲突时，将值填入溢出表。

**（4）链地址法**

链地址法是指在碰到哈希冲突的时候，将冲突的元素以链表的形式进行存储。也就是凡是哈希地址为**i**的元素都插入到同一个链表中，元素插入的位置可以是表头（**头插法**），也可以是表尾（**尾插法**）。我们以仍然以[19,24,6,33,51,15,25,72]
这一组数据为例，用链地址法来进行哈希冲突的处理，得到如下图所示的哈希表：

![链地址法](https://img-blog.csdnimg.cn/20200923223502759.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz,size_16,color_FFFFFF,t_70#pic_center)
我们可以向这组数据中再添加一些元素，得到一组新的数据[19,24,6,33,51,15,25,72,37,17,4,55,83]。使用链地址法得到如下哈希表：

![链地址法](https://img-blog.csdnimg.cn/20200923224438164.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz,size_16,color_FFFFFF,t_70#pic_center)

## 三、链地址法的弊端与优化
上一节中我们讲解了几种常用的处理哈希冲突的方法。其中比较常用的是链地址法，比如HashMap就是基于链地址法的哈希表结构。虽然链地址法是一种很好的处理哈希冲突的方法，但是在一些极端情况下链地址法也会出现问题。举个例子，我们现在有这样一组数据：[48,15,26,4,70,82,59]。我们将这组数据仍然散列存储到长度为11的数组中，此时则得到了如下的结果：

![链地址法存在的问题](https://img-blog.csdnimg.cn/20200923232503838.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz,size_16,color_FFFFFF,t_70#pic_center)

可以发现，此时的哈希表俨然已经退化成了一个链表，当我们在这样的数据结构中去查找某个元素的话，时间复杂度又变回了o(n)。这显然不符合我们的预期。因此，当哈希表中的链表过长时就需要我们对其进行优化。我们知道，二叉查找树的查询效率是远远高于链表的。因此，当哈希表中的链表过长时我们就可以把这个链表变成一棵红黑树。上面的一组数据优化后可得到如下结果：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200927225706751.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz,size_16,color_FFFFFF,t_70#pic_center)



红黑树是一个可以自平衡的二叉查找树。它的查询的时间复杂度为o(lgn)。通过这样的优化可以提高哈希表的查询效率。
## 四、哈希表的扩容与Rehash

在哈希表长度不变的情况下，随着哈希表中插入的元素越来越多，发生哈希冲突的概率会越来越大，相应的查找的效率就会越来越低。这意味着影响哈希表性能的因素除了哈希函数与处理冲突的方法之外，还与哈希表的**装填因子**大小有关。


>我们将哈希表中元素数与哈希表长度的比值称为**装填因子**。装填因子 **α= $\frac{哈希表中元素数}{哈希表长度}$**  

很显然，**α**的值越小哈希冲突的概率越小，查找时的效率也就越高。而减小**α**的值就意味着降低了哈希表的使用率。显然这是一个矛盾的关系，不可能有完美解。为了兼顾彼此，装填因子的最大值一般选在0.65~0.9之间。比如HashMap中就将装填因子定为0.75。一旦HashMap的装填因子大于0.75的时候，为了减少哈希冲突，就需要对哈希表进行**扩容**操作。比如我们可以将哈希表的长度扩大到原来的2倍。

这里我们应该知道，扩容并不是在原数组基础上扩大容量，而是需要申请一个长度为原来2倍的新数组。因此，扩容之后就需要将原来的数据从旧数组中重新散列存放到扩容后的新数组。这个过程我们称之为**Rehash**。

接下来我们仍然以[19,24,6,33,51,15,25,72,37,17,4,55,83]这组数据为例来演示哈希表扩容与Rehash的过程。假设哈希表的初始长度为11，装载因子的最大值定位0.75，扩容前的数据插入如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201128120210212.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz,size_16,color_FFFFFF,t_70#pic_center)

当我们插入第9个元素的时候发现此时的装填因子已经大于了0.75，因此触发了扩容操作。为了方便画图，这里将数组长度扩展到了18。扩容后将[19,24,6,33,51,15,25,72,37,17,4,55,83]这组数据重新散列，会得到如下图所示的结果：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200925142348236.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz,size_16,color_FFFFFF,t_70#pic_center)


可以看到扩容前后元素存储位置大相径庭。Rehash的操作将会重新散列扩容前已经存储的数据，这一操作涉及大量的元素移动，是一个非常消耗性能的操作。因此，在开发中我们应该尽量避免Rehash的出现。比如，可以预估元素的个数，事先指定哈希表的长度，这样可以有效减少Rehash。

## 五、总结

哈希表是数据结构中非常重要的一个知识点，本篇文章详细的讲解了哈希表的相关概念，让大家对哈希表有了一个清晰的认识。哈希表弥补了线性表或者树的查找效率低的问题，通过哈希表在理想的情况下可以将查找某个元素的时间复杂度降低到o(1),但是由于哈希冲突的存在，哈希表的查找效率很难达到理想的效果。另外，哈希表的扩容与Rehash的操作对哈希表存储时的性能也有很大的影响。由此可见使用哈希表存储数据也并非一个完美的方案。但是，对于查找性能要求高的情况下哈希表的数据结构还是我们的不二选择。

最后了解了哈希表对于理解HashMap会有莫大的帮助。毕竟HashMap本身就是基于哈希表实现的。

[面试官：哈希表都不知道，你是怎么看懂HashMap的？](https://juejin.cn/post/6876105622274703368)