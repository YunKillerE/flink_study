# IMPLICIT CONVERSIONS
1. 什么是隐式转换？
2. 为什么scala中有隐式转换？
4. 隐式参数理解
5. 隐式转换理解
6. spark中的隐式转换举例
7. 什么是闭包？

## 什么是函数式编程？



## 什么是隐式转换？

Implicit Conversions are a set of methods that Scala tries to apply when it encounters an object of the wrong type being used

隐式转换是scala在遇到使用错误类型的对象时尝试应用的一组方法

## scala隐式转换的用途

1. 隐式值
2. 隐式转换为目标类型：把一种类型自动转换到另一种类型
3. 隐式转换调用类中本不存在的方法
4. 隐式类
5. 隐式对象

## 隐式参数理解 -- 隐式值

```
def person(implicit name : String) = name
```

```
scala> implicit val p = "mobin"   //p被称为隐式值
p: String = mobin
scala> person
res1: String = mobin
```

```
scala> implicit val p1 = "mobin1"
p1: String = mobin1
scala> person
<console>:11: error: ambiguous implicit values:
 both value p of type => String
 and value p1 of type => String
 match expected type String
              person
              ^
```

隐式转换必须满足无歧义规则，在声明隐式参数的类型是最好使用特别的或自定义的数据类型，不要使用Int,String这些常用类型，避免碰巧匹配

## 隐式转换理解

1. 隐式转换为目标类型：把一种类型自动转换到另一种类型
2. 隐式转换调用类中本不存在的方法
3. 隐式类
4. 隐式对象


### 隐式转换为目标类型：把一种类型自动转换到另一种类型

```
def foo(msg : String) = println(msg)
foo: (msg: String)Unit
  
scala> foo(100)
<console>:11: error: type mismatch;
found : Int(10)
required: String
foo(100)
```
显然不能转换成功，解决办法就是定义一个转换函数给编译器将int自动转换成String
```
scala> implicit def intToString(x : Int) = x.toString
intToString: (x: Int)String
  
scala> foo(100)
100
```

### 隐式转换调用类中本不存在的方法

```
val lines = sc.textFile("data.txt")
val pairs = lines.map(s => (s, 1))
val counts = pairs.reduceByKey((a, b) => a + b)
```

pairs这个RDD并没有reduceByKey这个函数，为什么能调用成功？

```
  implicit def rddToPairRDDFunctions[K, V](rdd: RDD[(K, V)])
    (implicit kt: ClassTag[K], vt: ClassTag[V], ord: Ordering[K] = null): PairRDDFunctions[K, V] = {
    new PairRDDFunctions(rdd)
  }
```

在RDD的object里面有这么个隐式函数会进行自动类型转换，PairRDDFunctions里面有这个函数

### 隐式类

```
object Stringutils {
  implicit class StringImprovement(val s : String){   //隐式类
    def increment = s.map(x => (x +1).toChar)
  }
}

object  Main extends  App{
  import Stringutils._
  println("mobin".increment)
}
```

编译器在mobin对象调用increment时发现对象上并没有increment方法，此时编译器就会在作用域范围内搜索隐式实体，发现有符合的隐式类可以用来转换成带有increment方法的StringImprovement类，最终调用increment方法


# spark中的隐式转换举例

在RDD.scala中Object对象里面有几个implicit函数，早期版本好像在SparkConetxt.scala中（before 1.3）

```
  implicit def rddToPairRDDFunctions[K, V](rdd: RDD[(K, V)])
    (implicit kt: ClassTag[K], vt: ClassTag[V], ord: Ordering[K] = null): PairRDDFunctions[K, V] = {
    new PairRDDFunctions(rdd)
  }
  
    implicit def rddToPairRDDFunctions[K, V](rdd: RDD[(K, V)])
      (implicit kt: ClassTag[K], vt: ClassTag[V], ord: Ordering[K] = null): PairRDDFunctions[K, V] = {
      new PairRDDFunctions(rdd)
    }
  
    implicit def rddToAsyncRDDActions[T: ClassTag](rdd: RDD[T]): AsyncRDDActions[T] = {
      new AsyncRDDActions(rdd)
    }
    
    .....
  
```

有这些隐式转换函数在就能调用RDD.scala中并不存在的，比如aggregateByKey，countByKey等函数，避免了写多个rdd的对象



参考：

		https://www.cnblogs.com/xia520pi/p/8745923.html
		https://www.jianshu.com/p/4b1b5eaa6611
		https://www.jianshu.com/p/a344914de895