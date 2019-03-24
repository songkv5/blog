---
author :  willis
date: 2019-03-05 00:00:00
---
### 类文件结构

整体结构如下（按顺序）

类型|名称|数量|描述
:---|:---|:---|:---
u4|magic|1|魔数
u2|minor_version|1|次版本号
u2|major_version|1|主版本号
u2|constant_pool_count|1|常量池常量数量
cp_info|constant_pool|constant_pool_count - 1|常量信息
u2|access_flag|1|访问标志
u2|this_class|1|类索引
u2|super_class|1|父类索引
u2|interface_count|1|实现的接口数量
u2|interfaces|interface_count|接口索引表
u2|field_count|1|字段表的字段数量
field_info|field|field_count|字段表项
u2|method_count|1|方法数量
method_info|methods|method_count|方法表的一个个方法项
u2|attribute_count|1|属性个数
attribute_info|attributes|attribute_count|属性项。参考[属性表](#属性表集合)

类文件结构，从前到后包括
1.、魔数CAFEBABE（4个字节）
2、次版本号（2个字节）
3.、主版本号（2个字节）
4.、常量池入口（前两个字节代表常量池容量，后面紧跟着常量的内容

**常量表类型**如下

类型|标志的值（每一项常量的第一个字节的值）|描述
:---|:---|:---
CONSTANT_Utf8_info|1|TF8编码的字符串
CONSTANT_Integer_info|3|整形字面量
CONSTANT_Float_info|4|浮点型字面量
CONSTANT_Long_info|5|长整形字面量
CONSTANT_Double_info|6|双精度浮点型字面量
CONSTANT_Class_info|7|类或接口的符号引用
CONSTANT_String_info|8|字符串类型字面量
CONSTANT_Fieldref_info|9|字段的符号引用
CONSTANT_Methodref_info|10|类中方法的符号引用
CONSTANT_InterfaceMethodref_info|11|接口方法的符号引用
CONSTANT_NameAndType_info|12|字段或方法的部分符号引用
CONSTANT_MethodHandle_info|15|表示方法句柄
CONSTANT_MethodType_info|16|标识方法类型
CONSTANT_InvokeDynamic_info|18|表示一个动态方法调用点

<u>在class文件中，常量池的每一项常量都是一个表，目前共有14中表结构，他们有一个共同点就是，第一个字节是标志位（tag），用来标识常量类型</u>

*以上14中常量类型，都有自己特有的结构，例如 **CONSTANT_Class_info**的结构为*

类型|名称|数量
:---|:---|:---
u1|tag|1
u2|name_index|1

tag理解同上，都是标志一个常量类型。
***name_index*** 是索引值，指向另一个常量。如果name_index的值是4，就意味着它执行常量池中第4个常量，只要看第四个常量是什么即可。


5、访问标志（2个字节），紧跟常量池之后。包括了这个class是类还是接口，是否是public，是否为abstract，是否是final等

其中，***类的访问标志的值***分别代表的意义如下

标志|标志值|含义
:---|:---|:---
ACC_PUBLIC|0x0001|是否public
ACC_FINAL|0x0010|是否final
ACC_SUPER|0x0020|--
ACC_INTERFACE|0x0200|标志着是一个接口
ACC_ABSTRTACT|0x0400|是否为abstract类型，接口和抽象类都为真
ACC_SYNTHETIC|0x1000|标志由非用户代码产生
ACC_ANNOTATION|0x2000|标志这是一个注解
ACC_NUME|0x4000|字段是否enum

6、类索引（2个字节）、父类索引（2个字节）、接口索引集合（一组2个字节的数据集合）。类索引用于确定这个类的全限定名，父类索引用于确定父类的全限定名。接口索引集合用来描述这个类实现了哪些接口
7、字段表集合。用于描述接口或类中申明的变量。第一个U2类型的数据代表字段的数量。后面是字段表

***字段表结构***

类型|名称|数量|说明
:---|:---|:---|
u2|access_flags|1|访问标志。用来标识访问属性，是否为public等
u2|name_index|1|名字索引，索引值，指向某个常量池的常量，该常量描述了代表字段的简单名称，如main()方法的简单名字就是main
u2|descriptor_index|1|索引值，指向常量池的某个常量，该常量是字段和方法的描述符。描述字段的数据类型、方法的参数列表，参考[描述符](#描述符)
u2|attributes_count|1|记录一些额外信息
attribute_info|attributes|attributes_count|-

**其中，顺序是有严格要求，如上表**

其中，***访问标志的值***分别代表的意义如下

标志|标志值|含义
:---|:---|:---
ACC_PUBLIC|0x0001|字段是否public
ACC_PRIVATE|0x0002|字段是否private
ACC_PROTECTED|0x0004|字段是否protected
ACC_STATIC|0x0008|字段是否static
ACC_FINAL|0x0010|字段是否final
ACC_VOLATILE|0x0040|字段是否volatile
ACC_TRANSIENT|0x0080|字段是否transient
ACC_SYNTHETIC|0x1000|字段是否由编译器自动产生
ACC_NUME|0x4000|字段是否enum


***描述符***

标识字符|含义
:---|:---
B|基本类型byte
C|基本类型char
D|基本类型double
F|基本类型float
I|基本类型int
J|基本类型long
S|基本类型short
Z|基本类型boolean
V|特殊类型void
L|对象类型。如Ljava/lang/Object

<u>数组描述符：一位数组用前缀"["描述，二维数组用“[[”前缀描述；</u>

<u>方法描述符：先参数列表，后返回值类型来描述。如java.lang.String toString()的描述符就是()Ljava/lang/String;</u>

8、方法表集合

class文件中堆方法的描述与对字段的描述几乎采用了完全一致的方式，方法表的结构同字段表一样。

***方法表结构***

类型|名称|数量|说明
:---|:---|:---|:---
u2|access_flags|1|访问标志。用来标识访问属性，是否为public等
u2|name_index|1|名字索引，索引值，指向某个常量池的常量，该常量描述了代表方法的简单名称，如main()方法的简单名字就是main
u2|descriptor_index|1|索引值，指向常量池的某个常量，该常量是字段和方法的描述符。描述字段的数据类型、方法的参数列表，参考[描述符](#描述符)
u2|attributes_count|1|记录一些额外信息
attribute_info|attributes|attributes_count|-

**其中，顺序是有严格要求，如上表**

***方法表的访问标志***如下

标志|标志值|含义
:---|:---|:---
ACC_PUBLIC|0x0001|方法是否public
ACC_PRIVATE|0x0002|方法是否private
ACC_PROTECTED|0x0004|方法是否protected
ACC_STATIC|0x0008|方法是否static
ACC_FINAL|0x0010|方法是否final
ACC_SYNCHRONIZED|0x0020|方法是否synchronized
ACC_BRIDGE|0x0040|方法是否由编译器产生的桥接方法
ACC_VARARGS|0x0080|方法是否接受不定参数
ACC_NAVITE|0x0100|方法是否navite
ACC_ABSTRACT|0x0400|方法是否abstract
ACC_STRICTFP|0x0800|方法是否为strictfp
ACC_SYNTHETIC|0x1000|字段是否由编译器自动产生的

通过方法表，可以描述方法，方法的代码块在哪？其实，方法被编译成自己吗指令后，存放在方法属性表（attributes_count、attribute_info）中一个名字为“Code”的属性中，具体可以参考[属性表](#属性表集合)

9、属性表集合
属性表集合多次出现，在方法表、字段表、class文件中都有携带自己的属性表。与class文件其他项不同，属性表不要求各个属性表有严格的顺序

***虚拟机规范属性***

属性|使用位置|含义
:---|:---|:---
Code|方法表|java编译成的字节码指令
ConstantValue|字段表|final关键字定义的常量值
Deprecated|类、方法表、字段表|被声明为deprecated的方法和字段
Exceptions|方法表|方法抛出的异常
EnclosingMethod|类文件|仅当一个类为局部类或者匿名类时才能拥有这个属性，这个属性用于标志这个类所在的外围方法
InnerClass|类文件|内部类列表
lineNumberTable|Code属性|java源码的行号与字节码指令的对应关系
LocalVariableTable|Code属性|方法的局部变量描述
StackMapTable|Code属性|JDK1.6中新增的属性，供新的类型检查验证器检查和处理目标方法的局部变量和操作数栈所需要的类型是否匹配
Signatrue|类、方法表、字段表|--
SourceFile|类文件|记录源文件名称
SourceDebugExtention|类文件|JDK新增的属性，用于存储额外的调试信息
Synthetic|类、方法表、字段表|标识方法或字段为编译器自动生成的
LocalVariavleTypeTable|类|JDK1.5中新增的属性
RuntimeVisibleAnnotations|类、方法表、字段表|JDK1.5新增的属性，为动态注解提供支持，用于表明哪些注解是运行时（运行时进行反射调用）是可见的
RuntimeInvisibleAnnotations|类、方法表、字段表|和RuntimeVisibleAnnotation想法，用于表明哪些注解是运行时是不可见的
RuntimeVisibleParameterAnnotations|方法表|JDK1.5新增的属性，作用与RuntimeVisibleAnnotations相似，只不过作用对象为方法参数
RuntimeInVisibleParameterAnnotations|方法表|JDK1.5新增的属性，作用与RuntimeInVisibleAnnotations相似，只不过作用对象为方法参数
AnnotationDefault|方法表|JDK1.5新增的属性，用于记录注解元素的默认值
BootstrapMethod|类文件|JDK1.7新增属性，用于保存invokedynamic指令引用的引导方法限定符

***Code属性表***
属性名称|类型|数量|说明
:---|:---|:---|:---
attribute_name_index|u2|1|指向Constant_Utf8_info型常量索引，固定为“Code”
attribute_length|u4|1|指示了属性值的长度
max_stack|u2|1|代表了操作数栈深度的最大值
max_locals|u4|1|代表了局部变量表所需要的存储空间
code_length|u4|1|字节码长度
code|u1|code_length|保存字节码指令，每个code就是一个字节码指令，共有code_length个
exception_table_length|u2|1|异常处理表


10、

---
1. 深入理解Java虚拟机》第二版


