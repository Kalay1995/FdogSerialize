

# FdogSerialize

​		**FdogSerialize**是一个用于C++序列化和反序列化快速互转开源库，采用**非入侵方式**，无需在原有结构体上进行修改，**支持别名**，**最少三行代码**即可完成转换使得它足够方便。

---

### 目录

- [FdogSerialize](#FdogSerialize)
    - [目录](#%E7%9B%AE%E5%BD%95)
    - [背景](#%E8%83%8C%E6%99%AF)
    - [类型支持](#%E7%B1%BB%E5%9E%8B%E6%94%AF%E6%8C%81)
    - [使用说明](#%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E)
    - [作者](#%E4%BD%9C%E8%80%85)
    - [许可](#%E8%AE%B8%E5%8F%AF)
    - [其他](#%E5%85%B6%E4%BB%96)

---

### 背景

​		经常使用java或者go的人应该知道这些语言在进行序列化和反序列化是很容易的，但是对于伟大的C++而言，这是困难的，根本原因是C++不支持反射，基于c++的语言哲学，C++宁死也要坚守的zero overhead，不为用不到的特性付出任何代价，不管这个代价有多小，也不管是怎样的语言特性，都不会妥协。

​		虽然C++不支持反射，但是我们依旧可以通过自己的方式实现序列化与反序列化，记得我在大二时用C++写的一个client-server小项目，自己规定了传输的数据格式（当时觉得自己解析Json很麻烦），第一个字段应该是什么，第二个字段应该是什么，正是因为没有方便的函数进行转换，我每次都需要使用非常繁琐的代码去拼出一个可以传递的字符串，是的，这样确实可以完成我想要的功能，但是我自己定的数据格式只适合自己用，这种方式长期必然行不通，而大多数人使用JSON和XML这两种数据格式来保存对象，如果我的项目想要使用这种大众化的数据格式，我又将重构我的代码。

​		我试着在github寻找一些用于C++的序列化与反序列的库，看看有没有什么办法可以帮助我快速把对象转变成JSON，我找到了一些类似的库，但是有一些设计不太合我心意。



不太和我心意的设计：

* 使用者需要添加过多代码
- 采用入侵方式，需要改变原有的结构体

* 不需要入侵，但是在注册的时候需要一个一个指定类型

* 只支持基础类型组成的结构体转换
- 不支持别名（由于go的特性，在go中经常被使用）
- 不支持指针类型（如果原项目中存在指针类型便需要指针类型）



既然不满意，就自己动手写一个吧，说实话还是有点困难的，虽然说C++中应该尽可能的少用宏，但就目前来说C++没有反射机制，编译后不保存结构体的元信息来说，使用宏是无法绕开的。

暂时分成五期：

1. 支持由基础类型组合成结构体和Json互转【完成】
2. 支持由复杂类型组合成结构体和Json互转【完成】
3. 支持类和Json互转【未完成】
4. 支持vector，map，list等数据类型和Json互转【未完成】
5. 支持XML数据格式【未完成】



杂项支持：

1. 支持别名【未完成】
1. 

暂时就想到怎么多，欢迎补充！

---

### 类型支持

| 序号 |      基础类型      |          说明          | 序号 |        基础类型        |       说明       |
| :--: | :----------------: | :--------------------: | :--: | :--------------------: | :--------------: |
|  1   |        bool        |        布尔类型        |  11  |      unsigned int      |    无符号整型    |
|  2   |        char        |      无符号字符型      |  12  |        long int        |      长整型      |
|  3   |    signed char     |      有符号字符型      |  13  |    signed long int     |   有符号长整型   |
|  4   |   unsigned char    |      无符号字符型      |  14  |   unsigned long int    |   无符号长整型   |
|  5   |       char *       | 字符指针（字符串类型） |  15  |     long long int      |     超长整型     |
|  6   |     short int      |         短整型         |  16  |  signed long long int  |  有符号超长整型  |
|  7   |  signed short int  |      有符号短整型      |  17  | unsigned long long int |  无符号超长整型  |
|  8   | unsigned short int |      无符号短整型      |  18  |         float          |  单精度浮点类型  |
|  9   |        int         |          整型          |  19  |         double         |  双进度浮点类型  |
|  10  |     signed int     |       有符号整型       |  20  |      long double       | 长双进度浮点类型 |

​	基于C++的基础类型，FdogSerialize支持如上的20种类型，准确说是15种类型，例如int类型等于signed int类型，只要你的结构体中**最基本类型**在这20种范围之内，便可完成转换。



什么是最基本类型？例如：

```cpp
struct headmaster{
	char * name;
	int age;
};

struct student{
	char * name;
	int age;
};

struct teacher{
	char * name;
	int age;
};

struct school{
	char * schoolName;
	headmaster name;
	student stu[1];
	teacher tea[1];
};
```

虽然school这个结构体包含了headmaster，student，teacher三种自定义类型，但追寻本质，他们的底层都属于**最基本类型**。

- headmaster，它包含的最基本类型是char *和int。

- student[]是由多个student组成，而student是由最基本类型char *和int构成。

因此school这个结构体完全可以被正常解析，事实上大多数结构体的最基本类型都在这里了。



你可能会有如下疑问：

1. 为什么char * 类型算作基础类型

​		因为json中除了数值型数据，剩下的便是字符串类型，考虑到字符串使用的频率，再加上C++对于C兼容的原因，决定把他作为最基础类型。

2. char 类型怎么传递？'a' ？

   由于JSON并不支持单引号，所以将使用数值传递并还原，例如：

   ```cpp
   char ch = 'A'; //ASCII码十进制为98
   
   //如果一个包含char的结构体转Json，检测到ch的类型为char将自动转为字符，//注意C++的中的转义
   std::string json = "{\"ch\":98}";
   
   //相对的，如果一个包含char的json想转为struct
   std::string json = "{\"ch\":98}"; //建议使用这种方式
   
   //又或者你不知道字符'A'的ASCII码是多少，那么可以使用如下方式，一般情况下不会遇到自己写json
   std::string json = "{\"ch\":\"char('A')\"}";//库会根据ch的类型，若ch为char类型自动将\"char('A')\"转为98
   
   ```
   
3. 目前支持20个成员的结构体。



---



### 使用说明

1. 不包含结构体嵌套的转换

```cpp
//该结构体包含基础类型
struct baseTest{
    bool bool_b;
    char * char_str;
    char char_c;
    signed char char_s;
    unsigned char char_u;
    int int_i;
    signed int int_s;
    unsigned int int_u;
    short int short_i;
    signed short int short_s;
    unsigned short int short_u;
    long int long_i;
    signed long int long_s;
    unsigned long int long_u;
    long long int long_li;
    signed long long int long_ls;
    unsigned long long int long_lu;
    float float_f;
    double double_d;
    long double double_l;
};

//注册
REGISTEREDMEMBER(baseTest, bool_b, char_str, char_c, char_s, char_u, int_i, int_s, int_u, short_i, short_s, short_u, long_i, long_s, long_u, long_li, long_ls, long_lu, float_f, double_d, double_l);

//调用接口
baseTest test = {true, "花狗", 'A', 'A', 'A', -123, -123, 123, -123, -123, 123, -123, -123, 123, -123, -123, 123, 158.132f, 132.45, 11654.32131};
string json_1;
FdogSerialize(json_1, test);
```

输出：

```JSON
{
    "bool_b":true,
    "char_str":"花狗",
    "char_c":65,
    "char_s":65,
    "char_u":65,
    "int_i":-123,
    "int_s":-123,
    "int_u":123,
    "short_i":-123,
    "short_s":-123,
    "short_u":123,
    "long_i":-123,
    "long_s":-123,
    "long_u":123,
    "long_li":-123,
    "long_ls":-123,
    "long_lu":123,
    "float_f":158.132004,
    "double_d":132.45,
    "double_l":11654.32131
}
```



2. 包含结构体嵌套的转换

相当于普通结构体的转换，包含结构体嵌套的结构体需要多做一步

```cpp
struct headmaster{
	char * name;
	int age;
};

struct student{
	char * name;
	int age;
};

struct school{
	char * schoolName;
	headmaster master;
    int number [3];
	student stu[2];
};

//注册结构体
REGISTEREDMEMBER(headmaster, name, age);
REGISTEREDMEMBER(student, name, age);
REGISTEREDMEMBER(school, schoolName, master, stu, tea);

//如若存在嵌套结构体需要序列化应在此添加
#define REGISTERED_OBJECT_SERIALIZE_ALL\
    REGISTERED_OBJECT_SERIALIZE(headmaster)\
    REGISTERED_OBJECT_SERIALIZE(school)\
    REGISTERED_OBJECT_SERIALIZE(teacher)\
    REGISTERED_OBJECT_SERIALIZE(student)\

//如若存在嵌套结构体数组需要序列化应在此添加
#define REGISTERED_OBJECT_ARRAY_SERIALIZE_ALL\
    REGISTERED_OBJECT_ARRAY_SERIALIZE(headmaster)\
    REGISTERED_OBJECT_ARRAY_SERIALIZE(school)\
    REGISTERED_OBJECT_ARRAY_SERIALIZE(teacher)\
    REGISTERED_OBJECT_ARRAY_SERIALIZE(student)\

//如若存在嵌套结构体需要反序列化应在此添加
#define REGISTERED_OBJECT_DESSERIALIZE_ALL\
    REGISTERED_OBJECT_DESSERIALIZE(headmaster)\
    REGISTERED_OBJECT_DESSERIALIZE(school)\
    REGISTERED_OBJECT_DESSERIALIZE(teacher)\
    REGISTERED_OBJECT_DESSERIALIZE(student)\

//如若存在嵌套结构体数组需要反序列化应在此添加
#define REGISTERED_OBJECT_ARRAY_DESSERIALIZE_ALL\
    REGISTERED_OBJECT_ARRAY_DESSERIALIZE(headmaster)\
    REGISTERED_OBJECT_ARRAY_DESSERIALIZE(school)\
    REGISTERED_OBJECT_ARRAY_DESSERIALIZE(teacher)\
    REGISTERED_OBJECT_ARRAY_DESSERIALIZE(student)\

//调用接口
string json_ = "{\"schoolName\":\"wyai\",\"master\":{\"name\":\"花狗Fdog\",\"age\":21},\"number\":[1,2,3],\"stu\":[{\"name\":\"于静\",\"age\":21},{\"name\":\"二狗\",\"age\":21}]}";
school sch;
FdogDesSerialize(sch, json_);
cout << "schoolName=" << sch.schoolName << endl;
cout << "master.name=" << sch.master.name << endl;
cout << "master.age=" << sch.master.age << endl;
cout << "number[0]=" << sch.number[0] << endl;
cout << "number[1]=" << sch.number[1] << endl;
cout << "number[2]=" << sch.number[2] << endl;
cout << "stu[0]=" << sch.stu[0].name << "," << sch.stu[0].age << endl;
cout << "stu[1]=" << sch.stu[1].name << "," << sch.stu[1].age << endl;
string json_2 = "";
FdogSerialize(json_2, sch);
cout << "json=" << json_2 << endl;
```

输出：

```cpp
schoolName=scjxy
master.name=花狗Fdog
master.age=21
number[0]=1
number[1]=2
number[2]=3
stu[0]=于静,21
stu[1]=二狗,21
json={"schoolName":"scjxy","master":{"name":"花狗Fdog","age":21},"number":[1,2,3],"stu":[{"name":"于静","age":21},{"name":"二狗","age":21}]}
```



---

### 作者

花狗Fdog(张旭)

vx：sui2506897252

EMail：2506897252@qq.com

---

### 许可

该项目签署了Apache-2.0 License，详情请参见LICENSE

Copyright © 2021 花狗Fdog(张旭)

   根据 Apache 许可，版本 2.0（“许可”）获得许可；
   除非遵守许可，否则您不得使用此文件。
   您可以在以下网址获得许可证的副本

       http://www.apache.org/licenses/LICENSE-2.0

   除非适用法律要求或书面同意，否则软件根据许可证分发的分发是按“原样”分发的，不作任何形式的明示或暗示的保证或条件。
   请参阅许可证以了解管理权限的特定语言和许可证下的限制。

---

### 其他

如果对您有帮助的话，请帮我点个star！

如果觉得它好用的话，可以推广给你的朋友们！

---



















