---
layout: post
title: OpenGL Shader基础
catalog: true
tags:
    - OpenGL
    - GLSL
    - Shader
author: 东方VS清扬
---

本篇文章基于OpenGL4.5进行讲解

### 目标
* 能区分OpenGL所使用的不同类型的Shader
* 利用GLSL编写并编译Shader
* 利用OpenGL中多种机制向Shader传递数据
* 利用GLSL高级能力实现Shader的复用

### OpenGL可编程渲染管线
OpenGL4.5版本渲染管线提供了四个可编程阶段另加一个计算阶段(compute stage)，每一个阶段都可使用Shader进行控制，下面分别介绍这五个可编程阶段：

* 顶点着色阶段(Vertex Shading Stage)接收在顶点缓存对象(vertex-buffer objects)中指定的数据并独立处理每一个顶点。这是OpenGL应用程序中的强制性阶段，每一个OpenGL程序都要在该阶段有一个对应的顶点着色器。
* 曲面细分着色阶段(Tessellation Shading Stage)是一个可选的阶段，该阶段会在OpenGL管线中产生额外的几何，如果该阶段被启用，它会接受来自于顶点着色阶段的输出并对接受的数据做进一步的处理。曲面细分阶段实际上被分为两个阶段：细分控制着色器(Tessellation Control Shader)和曲面计算着色器(Tessellation Evaluation Shader)。
* 几何着色阶段(Geometry Shading Stage)是另一个可选的阶段，该阶段会在OpenGL管线中作用于独立的几何图元(geometric primitives)。可以从输入的图元中产生更多的几何，修改几何图元类型(e.g,converting triangles to lines)或者舍弃整个几何图元。它接收来自于顶点着色阶段或者曲面细分着色阶段输出的数据。
* 片段着色阶段(Fragment Shading Stage)是OpenGL管线中最后一个阶段，该阶段用于处理OpenGL光栅化器(Rasterizer)产生的每一个独立片段(Fragment)或者采样(Sample，当Sample-Shading模式被启用)。在该阶段片段的颜色或者深度值被计算并被发送至OpenGL管线中的片段测试(Fragment-Testing)和片段混合(Fragment-Blending)部分，该阶段必须要有一个对应的片段着色器。
* 计算着色阶段(Computing Shading Stage)并不是图形管线的一部分，它自己代表了一个单独的阶段，处理由应用程序指定的工作。计算着色器可以处理由其他着色器创建和使用的缓存，例如：帧缓存后处理效果(FrameBuffer Post-Processing Effect)等等。

### GLSL简介
GLSL(OpenGL Shading Language)是OpenGL中创建Shader的语言，支持OpenGL中所有阶段的Shader的编写，首先介绍一下GLSL的要求，类型和所有阶段共用的一些语言组成部分，然后介绍GLSL用于每个阶段的独有特性。
##### 使用GLSL创建Shader
同C语言程序一样，每一个Shader程序都有一个`main()`入口，如下：
~~~ glsl
#version 330 core
void main()
{
    // your code goes here
}
~~~
第一行指定了GLSL语言使用的版本，`330`表示使用OpenGL3.3对应的GLSL版本，GLSL版本基于OpenGL版本的命名方式从OpenGL3.3开始，在此之前GLSL使用不同版本号规则。`Core`则指明程序使用OpenGL's core profile。每一个Shader在起始处都要有一个`#version`行，否则默认应用GLSL版本`#version 110`。

和C语言一样，`//`表示单行注释，`/**/`可用于多行注释，每一条语句用`;`结束，与ANSI C语言不同的是`main()`函数并不能返回一个整数值也不接受任何参数。这段简单的代码在OpenGL完全合法(无论在编译还是运行阶段)，尽管无实质性功能。

