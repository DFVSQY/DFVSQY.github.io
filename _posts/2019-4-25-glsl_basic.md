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
---------|----------
 uint    | int
 float   | int,uint
 double  | int,uint,float

同样的规则也适用于由此组合而成的向量，矩阵类型，但转换规则不适用于数组和结构。对于其他所有的类型转换需显式使用转换构造函数，如下：
~~~ glsl
float f = 10.0
int ten = int(f);
double d = 5.0LF;
int five = int(d);
~~~

与C语言不同的是，GLSL中的函数支持函数重载。

GLSL中对于由基本类型组合而成的向量，矩阵类型的支持如下：

Base Type |  2D Vec  |  3D Vec  |  4D Vec  | Matrix Type
----------|----------|----------|----------|-------------
 float    | vec2     | vec3     | vec4     |mat2 mat3 mat4<br>mat2x2 mat2x3 mat2x4<br>mat3x2 mat3x3 mat3x4<br>mat4x2 mat4x3 mat4x4
 double   | dvec2    | dvec3    | dvec4    |dmat2 dmat3 dmat4<br>dmat2x2 dmat2x3 dmat2x4<br>dmat3x2 dmat3x3 dmat3x4<br>dmat4x2 dmat4x3 dmat4x4
 int      | ivec2    | ivec3    | ivec4    |    -
 uint     | uvec2    | uvec3    | uvec4    |    -
 bool     | bvec2    | bvec3    | bvec4    |    -

对于矩阵类型，第一个维度数表示矩阵列数，第二个维度数表示矩阵行数。

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

vec2 column4 = vec2(1.0, 4.0);
vec2 column5 = vec2(2.0, 5.0);
vec2 column6 = vec2(3.0, 6.0);

// equal to M3
mat3 M5 = mat3(column4, 7.0, column5, 8.0, column6, 9.0);
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
--------------------|----------
 (x,y,z,w)          | Components associated with positions
 (r,g,b,a)          | Components associated with colors
 (s,t,p,q)          | Components associated with texture coordinates

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


未完待续！

<!-- 
##### Storage Qualifiers
##### 语句
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
