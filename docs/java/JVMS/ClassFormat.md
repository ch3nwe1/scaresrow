# Class Format

本文为字节码文件规范,以一个实例来研究java编译的class文件中每一个字节码进行分析其具体内容是什么!!!参考Oracle官网的Java虚拟机规范

## 源文件

```java
public class Student implements Serializable, Cloneable {

    private long id;
    private String name;

    public long getId() {
        return id;
    }

    public void setId(long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Student() {
    }

    public Student(long id, String name) {
        this.id = id;
        this.name = name;
    }
    
}
```

## 字节码文件

```
CA FE BA BE 00 00 00 34 00 21 09 00 04 00 1C 09 
00 04 00 1D 0A 00 05 00 1E 07 00 1F 07 00 20 01 
00 02 69 64 01 00 01 4A 01 00 04 6E 61 6D 65 01 
00 12 4C 6A 61 76 61 2F 6C 61 6E 67 2F 53 74 72 
69 6E 67 3B 01 00 05 67 65 74 49 64 01 00 03 28 
29 4A 01 00 04 43 6F 64 65 01 00 0F 4C 69 6E 65 
4E 75 6D 62 65 72 54 61 62 6C 65 01 00 12 4C 6F 
63 61 6C 56 61 72 69 61 62 6C 65 54 61 62 6C 65 
01 00 04 74 68 69 73 01 00 16 4C 63 6F 6D 2F 61 
73 6D 2F 74 65 73 74 2F 53 74 75 64 65 6E 74 3B 
01 00 05 73 65 74 49 64 01 00 04 28 4A 29 56 01 
00 07 67 65 74 4E 61 6D 65 01 00 14 28 29 4C 6A 
61 76 61 2F 6C 61 6E 67 2F 53 74 72 69 6E 67 3B 
01 00 07 73 65 74 4E 61 6D 65 01 00 15 28 4C 6A 
61 76 61 2F 6C 61 6E 67 2F 53 74 72 69 6E 67 3B 
29 56 01 00 06 3C 69 6E 69 74 3E 01 00 03 28 29 
56 01 00 16 28 4A 4C 6A 61 76 61 2F 6C 61 6E 67 
2F 53 74 72 69 6E 67 3B 29 56 01 00 0A 53 6F 75 
72 63 65 46 69 6C 65 01 00 0C 53 74 75 64 65 6E 
74 2E 6A 61 76 61 0C 00 06 00 07 0C 00 08 00 09 
0C 00 17 00 18 01 00 14 63 6F 6D 2F 61 73 6D 2F 
74 65 73 74 2F 53 74 75 64 65 6E 74 01 00 10 6A 
61 76 61 2F 6C 61 6E 67 2F 4F 62 6A 65 63 74 00 
21 00 04 00 05 00 00 00 02 00 02 00 06 00 07 00 
00 00 02 00 08 00 09 00 00 00 06 00 01 00 0A 00 
0B 00 01 00 0C 00 00 00 2F 00 02 00 01 00 00 00 
05 2A B4 00 01 AD 00 00 00 02 00 0D 00 00 00 06 
00 01 00 00 00 09 00 0E 00 00 00 0C 00 01 00 00 
00 05 00 0F 00 10 00 00 00 01 00 11 00 12 00 01 
00 0C 00 00 00 3E 00 03 00 03 00 00 00 06 2A 1F 
B5 00 01 B1 00 00 00 02 00 0D 00 00 00 0A 00 02 
00 00 00 0D 00 05 00 0E 00 0E 00 00 00 16 00 02 
00 00 00 06 00 0F 00 10 00 00 00 00 00 06 00 06 
00 07 00 01 00 01 00 13 00 14 00 01 00 0C 00 00 
00 2F 00 01 00 01 00 00 00 05 2A B4 00 02 B0 00 
00 00 02 00 0D 00 00 00 06 00 01 00 00 00 11 00 
0E 00 00 00 0C 00 01 00 00 00 05 00 0F 00 10 00 
00 00 01 00 15 00 16 00 01 00 0C 00 00 00 3E 00 
02 00 02 00 00 00 06 2A 2B B5 00 02 B1 00 00 00 
02 00 0D 00 00 00 0A 00 02 00 00 00 15 00 05 00 
16 00 0E 00 00 00 16 00 02 00 00 00 06 00 0F 00 
10 00 00 00 00 00 06 00 08 00 09 00 01 00 01 00 
17 00 18 00 01 00 0C 00 00 00 33 00 01 00 01 00 
00 00 05 2A B7 00 03 B1 00 00 00 02 00 0D 00 00 
00 0A 00 02 00 00 00 18 00 04 00 19 00 0E 00 00 
00 0C 00 01 00 00 00 05 00 0F 00 10 00 00 00 01 
00 17 00 19 00 01 00 0C 00 00 00 59 00 03 00 04 
00 00 00 0F 2A B7 00 03 2A 1F B5 00 01 2A 2D B5 
00 02 B1 00 00 00 02 00 0D 00 00 00 12 00 04 00 
00 00 1B 00 04 00 1C 00 09 00 1D 00 0E 00 1E 00 
0E 00 00 00 20 00 03 00 00 00 0F 00 0F 00 10 00 
00 00 00 00 0F 00 06 00 07 00 01 00 00 00 0F 00 
08 00 09 00 03 00 01 00 1A 00 00 00 02 00 1B
```

