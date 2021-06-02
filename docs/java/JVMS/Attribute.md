## Code

​	Java程序方法体里面的代码经过Javac编译处理后，最终变成字节码指令存储在Code属性内。Code属性出现在方法表的属性集合之中，并非所有的方法都必须存在这个属性，譬如接口和或者抽象类中的方法就不存在Code属性，如果方法内有Code属性存在，他的结构如下表所示。

|      类型      |          名称          |          数量          |
| :------------: | :--------------------: | :--------------------: |
|       u2       |  attribute_name_index  |           1            |
|       u4       |    attribute_length    |           1            |
|       u2       |       max_stack        |           1            |
|       u2       |       max_locals       |           1            |
|       u4       |      code_length       |           1            |
|       u1       |          code          |      code_length       |
|       u2       | exception_table_length |           1            |
| exception_info |    exception_table     | exception_table_length |
|       u2       |    attributes_count    |           1            |
| attribute_info |       attributes       |    attributes_count    |

源代码：

```java
public class CodeAttribute {

    public int divide(int a, int b) {
        int c = 0;
        try {
            c = a / b;
        } catch (ArithmeticException ex) {
            ex.printStackTrace();
        } catch (Exception ex) {
            ex.printStackTrace();
        } finally {
            c = Integer.MIN_VALUE;
        }
        return c;
    }
}
```

javap -verbose

```java
public class com.asm.test.CodeAttribute
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #9.#33         // java/lang/Object."<init>":()V
   #2 = Class              #34            // java/lang/Integer
   #3 = Integer            -2147483648
   #4 = Class              #35            // java/lang/ArithmeticException
   #5 = Methodref          #4.#36         // java/lang/ArithmeticException.printStackTrace:()V
   #6 = Class              #37            // java/lang/Exception
   #7 = Methodref          #6.#36         // java/lang/Exception.printStackTrace:()V
   #8 = Class              #38            // com/asm/test/CodeAttribute
   #9 = Class              #39            // java/lang/Object
  #10 = Utf8               <init>
  #11 = Utf8               ()V
  #12 = Utf8               Code
  #13 = Utf8               LineNumberTable
  #14 = Utf8               LocalVariableTable
  #15 = Utf8               this
  #16 = Utf8               Lcom/asm/test/CodeAttribute;
  #17 = Utf8               divide
  #18 = Utf8               (II)I
  #19 = Utf8               ex
  #20 = Utf8               Ljava/lang/ArithmeticException;
  #21 = Utf8               Ljava/lang/Exception;
  #22 = Utf8               a
  #23 = Utf8               I
  #24 = Utf8               b
  #25 = Utf8               c
  #26 = Utf8               StackMapTable
  #27 = Class              #38            // com/asm/test/CodeAttribute
  #28 = Class              #35            // java/lang/ArithmeticException
  #29 = Class              #37            // java/lang/Exception
  #30 = Class              #40            // java/lang/Throwable
  #31 = Utf8               SourceFile
  #32 = Utf8               CodeAttribute.java
  #33 = NameAndType        #10:#11        // "<init>":()V
  #34 = Utf8               java/lang/Integer
  #35 = Utf8               java/lang/ArithmeticException
  #36 = NameAndType        #41:#11        // printStackTrace:()V
  #37 = Utf8               java/lang/Exception
  #38 = Utf8               com/asm/test/CodeAttribute
  #39 = Utf8               java/lang/Object
  #40 = Utf8               java/lang/Throwable
  #41 = Utf8               printStackTrace
{
  public com.asm.test.CodeAttribute();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/asm/test/CodeAttribute;

  public int divide(int, int);
    descriptor: (II)I
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=6, args_size=3
         0: iconst_0
         1: istore_3
         2: iload_1
         3: iload_2
         4: idiv
         5: istore_3
         6: ldc           #3                  // int -2147483648
         8: istore_3
         9: goto          46
        12: astore        4
        14: aload         4
        16: invokevirtual #5                  // Method java/lang/ArithmeticException.printStackTrace:()V
        19: ldc           #3                  // int -2147483648
        21: istore_3
        22: goto          46
        25: astore        4
        27: aload         4
        29: invokevirtual #7                  // Method java/lang/Exception.printStackTrace:()V
        32: ldc           #3                  // int -2147483648
        34: istore_3
        35: goto          46
        38: astore        5
        40: ldc           #3                  // int -2147483648
        42: istore_3
        43: aload         5
        45: athrow
        46: iload_3
        47: ireturn
      Exception table:
         from    to  target type
             2     6    12   Class java/lang/ArithmeticException
             2     6    25   Class java/lang/Exception
             2     6    38   any
            12    19    38   any
            25    32    38   any
            38    40    38   any
      LineNumberTable:
        line 6: 0
        line 8: 2
        line 14: 6
        line 15: 9
        line 9: 12
        line 10: 14
        line 14: 19
        line 15: 22
        line 11: 25
        line 12: 27
        line 14: 32
        line 15: 35
        line 14: 38
        line 15: 43
        line 16: 46
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
           14       5     4    ex   Ljava/lang/ArithmeticException;
           27       5     4    ex   Ljava/lang/Exception;
            0      48     0  this   Lcom/asm/test/CodeAttribute;
            0      48     1     a   I
            0      48     2     b   I
            2      46     3     c   I
      StackMapTable: number_of_entries = 4
        frame_type = 255 /* full_frame */
          offset_delta = 12
          locals = [ class com/asm/test/CodeAttribute, int, int, int ]
          stack = [ class java/lang/ArithmeticException ]
        frame_type = 76 /* same_locals_1_stack_item */
          stack = [ class java/lang/Exception ]
        frame_type = 76 /* same_locals_1_stack_item */
          stack = [ class java/lang/Throwable ]
        frame_type = 7 /* same */
}
SourceFile: "CodeAttribute.java"

```

字节码文件：