Shader中的一个重要基础是理解OpenGL管线中数据流是如何在各个阶段的流入流出的，每一个阶段都类似于函数的调用：数据传入，数据处理，数据传出。在C语言中这可以通过全局变量或者函数参数来实现，GLSL则不同，每一个Shader都像一个完整的C语言程序，入口为`main()`函数，由于`main()`不接受任何参数，所以任何传入或传出Shader的数据都是用特定的全局变量。如下：
~~~ glsl
#version 450 core
in vec4 vPosition;
int vec4 vColor;
out vec4 color;
uniform mat4 ModelViewProjectionMatrix;
void main()
{
    color = vColor;
    gl_Position = ModelViewProjectionMatrix * vPosition;
}
~~~
上述程序代码中使用的每一个输入输出数据都有一个对应的全局变量，`gl_Position`为内置支持变量无需声明，每一个变量在声明处都有一个对应的类型同时`in`关键字表示流入Shader中的数据，`out`关键字表示流出Shader的数据，这些变量的值在每次执行该Shader时更新(对于顶点着色器处理每一个顶点时更新，对于片段着色器处理每一个片段时更新)，其他从OpenGL应用程序接收数据的变量类型为`uniform`，该变量值的更新由应用程序决定。


GLSL中的每一个变量必须声明后才能使用，每一个变量的声明都有一个对应的类型，GLSL中的类型分为`Transparent Types`和`Opaque Types`.

`Transparent Types`包括：`float`(32位单精度浮点数),`double`(32位双精度浮点数),`int`(32位有符号整数),`uint`(32位无符号整数),`bool`(布尔值)以及上述基本类型组合而成的组合类型。

`Opaque Types`包括：`sampler types`,`image types`和`atomic counter types`,这些类型分别用于访问'texture maps','images'和'atomic counters'。

同C语言一样，GLSL的变量必须在使用之前进行声明，同时变量的作用域与C语言规则相同：
* 任何函数定义外声明的变量具有全局性，对同一个Shader程序中的后续函数可见
* 花括号内声明的变量仅在换号内起作用
* 循环迭代变量仅在循环体内起作用

变量在声明的同时也可以进行初始化，例如：
~~~ glsl
int i, j = 20, oct = 0247, hex1 = 0x32Abc, hex2 = 0X32abc;
float f, p = -9.8, s = 3E-7, x = 3.2f, y = 3.2F;
bool b = false;
uint m = 34u, n = 76U;
double d = 3.1415LF, a = 32.2lF;
~~~
整数类型的字面常量可以用十进制，八进制或者十六进制表示，后接`u`或`U`表示`uint`类型。

浮点性数字可以用十进制小数类型或者科学计数法表示，对于`float`类型后接可选的`f`或`F`,对于`double`类型必须后接`lF`或`LF`。

布尔变量的可选值为`true`或`false`。

GLSL相比于C语言有着更高的类型安全，因为GLSL允许更少的类型隐式转换，例如：
~~~ glsl
int i = false
~~~
上述代码会导致GLSL的编译错误，GLSL中所允许的隐式类型转换如下：

Type Needed | Can Be Implicitly Converted From
------------|---------------------------------
uint        | int
float       | int,uint
double      | int,uint,float

同样的规则也适用于由此组合而成的向量，矩阵类型，但转换规则不适用于数组和结构。对于其他所有的类型转换需显式使用转换构造函数，如下：
~~~ glsl
float f = 10.0
int ten = int(f);
double d = 5.0LF;
int five = int(d);
~~~

与C语言不同的是，GLSL中的函数支持函数重载。

GLSL中对于由基本类型组合而成的向量，矩阵类型的支持如下：

Base Type | 2D Vec | 3D Vec | 4D Vec | Matrix Type
----------|--------|--------|--------|---------------------------------------------------------------------------------------------------
float     | vec2   | vec3   | vec4   | mat2 mat3 mat4<br>mat2x2 mat2x3 mat2x4<br>mat3x2 mat3x3 mat3x4<br>mat4x2 mat4x3 mat4x4
double    | dvec2  | dvec3  | dvec4  | dmat2 dmat3 dmat4<br>dmat2x2 dmat2x3 dmat2x4<br>dmat3x2 dmat3x3 dmat3x4<br>dmat4x2 dmat4x3 dmat4x4
int       | ivec2  | ivec3  | ivec4  | -
uint      | uvec2  | uvec3  | uvec4  | -
bool      | bvec2  | bvec3  | bvec4  | -

对于矩阵类型，第一个维度数表示对应数学矩阵的列数，第二个维度数表示对应数学矩阵的行数。

向量的初始化和类型转换与其对应的标量类似，同时向量的构造函数支持维度的收缩和扩展，如下：
~~~ glsl
// vector constructor
vec3 velocity = vec3(2.0,2.0,2.0);
dvec3 dvelocity = vec(2.0LE,2.0LE,2.0LE);

// vector constructor, equal to vec3(2.0,2.0,2.0)
vec3 velociy = vec3(2.0)