## 反编译文件

```java
Classfile /home/amarsoft/workspace/app/asm-test/target/classes/com/asm/test/Student.class
  Last modified 2020-8-26; size 847 bytes
  MD5 checksum 7fd5f26737b6e1effb65d05864328b9c
  Compiled from "Student.java"
public class com.asm.test.Student
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Fieldref           #4.#28         // com/asm/test/Student.id:J
   #2 = Fieldref           #4.#29         // com/asm/test/Student.name:Ljava/lang/String;
   #3 = Methodref          #5.#30         // java/lang/Object."<init>":()V
   #4 = Class              #31            // com/asm/test/Student
   #5 = Class              #32            // java/lang/Object
   #6 = Utf8               id
   #7 = Utf8               J
   #8 = Utf8               name
   #9 = Utf8               Ljava/lang/String;
  #10 = Utf8               getId
  #11 = Utf8               ()J
  #12 = Utf8               Code
  #13 = Utf8               LineNumberTable
  #14 = Utf8               LocalVariableTable
  #15 = Utf8               this
  #16 = Utf8               Lcom/asm/test/Student;
  #17 = Utf8               setId
  #18 = Utf8               (J)V
  #19 = Utf8               getName
  #20 = Utf8               ()Ljava/lang/String;
  #21 = Utf8               setName
  #22 = Utf8               (Ljava/lang/String;)V
  #23 = Utf8               <init>
  #24 = Utf8               ()V
  #25 = Utf8               (JLjava/lang/String;)V
  #26 = Utf8               SourceFile
  #27 = Utf8               Student.java
  #28 = NameAndType        #6:#7          // id:J
  #29 = NameAndType        #8:#9          // name:Ljava/lang/String;
  #30 = NameAndType        #23:#24        // "<init>":()V
  #31 = Utf8               com/asm/test/Student
  #32 = Utf8               java/lang/Object
{
  public long getId();
    descriptor: ()J
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: getfield      #1                  // Field id:J
         4: lreturn
      LineNumberTable:
        line 9: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/asm/test/Student;

  public void setId(long);
    descriptor: (J)V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=3, args_size=2
         0: aload_0
         1: lload_1
         2: putfield      #1                  // Field id:J
         5: return
      LineNumberTable:
        line 13: 0
        line 14: 5
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       6     0  this   Lcom/asm/test/Student;
            0       6     1    id   J

  public java.lang.String getName();
    descriptor: ()Ljava/lang/String;
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: getfield      #2                  // Field name:Ljava/lang/String;
         4: areturn
      LineNumberTable:
        line 17: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/asm/test/Student;

  public void setName(java.lang.String);
    descriptor: (Ljava/lang/String;)V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: putfield      #2                  // Field name:Ljava/lang/String;
         5: return
      LineNumberTable:
        line 21: 0
        line 22: 5
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       6     0  this   Lcom/asm/test/Student;
            0       6     1  name   Ljava/lang/String;

  public com.asm.test.Student();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #3                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 24: 0
        line 25: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/asm/test/Student;

  public com.asm.test.Student(long, java.lang.String);
    descriptor: (JLjava/lang/String;)V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=4, args_size=3
         0: aload_0
         1: invokespecial #3                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: lload_1
         6: putfield      #1                  // Field id:J
         9: aload_0
        10: aload_3
        11: putfield      #2                  // Field name:Ljava/lang/String;
        14: return
      LineNumberTable:
        line 27: 0
        line 28: 4
        line 29: 9
        line 30: 14
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      15     0  this   Lcom/asm/test/Student;
            0      15     1    id   J
            0      15     3  name   Ljava/lang/String;
}
SourceFile: "Student.java"
```



