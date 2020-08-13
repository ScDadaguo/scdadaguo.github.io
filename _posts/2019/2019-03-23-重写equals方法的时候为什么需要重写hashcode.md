---
layout: post
title: 重写equals方法的时候为什么需要重写hashcode
category: 默认分类
tags: []
keywords: 
---

#重写equals方法的时候为什么需要重写hashcode/n重写equals方法的时候为什么需要重写hashcode
============================


一、equals()方法
============

先说说equals()方法。  
  查看Java的Object.equals()方法，如下：

    public boolean equals(Object object){
          return(this == obj);
    }
    

可以看到这里直接用'=='来直接比较，引用《Java编程思想》里的一句话：“关系操作符生成的是一个boolean结果，它们计算的是操作数的值之间的关系”。那么'=='比较的值到底是什么呢？  
  我们知道Java有8种基本类型：数值型（byte、short、int、long、float、double）、字符型（char）、布尔型（boolean），对于这8种基本类型的比较，变量存储的就是值，所以比较的就是'值'本身。如下，值相等就是true，不等就是false。

    public static void main(String[] args) {  
            int a=3;                                           
            int b=4;
            int c=3;
            System.out.println(a==b);   //false
            System.out.println(a==c);   //true
        }
    

对于非基本类型，也就是常说的引用数据类型：类、接口、数组，由于变量种存储的是内存中的地址，并不是'值'本身，所以真正比较的是该变量存储的地址，可想而知，如果声明的时候是2个对象，地址固然不同。

    public static void main(String[] args) {
            String str1 = new String("123");
            String str2 = new String("123");
            System.out.println(str1 == str2);  //false
        }
    

可以看到，上面这种比较方法，和Object类中的equals()方法的具体实现相同，之所以为false，是因为直接比较的是str1和str2指向的地址，也就是说**Object中的equals方法是直接比较的地址**，因为Object类是所有类的基类，所以调用新创建的类的equals方法，比较的就是两个对象的地址。那么就有人要问了，如果就是想要比较引用类型实际的值是否相等，该如何比较呢？  
    铛铛铛...... 重点来了

* * *

要解决上面的问题，就是今天要说的equals()，具体的比较由各自去重写，比较具体的值的大小。我们可以看看上面字符串的比较，如果调用String的equals方法的结果。

    public static void main(String[] args) {
            String str1 = new String("123");
            String str2 = new String("123");
            System.out.println(str1.equals(str2));  //true
        }
    

可以看到返回的true，由兴趣的同学可以去看String equals()的源码。

* * *

所以可以通过重写equals()方法来判断对象的值是否相等，但是有一个要求：**equals()方法实现了等价关系**，即：

*   自反性：对于任何非空引用x，x.equals(x)应该返回true；
*   对称性：对于任何引用x和y，如果x.equals(y)返回true，那么y.equals(x)也应该返回true；
*   传递性：对于任何引用x、y和z，如果x.equals(y)返回true，y.equals(z)返回true，那么x.equals(z)也应该返回true；
*   一致性：如果x和y引用的对象没有发生变化，那么反复调用x.equals(y)应该返回同样的结果；
*   非空性：对于任意非空引用x，x.equals(null)应该返回false；

二、hashCode()方法
==============

此方法返回对象的哈希码值，什么是哈希码？度娘找到的相关定义：

> 哈希码产生的依据：哈希码并不是完全唯一的，它是一种算法，让同一个类的对象按照自己不同的特征尽量的有不同的哈希码，但不表示不同的对象哈希码完全不同。也有相同的情况，看程序员如何写哈希码的算法。

简单理解就是一套算法算出来的一个值，且这个值对于这个对象相对唯一。哈希算法有一个协定：在 Java 应用程序执行期间，在对同一对象多次调用 hashCode 方法时，必须一致地返回相同的整数，_前提是将对象进行hashcode比较时所用的信息没有被修改_。（ps：要是每次都返回不一样的，就没法玩儿了）

    public static void main(String[] args) {
            List<Long> test1 = new ArrayList<Long>();
            test1.add(1L);
            test1.add(2L);
            System.out.println(test1.hashCode());  //994
            test1.set(0,2L);
            System.out.println(test1.hashCode());  //1025
        }
    