// explicitly type convert
ivec3 = ivec3(velocity);

// truncate vector
vec4 color;
vec3 rgb = vec3(color);     // rgb only has three elements

// lengthen vector
vec3 white = vec3(1.0);
vec4 translucent = vec4(white, 0.5);
~~~

矩阵的初始化与其对应的标量类似，如下：
~~~ glsl
mat3 M1 = mat3(4.0, 0, 0,
              0, 4.0, 0,
              0, 0, 4.0);

// equal to M1
mat3 M2 = mat3(4.0)

mat3 M3 = mat3(1.0, 2.0, 3.0,
               4.0, 5.0, 6.0,
               7.0, 8.0, 9.0);

vec3 column1 = vec3(1.0, 2.0, 3.0);
vec3 column2 = vec3(4.0, 5.0, 6.0);
vec3 column3 = vec3(7.0, 8.0, 9.0);

// equal to M3
mat3 M4 = mat3(column1, column2, column3);

vec2 column4 = vec2(1.0, 2.0);
vec2 column5 = vec2(4.0, 5.0);
vec2 column6 = vec2(7.0, 8.0);

// equal to M3
mat3 M5 = mat3(column4, 3.0, column5, 6.0, column6, 9.0);
~~~
`M3`,`M4`和`M5`均表示以下数学矩阵：

$$
 \left[
 \begin{matrix}
   1.0 & 4.0 & 7.0 \\
   2.0 & 5.0 & 8.0 \\
   3.0 & 6.0 & 9.0
  \end{matrix}
  \right]
$$

向量和矩阵的单个元素可被读写，向量的单元素有两种访问方式：

* 基于named-component的方式
* 基于array-like的方式

~~~ glsl
// named-component 
float red = color.r;
float velocity_y = velocity.y;
float tex_p = tex.p;

// array-like
float red = color[0];
float velocity_y = velocity[1];
float tex_p = tex[2];
~~~

GLSL支持向量的三种名称组件集合，如下：

Component Accessors | Description
--------------------|-----------------------------------------------
(x,y,z,w)           | Components associated with positions
(r,g,b,a)           | Components associated with colors
(s,t,p,q)           | Components associated with texture coordinates

向量元素访问示例：
~~~ glsl
// repeatly access the same element
vec3 luminance = color.rrr;

// reverse the components of a color
color = color.abgr; 

// Error, 'z' is from a different group
vec4 color = otherColor.rgz;

// Error, no 'z' component in 2D vectors
vec2 pos;
float zPos = pos.z;
~~~

矩阵支持二维array-like的访问方式，例如：
~~~ glsl
mat3 M = mat3(1.0, 2.0, 3.0,
              4.0, 5.0, 6.0,
              7.0, 8.0, 9.0);

// (4.0, 5.0, 6.0)
vec3 yVec = M[1];

// 7.0
float yScale = M[2][0]
~~~

GLSL同样支持结构体，如下：
~~~ glsl
// define a struct
struct Particle
{
    float lifetime;
    vec3 position;
    vec3 velocity;
}

// construct a struct variable
Particle p = Particle(10.0, pos, vel);

// access an element
float f = p.lifetime;
~~~

GLSL也支持任何类型的数组，同C语言一样使用`[]`进行索引，起始索引从0开始，但与C语言不同的是GLSL不支持负数索引与正数越界索引。从OpenGL4.3开始支持多维数组，即数组元素也可以是数组。

数组的声明可以指定大小也可不指定，如下：
~~~ glsl
// declare with a size
float coeff[3];

// same to above
float[3] coeff;

// unsized, redeclare later with a size
int indices[];
~~~

数组的初始化如下：
~~~ glsl
float coeff[3] = float[3]{2.38, 1.24, 7.65};

// same to above
float coeff[3] = float[]{2.38, 1.24, 7.65};
~~~

每一个数组对象都包含一个`length()`函数用于返回当前数组元素个数，如下：
~~~ glsl
for(int i = 0; i < coeff.length(); ++i)
{
    coeff[i] *= 2.0;
}
~~~

向量和矩阵同数组一样也支持`length()`函数，向量`length()`返回组件的个数，矩阵`lenght()`返回所表示的数学矩阵所包含的列数，如下：
~~~ glsl
mat3x4 m;

// number of columns in m: 3
int c = m.length();