## 字节码文件结构

```
ClassFile {
    u4 magic;
    u2 minor_version;
    u2 major_version;
    u2 constant_pool_count;
    cp_info constant_pool[constant_pool_count-1];
    u2 access_flags;
    u2 this_class;
    u2 super_class;
    u2 interfaces_count;
    u2 interfaces[interfaces_count];
    u2 fields_count;
    field_info fields[fields_count];
    u2 methods_count;
    method_info methods[methods_count];
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

其中u1,u2,u4分别代表1,2,4个字节

### magic

magic是字节码文件的魔法数,共有四个字节,`CA FE BA BE`,这是Java字节码文件的固定值

### minor_version

占用两个字节, 值为`00 00`,代表主版本号,目前的java版本直到jdk11都是0

### major_version

次版本号`00 34`,转化为十进制为52,代表jdk1.8版本

### constant_pool_count

常量池数量,占用两个字节,值为`00 21`,转化为十进制为33,代表常量池的数量是32 + 1

### cp_info(constant_pool_info)

常量池信息字节是不固定的,但也有其规律可循,总体结构为

```
cp_info {
    u1 tag;
    u1 info[];
}
```

1个字节的tag, info数组随着tag的不同其信息也不同,下面会一一介绍,首先来接收tag的类型

|        Constant Type        | Value |
| :-------------------------: | :---: |
|       CONSTANT_Class        |   7   |
|      CONSTANT_Fieldref      |   9   |
|     CONSTANT_Methodref      |  10   |
| CONSTANT_InterfaceMethodref |  11   |
|       CONSTANT_String       |   8   |
|      CONSTANT_Integer       |   3   |
|       CONSTANT_Float        |   4   |
|        CONSTANT_Long        |   5   |
|       CONSTANT_Double       |   6   |
|    CONSTANT_NameAndType     |  12   |
|        CONSTANT_Utf8        |   1   |
|    CONSTANT_MethodHandle    |  15   |
|     CONSTANT_MethodType     |  16   |
|   CONSTANT_InvokeDynamic    |  18   |

下面介绍每一个tag对应的info数组

```json
//占用三个字节
CONSTANT_Class_info {
    u1 tag;
    u2 name_index; # 指向常量池表中的索引
}
```

```json
//占用五个字节
CONSTANT_Fieldref_info {
 u1 tag;
 u2 class_index; //所属类的索引
 u2 name_and_type_index; //名字和类型的索引
}
CONSTANT_Methodref_info {
 u1 tag;
 u2 class_index;
 u2 name_and_type_index;
}
CONSTANT_InterfaceMethodref_info {
 u1 tag;
 u2 class_index;
 u2 name_and_type_index;
}
```

```json
//占用三个字节
CONSTANT_String_info {
 u1 tag;
 u2 string_index; //字符串索引
}
```

```java
//占用五个字节
CONSTANT_Integer_info {
 u1 tag;
 u4 bytes;
}
CONSTANT_Float_info {
 u1 tag;
 u4 bytes; //占用四个字节的byte数组, 代表其真实值
}
```

```java
//5个字节
CONSTANT_NameAndType_info {
 u1 tag;
 u2 name_index; // 名称索引
 u2 descriptor_index; // 描述符索引
}
```

```java
//非固定长度
CONSTANT_Utf8_info {
 u1 tag;
 u2 length;
 u1 bytes[length];
}
```

```java
// 4个字节
CONSTANT_MethodHandle_info {
 u1 tag;
 u1 reference_kind;
 u2 reference_index;
}
```

```java
//3个字节
CONSTANT_MethodType_info {
 u1 tag;
 u2 descriptor_index;
}
```

```java
//5个字节
CONSTANT_InvokeDynamic_info {
 u1 tag;
 u2 bootstrap_method_attr_index;
 u2 name_and_type_index;
}
```

```java
//9个字节
CONSTANT_Long_info {
 u1 tag;
 u4 high_bytes;
 u4 low_bytes;
}
CONSTANT_Double_info {
 u1 tag;
 u4 high_bytes;
 u4 low_bytes;
}
```



紧随常量池数量后面的字节是**`09`**, 代表是一个`CONSTANT_Fieldref`,其格式包含1个字节的tag, 2个字节的`class_index`(**`00 04`**)和2个字节的`name_and_type_index`(**`00 1C`**),所以常量池表的第一个常量分析完毕,正好与反编译文件对应

> #1 = Fieldref           #4.#28 // #1 代表第一个索引,#4,#28现在还不清楚,先不要着急,慢慢的分析

### access_flags(u2)

继常量池后的是2个字节的访问标识符



| flag_name      | VALUE  | Interpretation                                 |
| -------------- | ------ | ---------------------------------------------- |
| ACC_PUBLIC     | 0x0001 | 声明public,可以在包的外面访问                  |
| ACC_FINAL      | 0x0010 | 不能被继承                                     |
| ACC_SUPER      | 0x0020 | 当被invokespecial指定调用时,针对特殊的父类方法 |
| ACC_INTERFACE  | 0x0200 | 声明为一个接口,二不是一个类                    |
| ACC_ABSTRACT   | 0x0400 | 抽象类,不能被实例化                            |
| ACC_SYNTHETIC  | 0x1000 | 不会在源代码中出现,通过编译生成                |
| ACC_ANNOTATION | 0x2000 | 声明为注解                                     |
| ACC_ENUM       | 0x4000 | 声明为一个枚举类                               |

### this_class(u2)

2个字节的索引,指向常量池中的CONSTANT_Class_info,class_info的结构上文已经提过,在这里再说一下,class_info包含1个字节的tag和2个字节的索引指向常量池中CONSTANT_Utf8_info 常量

```java
CONSTANT_Class_info {
 	u1 tag; // (7)
 	u2 name_index;
}

