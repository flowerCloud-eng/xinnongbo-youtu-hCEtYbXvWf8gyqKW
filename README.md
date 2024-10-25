
最近在学习CodeQL，对于CodeQL就不介绍了，目前网上一搜一大把。本系列是学习CodeQL的个人学习笔记，根据个人知识库笔记修改整理而来的，分享出来共同学习。个人觉得QL的语法比较反人类，至少与目前主流的这些OOP语言相比，还是有一定难度的。与现在网上的大多数所谓CodeQL教程不同，本系列基于[官方文档](https://github.com)和[情景实例](https://github.com):[豆荚加速器](https://yirou.org)，包含大量的个人理解、思考和延伸，直入主题，只切要害，几乎没有废话，并且坚持用从每一个实例中学习总结归纳，再到实例中验证。希望能给各位一点不一样的见解和思路。当然，也正是如此必定会包含一定的错误，希望各位大佬能在评论区留言指正。




---


先来看一下一些基本的概念和结构



```
// 基本结构
from /* variable declarations */
where /* logical formulas */
select /* expressions */

from int a, int b
where x = 3, y = 4
select x, y
// 找出1-10内的勾股数
from int x, int y, int z
where x in [1..10], y in [1..10], z in [1..10] and
    x * x + y * y = z * z
select x, y, z

// 或下面这种类写法，封装和方法复用
class SmallInt extends int {
    SmallInt(){
        this in [1..10]
    }
    int square(){
        result = this * this
    }
}
from SmallInt x, SmallInt y, SmallInt z
where x.sqrt() + y.square() = z.sqrt()
select x, y, z

```

## 逻辑连接词、量词、聚合词



```
from Person p
where p.getAge() = max(int i | exists(Person t | t.getAge() = i ) | i)        // 通用聚合语法，比较冗长
select p

// 或用以下有序聚合方式
select max(Person p | | p order by p.getAge())

```


> `exists(<变量声明> | <条件表达式>)`
> 
> 
>  `( <变量声明> | <逻辑表达式(限制符合条件的数据范围)> | <表达式(返回经过筛选的)> )`
> 
> 
> e.g. `exists( Person p | p.getName() = "test" )`，判断是否存在一个人的名字为test
> 
> 
> `max(int i | exists(Person p | p.getAge() = i) | i)`，第二部分的意思是拿到所有人的年龄放入i，第三部分是作用范围为i，目前i为int数组，存放所有人的年龄，即最终计算的是`max(i)`
> 
> 
> `select max(Person p | | p order by p.getAge())`，考虑每个人，取出年龄最大的人。过程是按照年龄来取最大值，换句话说，`order by p.getAge()` 是告诉 max() 函数要基于 getAge() 来找最大值，并不涉及对所有对象的排序操作。



```
// 其他有序聚合练习
select min(Person p | p.getLocation() = "east" | p order by p.getHeight())  // 村东最矮的人
select count(Person p | p.getLocation() = "south" | p)  // 村南的人数
select avg(Person p |  | p.getHeight()) // 村民平均身高
select sum(Person p | p.getHairColor() = "brown" | p.getAge())  // 所有棕色头发的村民年龄总和
// 综合练习，https://codeql.github.com/docs/writing-codeql-queries/find-the-thief/#the-real-investigation
import tutorial
from Person p 
where 
    p.getHeight() > 150 and // 身高超过150
    not p.getHairColor() = "blond" and  // 头发颜色不是金发
    exists(string c | p.getHairColor() = c) and // 不是秃头。这里表示这个人存在某种发色，但不用具象化
    not p.getAge() < 30 and // 年龄满30岁。也可以是p.getAge() >= 30
    p.getLocation() = "east" and    // 住在东边
    ( p.getHairColor() = "black" or p.getHairColor() = "brown" ) and    // 头发是黑色或棕色
    not (p.getHeight() > 180 and p.getHeight() < 190) and   // 没有（超过180且矮于190）
    exists(Person t | t.getAge() > p.getAge()) and   // 不是最年长的人。这里用存在语法是，存在一个人比他的年龄大
    exists(Person t | t.getHeight() > p.getHeight()) and    // 不是最高的人
    p.getHeight() < avg(Person t | | t.getHeight()) and // 比平均身高要矮。所有人，没有限制范围
    p = max(Person t | t.getLocation() = "east" | t order by t.getAge())    // 东部年纪最大的人。这一行是官方给的参考，但是官方文档中说 "Note that if there are several people with the same maximum age, the query lists all of them."，如果存在最大年龄相同的两个人会同时列出，可能会造成不可控的后果。
    // p.getAge() = max(Person t | t.getLocation() = "east" | t.getAge())   // 按照个人理解和chatgpt的解答，应该使用这种方式
select p

```



---


## 谓词和类别


CodeQL中的谓词(Predicates)的说法大概可以理解为其他高级编程语言中的函数，同样拥有可传参、返回值、可复用等特性


先来看一个简单的例子



```
import tutorial
predicate isSouthern(Person p) {
    p.getLocation() = "south"
}

from Person p
where isSouthern(p)
select p

```


> 这里的predicate为一个逻辑条件判断，返回true or false，有些类似于boolean，当然ql中有单独的boolean类型，还是有一定区别的，只是理解上可以联系起来理解，这里先不展开


谓词的定义方式和函数类似，其中的predicate可以替换为返回结果类型，例如`int getAge() { result = xxx }`，谓词名称只能以小写字母开头


此外，还可以定义一个新类，直接包含isSouthern的人



```
class Southerner extends Person {
  Southerner() { isSouthern(this) }
}
from Southerner s
select s

```


> 这里类似于面向对象语言（OOL）中的类定义，同样拥有继承、封装、方法等；这里的`Southerner()`类似于构造函数，但是不同于类中的构造函数，这里是一个逻辑属性，并不会创建对象。ool类中的方法在ql中被称为类成员谓词
> 
> 
> 表达式`isSouthern(this)`定义了这个类的逻辑属性，称作`特征谓词`，他用一个变量`this`（这里的this理解同ool）表示：如果属性`isSouthern(this)`成立，则一个`Person`\-\-`this`是一个`Southerner`。简单理解就是ql中每个继承子类的特征谓词表示的是`什么样的父类是我这种子类`、`我这种子类在父类的基础上还具有什么特征/特性`
> 
> 
> 引用官方文档：QL 中的类表示一种逻辑属性：当某个值满足该属性时，它就是该类的成员。这意味着一个值可以属于许多类 — 属于某个特定类并不妨碍它属于其他类。


来看下面这个例子



```
class Child extends Person {
    Child(){
        this.getAge() < 10
    }
    override predicate isAllowedIn(string region) {
        region = this.getLocation()
    }
}
// Person父类中的isAllowedIn实现如下：
predicate isAllowedIn(string region) { region = ["north", "south", "east", "west"] }
// 父类isAllowedIn(region)方法永远返回的是true，子类返回的是当前所在区域才为true（getLocation()方法）

```

看一个完整的例子



```
import tutorial
predicate isSoutherner(Person p) {
    p.getLocation() = "south"
}
class Southerner extends Person {
    Southerner(){isSoutherner(this)}
}
class Child extends Person {
    Child(){this.getAge() < 10}
    override predicate isAllowedIn(string region) {
        region = this.getLocation()
    }
}

from Southerner s 
where s.isAllowedIn("north")
select s, s.getAge()

```

这里有个概念非常重要，要与ool的类完全区别开来，在ool的类中，继承的子类中重构的方法是不会影响其他继承子类的，每个子类不需要考虑是否交错。但是在QL中，引用官方文档的一句话`QL 中的类表示一种逻辑属性：当某个值满足该属性时，它就是该类的成员。这意味着一个值可以属于许多类 — 属于某个特定类并不妨碍它属于其他类`，在ql的每个子类中，只要满足其特征谓词，就是这个子类的成员。


针对如上代码中这个具体的例子，如果Person中有人同时满足Southerner和Child的特征关系，则同时属于这两个类，自然也会继承其中的成员谓词。


个人理解，其实QL中的子类就是把父类全部拿出来，然后根据特征谓词来匹配父类中的某些元素，然后去复写/重构其中的这些元素的成员谓词，事实上是对父类中的元素进行了修改。下面用三个实例来对比理解



```
// 从所有Person中取出当前在South的，然后取出其中能去north的。因为把child限定了只能呆在当地，因此取出的这部分Southerner中的Child全都没法去north，因此就把这部分（原本在South的）Child过滤了
from Southerner s
where s.isAllowedIn("north")
select s

// 取出所有Child，因此他们都只能呆在原地，因此找谁能去north就是找谁原本呆在north
from Child c
where c.isAllowedIn("north")
select c

// 取出所有Person，要找谁能去north的，即找所有成年人（默认所有人都可以前往所有地区）和找本来就呆在north的Child
from Person p
where p.isAllowedIn("north")
select p

```


> 延伸一下，如果多个子类别同时重构override了同一个成员谓词，那么遵循如下规则（先假定有三个类A、B、C）（后面有总结）：
> 
> 
> 1. 假定A是父类，即其中的某个成员谓词`test()`没有override，B和C同时继承A，并且都override了A的`test()`成员谓词。
> 	1. 如果from的谓词类型是A，则其中的`test()`方法会被B和C全部改写。碰到B与C重叠的部分，不冲突，保持并存
> 	2. 如果from的谓词类型是B或C，则以B/C为基础，在满足B/C的条件下加上与另一个重叠的部分，不冲突，保持并存
> 2. 如果A是父类，B继承A，C继承B，则C会把B中的相同成员谓词override掉，而不是共存
> 3. 对于多重继承，C同时继承A和B，如果A和B的成员谓词有重合，则C必须override这个谓词
> 
> 
> 例如：
> 
> 
> 
> ```
> class OneTwoThree extends int {
>   OneTwoThree() { // 特征谓词
>     this = 1 or this = 2 or this = 3
>   }
>   string getAString() { // 成员谓词
>     result = "One, two or three: " + this.toString()
>   }
> }
> 
> class OneTwo extends OneTwoThree {
>   OneTwo() {
>     this = 1 or this = 2
>   }
>   override string getAString() {
>     result = "One or two: " + this.toString()
>   }
> }
> 
> from OneTwoThree o
> select o, o.getAString()
> 
> /* result:
> o	getAString() result
> 1	One or two: 1
> 2	One or two: 2
> 3	One, two or three: 3
> 
> // 理解：onetwothree类定义了1 2 3，onetwo重构了onetwothree中1和2的成员谓词。因此onetwothree o中有3个，其中的1和2使用onetwo的成员谓词，3使用onetwothree的成员谓词
> */
> 
> ```
> 
> 情况1: 在这个基础上加上另一个类别（重要），A\-\>B, A\-\>C
> 
> 
> ![image-20241025105910532](https://serverless-page-bucket-lv779z7b-1307395653.cos.ap-shanghai.myqcloud.com/picgo/202410251059602.png)
> 
> 
> 
> ```
> class TwoThree extends OneTwoThree{
>   TwoThree() {
>     this = 2 or this = 3
>   }
>   override string getAString() {
>     result = "Two or three: " + this.toString()
>   }
> }
> 
> /* 
> command:
> from OneTwoThree o
> select o, o.getAString()
> 
> result:
> o	getAString() result
> 1	One or two: 1
> 2	One or two: 2
> 2	Two or three: 2
> 3	Two or three: 3
> // 理解：twothree和onetwo重合了two，但是不像其他ool，ql并不会冲突，而是并存。
> 
> ---
> command:
> from OneTwo o
> select o, o.getAString()
> 
> result:
> 1	One or two: 1
> 2	One or two: 2
> 2	Two or three: 2
> // 理解：twothree和onetwo都重构了其中的2，由于ql不会冲突，所以并存。由于o的类型是onetwo，因此"地基"是1和2，然后再加上twothree重构的2
> 
> ---
> command:
> from TwoThree o
> select o, o.getAString()
> 
> result:
> 2	One or two: 2
> 2	Two or three: 2
> 3	Two or three: 3
> // 理解： twothree和onetwo都重构了2，由于ql不会冲突，会并存。由于o的类型是twothree，所以“地基”是2和3，然后再加上onetwo重构的2
> */
> 
> ```
> 
> 情况2: A\-\>B\-\>C（继承链）
> 
> 
> ![image-20241025105937484](https://serverless-page-bucket-lv779z7b-1307395653.cos.ap-shanghai.myqcloud.com/picgo/202410251059556.png)
> 
> 
> 
> ```
> class Two extends TwoThree {
>     Two() {
>         this = 2
>     }
>     override string getAString() {
>         result = "Two: " + this.toString()
>     }
>   }
> 
> from TwoThree o
> select o, o.getAString()
> 
> /* result:
> o	getAString() result
> 1	One or two: 2
> 2	Two: 2
> 3	Two or three: 3
> 
> // 理解：在上面的例子的基础上，Two重构了twothree中的成员谓词，因此与twothree不是共存关系
> */
> 
> from OneTwo o
> select o, o.getAString()
> /* result:
> o	getAString() result
> 1	One or two: 1
> 2	One or two: 2
> 3	Two: 2
> 
> // 理解：在之前例子的基础上，OneTwo和TwoThree共存，但是Two把TwoThree中的一部分给override了（即Two和TwoThree并不是共存关系）
> */
> 
> ```
> 
> 阶段总结：根据上面这么多例子的学习，总结归纳起来其实很简单，核心要义就是搞清楚“继承链关系”。如果两个类别是继承的同一个父类，那么他们两个的结果共存；如果两个类是从属关系（父与子），那么子类覆盖父类对应的部分。
> 
> 
> 例如上面的例子中，OneTwo和TwoThree是并存关系，同时继承OneTwoThree，所以他们的结果共存，不冲突；TwoThree和Two是从属关系，所以根据最子类优先原则，覆盖对应TwoThree中的内容（Two也间接继承OneTwoThree，所以对所有父类包括OneTwoThree造成影响）。
> 
> 
> 情况3: 多重继承
> 
> 
> ![image-20241025105948674](https://serverless-page-bucket-lv779z7b-1307395653.cos.ap-shanghai.myqcloud.com/picgo/202410251059733.png)
> 
> 
> 
> ```
> class Two extends OneTwo, TwoThree {
>     Two() {
>         this = 2
>     }
>     override string getAString() {
>         result = "Two: " + this.toString()
>     }
>   }
> // 解释1：Two同时继承TwoThree和OneTwo，如果不写条件谓词，则默认为同时满足两个父类条件，如果写，则范围也要小于等于这个交集范围。
> // 解释2：如果多重继承的父类中同一名称的成员谓词有多重定义，则必须覆盖这些定义避免歧义。在这里的Two的getAString()是不能省略的
> 
> from OneTwoThree o
> select o, o.getAString()
> /* result:
> o	getAString() result
> 1	One or two: 1
> 2	Two: 2
> 3	Two or three: 3
> 
> // 理解：由于two与onetwo和twothree是父子关系，因此直接把共有的two全部覆盖，不是并存关系
> */
> 
> ```


在这个基础上再去创建一个谓词，用于判断是否是秃头isBald



```
predicate isBald(Person p) {
    not exists(string c | p.getHairColor() = c)    // 不加not表示某人有头发
}

// 获得最终结果，允许进入北方的南方秃头
from Southerner s 
where s.isAllowedIn("north") and isBald(s)
select s, s.getAge()

```