// number of components in column vector 0: 4
int r = m[0].length();
~~~

当数组，向量或者矩阵的长度在编译期可以确定时，`length()`会返回一个编译器常量，该常量可用于任何需要编译期常量的地方，如下：
~~~ glsl
mat4 m;

// array of size matching the matrix size
float diagonal[m.length()];

// array of size matching the number of geometry shader input vertices
float x[gl_in.length()];
~~~

对于绝大部分的数组，向量和矩阵来说，其长度都是在编译期间可以确定的，不过对于部分数组其长度需要在链接期间(link-time)才可以确定，例如在同一阶段来自多个Shader的情况下需要链接器(linker)推断数组尺寸。对于shader storage buffer objects，其长度需要到渲染时(render-time)才能确定。

GLSL多维数组的使用类似于C语言，如下：
~~~ glsl
// an array of size 3 of array of size 5
float coeff[3][5];

// inner-dimension index is 1, outer is 2
coeff[2][1] *= 2.0;

// return 3
coeff.length();

// an one-dimensional array of size 5
coeff[2];

// return 5
coeff[2].length();
~~~

##### 存储修饰符
GLSL变量在声明时可以指定存储修饰符，修饰符会影响到变量的相关行为，GLSL中支持的修饰符如下，当声明全局变量时相关行为会起作用：

Type Modifier | Description
--------------|----------------------------------------------------------------------------------------------------------------------
const         | Labels a variable as read-only. It will also be a compile-time constant if its initializer is a compile-time constant
in            | Specifies that the variable is an input to the shader stage
out           | Specifies that the variable is an output from a shader stage
uniform       | Specifies that the value is passed to the shader from the application and is constant across a given primitive
buffer        | Specifies read-write memory shared with the application. This memory is also referred to as a _shader storage buffer_
shared        | Specifies that the variables are shared within a local work group. This is used only in compute shaders

同C语言一样，`const`用于表示变量为只读类型，必须在声明时为该类型变量赋值。`in`用于表示流入shader的数据，每次执行shader时会根据输入数据自动更新对应变量。`out`用于表示流出shader的数据，比如顶点着色器输出的顶点的齐次坐标或片段着色器输出的颜色。`uniform`用于表示变量在shader执行前会被应用程序赋值，而且不会随着处理的vertex或者fragment不同而改变，该变量值直到下次应用程序指定新值时才会改变。`uniform`变量在所有被启用的shader阶段是共享的且必须被声明为全局变量。任何类型的变量(包括结构体和数组)都可以被声明为`uniform`类型，其值在shader中不可被改变只能由应用程序传入。

当OpenGL链接shader程序时GLSL的编译器会为`uniform`变量创建一个 _table_，在应用程序中设置`uniform`变量的值时首先需要使用`glGetUniformLocation(GLuint program, const char* name)`函数获得该变量在 _table_ 中的索引，然后使用`glUniform*()`或者`glUniformMatrix*()`函数设置具体值，如下：
~~~ c
GLint timeLoc = glGetUniformLocation(program, "time");
glUniform1f(timeLoc, 5.0f);
~~~

`buffer`类型是与应用程序共享大缓存时推荐的一种方式，其与`uniform`类型相似，只不过该类型变量可以在shader中被修改。

`shared`类型的变量仅用于compute shader，该类型用于创建与 _local work group_ 共享的内存。

##### 语句
GLSL支持的操作符和优先级如下表：

Precedence | Operators                                           | Accepted Types                                                                       | Description
-----------|-----------------------------------------------------|--------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
1          | ()                                                  | -                                                                                    | Grouping of operations
2          | []                                                  | arrays<br>matrices<br>vecotrs                                                        | Array subscripting
3          | f()<br>.(period)<br>++ --<br>++ --<br>+ -<br>~<br>! | functions<br>structures<br>arithmetic<br>arithmetic<br>arithmetic<br>integer<br>bool | Function calls and constructors<br>Structure field or method access<br>Post-increment and -decrement<br>Pre-increment and -decrement<br>Unary explicit positive or negation<br>Unary bit-wise not<br>Unary logical not
4          | * / %                                               | arithmetic                                                                           | Multiplicative operations
5          | + -                                                 | arithmetic                                                                           | Additive operations
6          | << >>                                               | integer                                                                              | Bit-wise operations
7          | < > <= >=                                           | arithmetic                                                                           | Relational operations
8          | == !=                                               | any                                                                                  | Equality operations
9          | &                                                   | integer                                                                              | Bit-wise and
10         | ^                                                   | integer                                                                              | Bit-wise exclusive or
11         | \|                                                  | integer                                                                              | Bit-wise inclusive or
12         | &&                                                  | bool                                                                                 | Logical and operation
13         | ^^                                                  | bool                                                                                 | Logical exclusive-or operation
14         | \|\|                                                | bool                                                                                 | Logical or operation
15         | a ? b : c                                           | bool ? any : any                                                                     | Ternary selection operation
16         | =<br>+= -=<br>*= /=<br>%= <<= >>=<br>&= ^= \|=      | any<br>arithmetic<br>arithmetic<br>integer<br>integer                                | Assignment<br>Arithmetic assignment<br><br><br><br>
17         | ,                                                   | any                                                                                  | Sequence of operations

