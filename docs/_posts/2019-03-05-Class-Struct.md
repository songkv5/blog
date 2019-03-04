---
author :  willis
date: 2019-03-05 00:00:00
---
### 类文件结构

类文件结构，从前到后包括
1. 魔数CAFEBABE（4个字节）
2. 次版本号（2个字节）
3. 主版本号（2个字节）
4. 常量池入口（前两个字节代表常量池容量，后面紧跟着常量的内容

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

在class文件中，常量池的每一项常量都是一个表，目前共有14中表结构，他们有一个共同点就是，第一个字节是标志位（tag），用来标识常量类型

*以上14中常量类型，都有自己特有的结构，例如 **CONSTANT_Class_info**的结构为*

类型|名称|数量
:---|:---|:---
u1|tag|1
u2|name_index|1

tag理解同上，都是标志一个常量类型。
***name_index*** 是索引值，指向另一个常量。如果name_index的值是4，就意味着它执行常量池中第4个常量，只要看第四个常量是什么即可。


5. 访问标志（2个字节），包括了这个class是类还是接口，是否是public，是否为abstract，是否是final等
6. 类索引（2个字节）、父类索引（2个字节）、接口索引集合（一组2个字节的数据集合）。类索引用于确定这个类的全限定名，父类索引用于确定父类的全限定名。接口索引集合用来描述这个类实现了哪些接口
7. 字段表集合。用于描述接口或类中申明的变量

***字段表结构***

类型|名称|数量|说明
:---|:---|:---|
u2|access_flags|1|用来标识访问属性，是否为public等
u2|name_index|1|指的值没有类型和参数修饰的方法或者字段的名称，如main()方法的简单名字就是main
u2|descriptor_index|1|描述字段的数据类型、方法的参数列表，参考[描述符](#描述符)
u2|attributes_count|1|记录一些额外信息
attribute_info|attributes|attributes_count|-

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

8. 方法表集合

---
1. 深入理解Java虚拟机》第二版