CONSTANT_Utf8_info {
 u1 tag; //(1)
 u2 length;
 u1 bytes[length];
}
```

### super_class(u2)

2个字节的索引,指向常量池中的CONSTANT_Class_info,class_info

### interfaces_count(u2)

2个字节,其值代表类实现接口的数量

### interfaces[interfaces_count]

按照源码中实现接口的顺序,每个接口的信息占用两个字节,其值指向常量池中的索引,(CONSTANT_Class_info)

### fields_count（u2）

代表类属性数量，占用两个字节

### field_info 

fields[fields_count]

### methods_count(u2)

方法数量

### method_info 

methods[methods_count]

### attributes_count

类属性数量

### attribute_info

attributes[attributes_count];

#### SOURCE_FILE

```json
SourceFile_attribute {
 u2 attribute_name_index; // 属性名索引
 u4 attribute_length; // 属性值长度 固定为2
 u2 sourcefile_index; // 指向常量池的索引
}
```

#### INNER_CLASSES

```json
InnerClasses_attribute {
 u2 attribute_name_index;
 u4 attribute_length;
 u2 number_of_classes;
 { 
 	u2 inner_class_info_index; # 内部类索引
 	u2 outer_class_info_index; # 外部类索引
 	u2 inner_name_index;  # 内部类名称索引
 	u2 inner_class_access_flags; # 访问标识符
 } classes[number_of_classes];
}
```

####  RuntimeVisibleAnnotations

```json
RuntimeVisibleAnnotations_attribute {
 u2 attribute_name_index; # 属性名索引，指向常量池 固定为 RuntimeVisibleAnnotations
 u4 attribute_length; # 属性长度 所有注解信息占用的字节数
 u2 num_annotations; # 注解数量
 annotation annotations[num_annotations];# 注解数组，元素是注解对象
}
```

```json
# 注解对象
annotation {
 u2 type_index;# 类型索引，指向常量池UTF-8索引，标识字段描述符
 u2 num_element_value_pairs;
 { u2 element_name_index;
 element_value value;
 } element_value_pairs[num_element_value_pairs];
}
```