三、标题解答
======

首先来看一段代码：

    public class HashMapTest {
        private int a;
    
        public HashMapTest(int a) {
            this.a = a;
        }
    
        public static void main(String[] args) {
            Map<HashMapTest, Integer> map = new HashMap<HashMapTest, Integer>();
            HashMapTest instance = new HashMapTest(1);
            map.put(instance, 1);
            Integer value = map.get(new HashMapTest(1));
            if (value != null) {
                System.out.println(value);
            } else {
                System.out.println("value is null");
            }
        } 
    
    }
    //程序运行结果： value is null
    

简单说下HashMap的原理，HashMap存储数据的时候，是取的key值的哈希值，然后计算数组下标，采用链地址法解决冲突，然后进行存储；取数据的时候，依然是先要获取到hash值，找到数组下标，然后for遍历链表集合，进行比较是否有对应的key。比较关心的有2点：_1._不管是put还是get的时候，都需要得到key的哈希值，去定位key的数组下标； _2._在get的时候，需要调用equals方法比较是否有相等的key存储过。  
  反过来，我们再分析上面那段代码，Map的key是我们自己定义的一个类，可以看到，我们没有重写equal方法，更没重写hashCode方法，意思是map在进行存储的时候是调用的Object类中equals()和hashCode()方法。为了证实，我们打印下hashCode码。

    public class HashMapTest {
        private Integer a;
    
        public HashMapTest(int a) {
            this.a = a;
        }
    
        public static void main(String[] args) {
            Map<HashMapTest, Integer> map = new HashMap<HashMapTest, Integer>();
            HashMapTest instance = new HashMapTest(1);
            System.out.println("instance.hashcode:" + instance.hashCode());
            map.put(instance, 1);
            HashMapTest newInstance = new HashMapTest(1);
            System.out.println("newInstance.hashcode:" + newInstance.hashCode());
            Integer value = map.get(newInstance);
            if (value != null) {
                System.out.println(value);
            } else {
                System.out.println("value is null");
            }
        }
    }
    //运行结果：
    //instance.hashcode:929338653
    //newInstance.hashcode:1259475182
    //value is null
    

不出所料，hashCode不一致，所以对于为什么拿不到数据就很清楚了。这2个key，在Map计算的时候，可能数组下标就不一致，就算数据下标碰巧一致，根据前面，最后equals比较的时候也不可能相等（很显然，这是2个对象，在堆上的地址必定不一样）。我们继续往下看，假如我们重写了equals方法，将这2个对象都put进去，根据map的原理，只要是key一样，后面的值会替换前面的值，接下来我们实验下：

    public class HashMapTest {
        private Integer a;
    
        public HashMapTest(int a) {
            this.a = a;
        }
    
        public static void main(String[] args) {
            Map<HashMapTest, Integer> map = new HashMap<HashMapTest, Integer>();
            HashMapTest instance = new HashMapTest(1);
            HashMapTest newInstance = new HashMapTest(1);
            map.put(instance, 1);
            map.put(newInstance, 2);
            Integer value = map.get(instance);
            System.out.println("instance value:"+value);
            Integer value1 = map.get(newInstance);
            System.out.println("newInstance value:"+value1);
    
        }
    
        public boolean equals(Object o) {
            if(o == this) {
                return true;
            } else if(!(o instanceof HashMapTest)) {
                return false;
            } else {
                HashMapTest other = (HashMapTest)o;
                if(!other.canEqual(this)) {
                    return false;
                } else {
                    Integer this$data = this.getA();
                    Integer other$data = other.getA();
                    if(this$data == null) {
                        if(other$data != null) {
                            return false;
                        }
                    } else if(!this$data.equals(other$data)) {
                        return false;
                    }
    
                    return true;
                }
            }
        }
        protected boolean canEqual(Object other) {
            return other instanceof HashMapTest;
        }
    
        public void setA(Integer a) {
            this.a = a;
        }
    
        public Integer getA() {
            return a;
        }
    }
    //运行结果：
    //instance value:1
    //newInstance value:2
    