_integer_：`int`, `uint` and `vectors` of them<br>
_floating-point_：`float`, `double` and `vectors` of them and `matrices` of them<br>
_arithmetic_：all _integer_ and _floating-point_ types<br>
_any_：additionally includes `structures` and `arrays`

在GLSL中大多数操作符都支持重载，即它们可操作受支持类型的不同集合。特别是在GLSL中算数操作符对于向量和矩阵都有相关的定义。例如：
~~~ glsl
// 向量的四则运算等同于对应的各元素的四则运算
// c = vec3(1.1, 2.2, 3.3)
// d = vec3(0.1, 0.4, 0.9)
vec3 a = vec3(1.0, 2.0, 3.0);
vec3 b = vec3(0.1, 0.2, 0.3);
vec3 c = a + b; 
vec3 d = a * b; 

// 向量作为右操作数与矩阵相乘时被视为列向量
// w = vec2(1. * 10. + 3. * 20., 2. * 10. + 4. * 20.)
vec2 v = vec2(10., 20.);
mat2 m = mat2(1., 2.,  3., 4.);
vec2 w = m * v; 

// 向量作为左操作数与矩阵相乘时被视为行向量
// w = vec2(1. * 10. + 2. * 20., 3. * 10. + 4. * 20.)
vec2 v = vec2(10., 20.);
mat2 m = mat2(1., 2.,  3., 4.);
vec2 w = v * m; 

// 矩阵和矩阵相乘遵从矩阵乘法的数学原则
// c = mat2(1. * 10. + 3. * 20., 2. * 10. + 4. * 20., 
//          1. * 30. + 3. * 40., 2. * 30. + 4. * 40.)
mat2 b = mat2(10., 20.,  30., 40.);
mat2 a = mat2(1., 2.,  3., 4.);
mat2 c = a * b; 

// 向量和标量相乘等同于对应的各个元素和标量相乘
// b = vec3(10.0, 20.0, 30.0)
// c = vec3(10.0, 20.0, 30.0)
float s = 10.0;
vec3 a = vec3(1.0, 2.0, 3.0);
vec3 b = s * a; 
vec3 c = a * s; 

// 矩阵和标量相乘等同于对应的各个元素和标量相乘
float s = 10.0;
mat3 m = mat3(1.0);
mat3 m2 = s * m; // = mat3(10.0)
mat3 m3 = m * s; // = mat3(10.0)
~~~

GLSL支持`if-else`和`switch`语句，如下：
~~~ glsl
if(flag)
{
    // true clause
}
else
{
    // false clause
}

switch(int_value)
{
    case n:
        // statements
        break;
    case m:
        // statements
        break;
    case p:
    case q:
        // statements
        break;
    default:
        // statements
        break;
}
~~~

GLSL支持的循环语句如下：
~~~ glsl
for(int i = 0; i < 10; ++i)
{
    // statements
}

while(n < 10)
{
    // statements
}

do
{
    // statements
} while(n < 10)
~~~

GLSL控制流语句除了常见的`break`，`continue`，`return`之外，还支持`discard`语句：

Statement       | Description
----------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
break           | Terminates execution of the block of a loop and continues execution after the scope of that block.
continue        | Terminates the current iteration of the enclosing block of a loop, resuming execution with the next iteration of the loop.
return [result] | Returns from the current function, optionally providing a value to be returned from the function (assuming return value matches the return type of the enclosing function).
discard         | Discards the current fragment and ceases shader execution. Discard statements are valid only in fragment shader programs.<br>The execution of the fragment shader may be terminated at the execution of the discard statement, but this is implementation-dependent.

