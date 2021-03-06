# Parser

## 抽象语法树

语法分析器工作的结果便是抽象语法树(AST:abstract syntax tree)。分析器读完Token流之后，在语言的语法规则指导下，对源程序有了自己的理解，将这种理解转化成方便后期使用的一种中间表示，这个中间表示是层次分明的树形结构，叫做抽象语法树。

本质来讲，抽象语法树可以看作是对该语言文法的“类型化”重写，即对于每个非终结符，每个产生式，给出确切定义，并赋予不同的属性。类型实例便成为要输出的抽象语法树的“节点”

### 树形结构表示

依据之前给出的程序文法，可以给出一个树形表示的大致结构

// TODO:
[插入树形SmartArt即可]
[Method节点可能要单独拿出来]

+ Program
  + MainClass
    + main
      + Statement
  + ClassList
    + Class
      + VarList
        + Var
        + ...
      + MethodList
        + Method
        + ...
    + ...

+ Method
  + Name
  + Return Type
  + FormalList
    + Formal
    + ...
  + VarList
    + Var
    + ...
  + StatementList
    + Statement
    + ...
  + Return Exp

+ Var
  + Type
  + Id

上文所述的每一个节点(不包含XXXList类型节点)，我们都有给出确切的定义，并且在节点内携带足够的信息(行号，类型信息等)，方便后期的语义分析和错误提示信息。

### 详细介绍及继承关系

我们的AST中共包含8种主要的类型如下

+ Program

  程序实体类，语法树的根节点，拥有 MainClass 和 ClassList 两个属性
  + MainClass   // 主类(程序入口)
  + ClassList   // 用户自定义的类

+ MainClass

  主类实体类，程序入口方法所在类，拥有 Name Statement(从属main方法) 两个属性
  + Name        // 类名
  + Statement   // main方法中的语句

+ Class

  类实体类，表示用户自定义的类，拥有 Name BaseClass FieldList MethodList 四个属性
  + Name        // 类名
  + BaseClass   // 父类(可空)
  + FieldList   // 字段列表
  + MethodList  // 实例方法列表

+ Method

  方法实体类，表示用户定义的实例方法
  + Name            // 方法名
  + ReturnType      // 返回类型
  + ParameterList   // 方法的参数
  + LocalVarList    // 方法内声明的本地变量
  + StatementList   // 语句
  + ReturnExp       // 返回表达式

+ Statement

  语句抽象类，赋值语句、输出语句、if-else语句、 while语句、语句块直接继承自此类

  // TODO: 插入树形关系图

+ Expression

  表达式抽象类，加减乘运算、小于运算、与运算、非运算、true/false、方法调用、this表达式、对象创建表达式(new)、整数常量、变量访问直接继承此抽象类
  + LineNumber  //表达式所带行号

  // TODO: 插入树形关系图

+ Variable

  变量/字段实体类
  + Type    // 该变量/字段的类型
  + Id      // 变量/字段名

+ Type

  类型抽象类，整型、布尔型、类类型直接继承自此抽象类

  // TODO: 插入树形关系图

## 语法分析实现

语法分析器的任务是读入记号流，在语言的语法规则指导下生成抽象语法树。

分析算法主要分为自顶向下分析和自底向上分析，其中自顶向下分析包括递归下降分析算法(预测分析算法)和LL分析算法，自底向上分析包括LR分析算法。

### 递归下降分析算法

本编译器采用的是递归下降分析算法(预测分析)，该算法的主要优点是

+ 分析高效，线性时间复杂度
+ 容易实现，方便手工编码
+ 错误定位和诊断信息准确

很多开源编译器、商业编译器也采用了该算法，比如GCC4.0，LLVM等

该算法的基本思想是：

+ 为每个非终结符构造一个分析函数
+ 通过**前看符号**指导产生式规则的选择

### 分析算法在本程序的应用

我们选择一个简单的部分来介绍递归下降分析算法在程序中的应用。

如上文所述，用户自定义的类由两部分构成，字段列表和实例方法列表(当然，这个两个列表都可以为空)，那么我们就给出这样一个方法

```java
// Class
//   -> class Id { VarDecList MethodDecList }
//   | class Id : Id { VarDecList MethodDecList }
Class ParseClass()
{
    // 其他代码
    _fieldList = ParseFieldList();
    _methodList = ParseMethodList();

    return new Class(_fieldList, _methodList);
}
```

如上述代码所示，在从记号流解析一个类型实体时，调用了字段列表和方法列表的解析方法，并将解析的结果作为一个类型实体的组成部分(此处并未体现类名、父类等信息)。

很显然的，在 `ParseFieldList`、`ParseMethodList`两个方法内部，必定含有对于单个字段、单个方法实体的解析方法的调用，并将单个的实体结果组织起来，以返回给外界。

思考一下我们的文法规定的一个方法的构成形式，返回类型、方法名、参数列表、本地变量列表、语句列表，返回语句。这些部分的一个组织是方法，那么很自然地，我们又能为此写一个解析方法。

以上就体现了分析算法中*为每个非终结符构造一个分析函数*思想。但是还有另外一部分思想，*通过**前看符号**指导产生式规则的选择*还未体现。

考虑另外一条文法，关于“语句”。语句在我们的程序中有五种形式，由{}组织的语句块，if-else语句，while语句，赋值语句，输出语句。我们只需要看第一个符号，就能知道应当选择哪一条产生式来解析，伪代码描述如下所示。

```java
// Statement
//    -> { StatementList }
//    | if (Exp) Statement else Statement
//    | while (Exp) Statement
//    | print(Exp);
//    | Id = Exp;
Statement ParseStatement()
{
    switch(firstToken)
    {
        case "{":
            return ParseStatementBlock();
        case "if":
            return ParseIfElseStatement();
        case "while":
            return ParseWhileStatement();
        // ...
        default:
            throw new WrongCodeException(message);
    }
}
```

从伪代码中很清晰地体现出来，我们只需要通过查看一个“前看符号”就能确定要选择哪个方向去解析当前语句。如果解析失败，那么必定是用户给的源程序出现了问题，导致我们程序选择了错误的方向，或者出现了错误，这就需要用户修改源代码，然后重新编译。

## 实例分析

对于文法附带给出的程序样例中的一行语句

```java
total = num * (this.Compute(num-1));
```

语法分析完成后的输出的抽象语法树如下示意

// TODO: 插入树形图

### 错误处理

在语法分析器里面依旧没有任何错误恢复手段，但是已经给出了足够的错误提示信息，用于告知用户程序处理到的位置和出现的错误。

其实在语法分析部分，可能出现的错误的仅有一类，预期是某个符号，但是得到的却是另外的符号，这样就会出现错误了。

举例如下：

```java
int DoSomething(int )
{
  // ...
}
```

很容易看出，在这个方法的参数列表部分，缺失了参数名，因此将会给出错误提示：

```text
Line 6: Expects Id, but got Rparen.
Syntax error line 6, compilation aborting...
```

给出错误提示之后就退出编译过程，等待用户修改源代码并重新编译。