你会发现，不对呀？同样的一个对象，为什么在map中存了2份，map的key值不是不能重复的么？没错，它就是存的2份，只不过在它看来，这2个的key是不一样的，因为他们的哈希码就是不一样的，可以自己测试下，上面打印的hash码确实不一样。那怎么办？只有重写hashCode()方法，更改后的代码如下：

    public class HashMapTest {
        private Integer a;
    
        public HashMapTest(int a) {
            this.a = a;
        }
    
        public static void main(String[] args) {
            Map<HashMapTest, Integer> map = new HashMap<HashMapTest, Integer>();
            HashMapTest instance = new HashMapTest(1);
            System.out.println("instance.hashcode:" + instance.hashCode());
            HashMapTest newInstance = new HashMapTest(1);
            System.out.println("newInstance.hashcode:" + newInstance.hashCode());
            map.put(instance, 1);
            map.put(newInstance, 2);
            Integer value = map.get(instance);
            System.out.println("instance value:"+value);
            Integer value1 = map.get(newInstance);
            System.out.println("newInstance value:"+value1);
    
        }
    
        public boolean equals(Object o) {
            if(o == this) {
                return true;
            } else if(!(o instanceof HashMapTest)) {
                return false;
            } else {
                HashMapTest other = (HashMapTest)o;
                if(!other.canEqual(this)) {
                    return false;
                } else {
                    Integer this$data = this.getA();
                    Integer other$data = other.getA();
                    if(this$data == null) {
                        if(other$data != null) {
                            return false;
                        }
                    } else if(!this$data.equals(other$data)) {
                        return false;
                    }
    
                    return true;
                }
            }
        }
        protected boolean canEqual(Object other) {
            return other instanceof HashMapTest;
        }
    
        public void setA(Integer a) {
            this.a = a;
        }
    
        public Integer getA() {
            return a;
        }
    
        public int hashCode() {
            boolean PRIME = true;
            byte result = 1;
            Integer $data = this.getA();
            int result1 = result * 59 + ($data == null?43:$data.hashCode());
            return result1;
        }
    }
    //运行结果：
    //instance.hashcode:60
    //newInstance.hashcode:60
    //instance value:2
    //newInstance value:2
    

可以看到，他们的hash码是一致的，且最后的结果也是预期的。

* * *

_完美的分界线_

ps.总结：对于这个问题，是比较容易被忽视的，曾经同时趟过这坑，Map中存了2个数值一样的key，所以大家谨记哟！ 在重写equals方法的时候，一定要重写hashCode方法。  
最后一点：有这个要求的症结在于，要考虑到类似HashMap、HashTable、HashSet的这种散列的数据类型的运用。


<!--more-->


HashMap基于hashing原理，我们通过put()和get()方法储存和获取对象。当我们将键值对传递给put()方法时，它调用键对象的hashCode()方法来计算hashcode，让后找到bucket位置来储存值对象。当获取对象时，通过键对象的equals()方法找到正确的键值对，然后返回值对象。HashMap使用LinkedList来解决碰撞问题，当发生碰撞了，对象将会储存在LinkedList的下一个节点中。 HashMap在每个LinkedList节点中储存键值对对象。

　　当两个不同的键对象的hashcode相同时会发生什么？ 它们会储存在同一个bucket位置的LinkedList中。键对象的equals()方法用来找到键值对。

```
hashmap里的hash方法 就是这个方法来找位置的 它调用的是Object的hashCode（）

 static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
所以想要当hashmap的键必须重写hashcode()，不然肯定就算相同的对象hashcode也不一样 没办法存了