GLSL中也支持自定义函数，在一个shader对象（shader object）中定义的函数可以在多个Shader程序（shader program）中重复使用。

GLSL的函数声明和C语言类似，只不过在参数中存在可选的访问修饰符，如下：
~~~ 
// 其中GLSL函数名称不能以数字，gl_或者双下划线 `__`开头
// 数组用于返回类型或者参数类型时必须显式地指定数组的大小
returnType functionName([accessModifier] type1 arg0,
                        [accessModifier] type2 arg1,
                        ...)
{
    return returnValue;         // unless returnType is 'void'
}
~~~
GLSL的函数在调用之前必须有对应的函数原型声明或者函数定义，这一点同C语言相同，否则GLSL编译器会报错。如果函数在不同于调用处的shader对象（shader object）中定义，则在调用处的shader对象中使用该函数时必须进行原型声明。

GLSL不像C和C++一样拥有指针或者引用的概念，因此函数传参可以使用访问修饰符指定该参数是否需要复制传入或者传出函数，GLSL中支持的访问修饰符如下：

Access Modifier | Description
----------------|------------------------------------------------------------------------
in              | Value copied into a function(default if not specified)
const in        | Read-only value copied into a function
out             | Value copied out of function(undefined upon entrance into the function)
inout           | Value copied into and out of a function


##### 计算的不变性
GLSL并不保证不同shader中的同一计算会产生相同值（因为不同的shader是单独编译的，在编译时glsl编译器会对shader进行优化，不同shader使用的优化方式可能不同），这个轻微的不同可能对于多道渲染算法（multipass algorithms）有明显影响，在这种情况下GLSL提供了两种方法确保不同shader的不变量，即`invariant`和`precise`。这两种方法都会保证由图形设备完成的同一计算得到相同结果，但是无法保证不同主机或者图形设备得到相同结果。另外编译期的常量表达式由编译器计算得到，该结果同样无法保证和图形设备计算得到的结果相同，例如：
~~~ glsl
uniform float ten;              // application sets this to 10.0
const float f = sin(10.0);      // computed on compiler host
float g = sin(ten);             // computed on graphics device

void main()
{
    if(f == g)                  // f and g might be not equal
    {
        // do something ...
    }
}
~~~

`invariant`限定符可用于shader输出的任何变量，这样可保证不同shader的同一计算公式得到相同的结果。被声明为`invariant`的输出变量可以是内置变量或者自定义变量，如下：
~~~ glsl
invariant gl_Position;
invariant centroid out vec3 Color;
~~~

为了调试方便可以强制所有可能会不同的变量使用`invariant`限定符，这可以通过下面的shader预处理器指令实现：
~~~ glsl
#pragma STDGL invariant (all)
~~~

全局的使用`invariant`可能会对性能造成影响，因为通常情况下为了保证不变性编译器会禁止可能的优化。

`precise`限定符可用于任何被计算的变量或者函数返回值，该限定符并不像其名称所显示的那样是为了增加精确度，而是增加计算的可重复性。其绝大部分用于曲面细分shader以避免几何体上破裂（forming cracks）情况的出现。

通常情况下如果想从同一表达式得到相同结果可以使用`precise`代替`invariant`，即便表达式所使用的值仅仅是数学意义上不影响最终结果的交换，如下`a`和`b`交换，`c`和`d`交换，甚至`a+b`和`c+d`交换：
~~~ glsl
Location = a * b + c * d;           ★
~~~

`precise`限定符可用于内置变量，用户自定义变量或者函数返回值，如下：
~~~ glsl
precise gl_Position;
precise out vec3 Location;
precise vec3 subdivide(vec3 p1, vec3 p2) {...}
~~~
`precise`可在变量被使用前的任何时刻被应用，也可以被应用于先前声明的变量。

`precise`对于编译器带来的一个实际影响是类似★的表达式不能同时使用两种不同的乘法命令参与计算，例如第一种用普通乘法，第二种用混合乘加（fused multiply-and-add, fma）,因为这两种命令对同一组数值结果可能会有微小差异，而这种差异不被`precise`允许，所以编译器会直接报错。因为fma对于性能提升很重要，不能完全禁止这种用法，所以OpenGL提供了一个内置函数`fma()`让用户替代原先的操作，如下：
~~~ glsl
precise out float result;
float f = c * d;
float result = fma(a,b,f)
~~~


##### Shader预处理
与C语言相似编译shader的第一步是进行预处理，并且GLSL提供了一系列命令提供条件编译和定义数值，与C语言不同的是，GLSL没有文件包含的预处理命令(#include)。

GLSL支持的预处理指令如下：

Preprocessor Directive                                     | Description
-----------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
\#define<br>\#undef                                        | Control the definition of constants and macros similar to the C preprocessor.
\#if<br>\#ifdef<br>\#ifndef<br>\#else<br>\#elif<br>\#endif | Conditional code management similar to the C preprocessor, including the `defined` operator<br>Conditional expressions evaluate integer expressions and defined values(as specified by #define)only
\#error text                                               | Cause the compiler to insert text(up to the first newline character)into the shader information log.
\#pragma options                                           | Control compiler specific options
\#extension options                                        | Specify compiler operation with respect to specified GLSL extensions
\#version number                                           | Mandate a specific version of GLSL version support.
\#line options                                             | Control diagnostic line numbering.


同C语言的预处理器一样，GLSL预处理器也支持宏定义，不过与C语言不同的是它不支持字符串替换和预编译连接符。

宏定义可以支持定义单一的值和带有参数的值，`#undef`则指定取消这些定义， 如下：
~~~ glsl
#define NUM_ELEMENTS 10

#define LPos(n) gl_LightSource[(n)].position

#undef LPos
~~~

GLSL也提供了一些预定义好的宏，可用于记录一些诊断信息（可以通过#error命令输出），如下：

Macro Name      | Description
----------------|--------------------------------------------------------------------------------------------------------------------
\_\_LINE\_\_    | Line number defined by one more than the number of newline characters processed and modified by the #line directive
\_\_FILE\_\_    | Source string number currently being processed
\_\_VERSION\_\_ | Integer representation of the GLSL version

GLSL预处理器也持支根据宏定义和整数常数的条件判断进入不同的代码段，宏定义可以通过两种方式参与条件表达式，使用`#ifdef`命令或者在`#if`和`#elif`命令中使用`defined`操作符。
~~~ glsl
#ifdef NUM_ELEMENTS
// some codes
#endif

#if defined(NUM_ELEMENTS) && NUM_ELEMENTS > 3
// some codes
#elif NUM_ELEMENTS < 7
// some codes
#endif
~~~

`#pragma`可以给编译器提供额外信息以控制shader如何被编译。例如`optimize`选项可以控制编译器是否启用优化，`debug`选项用于控制shader额外的诊断信息输出。例如：
~~~ glsl
#pragma optimize(on)
#pragma optimize(off)

#pragma debug(on)
#pragma debug(off)
~~~

GLSL也支持扩展，硬件厂商也可以添加自己定义的一些扩展。GLSL预处理器提供了`#extension`命令提示编译器如何处理这个扩展。对于任何一个扩展或者全部扩展都可以如下设置：
~~~ glsl
#extension extension_name : <directive>

#extension all : <directive>
~~~

\<directive\>可选的命令如下：

Directive | Description
----------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
require   | Flag and error if the extension is not supported or if the all-extension specification is used
enable    | Give a warning if the particular extensions specified are not supported, or flag and error if the all-extension specification is used
warn      | Give a warning if the particular extensions specified are not supported, or give a warning if andy extension use is detected during compilation
disable   | Disable support for the particular extensions listed(that is, have the compiler act as if the extension is not supported even if it is)or all extensions if `all` is present, issuing warnings and errors as if the extension were not present.






未完待续！


<!-- 
##### Computational Invariance
##### Shader Preprocessor
##### Compiler Control
##### Global Shader-Compilation Option


### 接口块
#### Uniform Blocks
#### Specifying Uniform Blocks in Shaders
#### Accessing Uniform Blocks from Application
#### Buffer Blocks
#### In/Out Blocks, Location, and Components


### 编译Shader


### Shader子程序
#### Advanced
#### GLSL Subroutine Setup
#### Selecting Shader Subroutines


### Separate Shader Objects
#### Advanced


### SPIR-V
#### Reasons to Choose SPIR-v
#### Using SPIR-V with OpenGL
#### Glslang
#### What's Inside SPIR-V? 
-->
