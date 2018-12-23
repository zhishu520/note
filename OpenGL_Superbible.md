
# OpenGLSuperbible 读书笔记

![图片](./images/superbible.jpg)  

[购买链接](https://www.amazon.cn/dp/0672337479/ref=sr_1_2?ie=UTF8&qid=1544971390&sr=8-2&keywords=opengl+superbible)

## 渲染管线概要图
![渲染管线](./images/graphics_pipeline.png)

顶点拾取 => 顶点着色器 => 
细分曲面着色器 => 细分曲面 =>  细分曲面评价着色器 => 
几何着色器 =>
光栅化 => 片段着色器 =>  帧缓冲区操作

## 第一个OpenGL程序

```cpp
#include <sb7.h>
class simpleclear_app : public sb7::application
{
    virtual void render(double currentTime)
    {
        static const GLfloat red[] = { 1.0f, 0.0f, 0.0f, 1.0f };
        glClearBufferfv(GL_COLOR, 0, red);
    }
};

DECLARE_MAIN(simpleclear_app)

```

### 使用着色器
OpenGL的着色器有 顶点着色器, 细分曲面着色器, 几何图形着色器 和 计算着色器
最简单的着色器只有一个顶点着色器，但是要想看到他的颜色还必须要一个片段着色器

```cpp
void glClearBufferfv(
    GLenum buffer,          // Specify the buffer to clear.
    GLint drawBuffer,       // Specify a particular draw buffer to clear.
    const GLfloat * value); // RGBA
```

vertex shader
```cpp
// 我们只使用 4.5 OpenGL core profile 的功能
# version 450 core
void main(void)
{
    // gl_Position vertex的位置
    gl_Position = vec4(0.0, 0.0, 0.5, 1.0);
}
```

fragment shader
```cpp
# version 450 core
out vec4 color; void main(void)
{
    color = vec4(0.0, 0.8, 1.0, 1.0);
}
```

### 编译着色器

```cpp

// shaderType:  GL_COMPUTE_SHADER, GL_VERTEX_SHADER, GL_TESS_CONTROL_SHADER, GL_TESS_EVALUATION_SHADER, GL_GEOMETRY_SHADER, or GL_FRAGMENT_SHADER.
// 创建一个shader对象
GLuint glCreateShader(GLenum shaderType);

// shader : glCreateShader 返回的句柄
// count : string 数组的size
// string : string数组
// length : string 的长度, 如果为NULL, 要求string以NULL结尾
// 取代shader对象里的代码
void glShaderSource(	GLuint shader,
                        GLsizei count,
                        const GLchar **string,
                        const GLint *length);

// shader : glCreateShader 返回的句柄
void glCompileShader(	GLuint shader);

// 创建空的program, shader可以attach
GLuint glCreateProgram(	void);

// program : glCreateProgram 返回的句柄
// shader : glCreateShader 返回的句柄
// 附加shader对象到program对象上
void glAttachShader(	GLuint program,
                        GLuint shader);

// 链接program对象
void glLinkProgram(	GLuint program);

// 删除program对象
void glDeleteProgram(	GLuint program);

// 删除shader对象
void glDeleteShader(	GLuint shader);


// 渲染阶段使用 program 对象
void glUseProgram(	GLuint program);

```

### Vertex Array Object (VAO) 顶点数组对象

```cpp

// n : 数组的大小
// arrays : 要绑定的句柄的指针
void glCreateVertexArrays(	GLsizei n,
                            GLuint *arrays);

// array : 绑定的句柄
void glBindVertexArray(	GLuint array);


// 和glCreateVertexArrays应该一致的
void glDeleteVertexArrays(	GLsizei n,
                            const GLuint *arrays);

```

### 绘画命令`glDrawArrays`

```cpp
// 给 OpenGL 管线输送顶点数据
// mode : 图元类型 GL_POINTS, GL_LINE_STRIP, GL_LINE_LOOP, GL_LINES, GL_LINE_STRIP_ADJACENCY, GL_LINES_ADJACENCY, GL_TRIANGLE_STRIP, GL_TRIANGLE_FAN, GL_TRIANGLES, GL_TRIANGLE_STRIP_ADJACENCY, GL_TRIANGLES_ADJACENCY and GL_PATCHES
// first : 开始的索引
// count : 渲染的个数
void glDrawArrays(	GLenum mode,
                    GLint first,
                    GLsizei count);


// 设置点的大小, 默认是1
// size : 光栅的直径
void glPointSize(	GLfloat size);
```

### 绘画三角形
```cpp
# version 450 core

void main()
{
    const vectices[3] = vec4[3](
        vec4(0.25, -0.25, 0.5, 1.0f),
        vec4(-0.25, -0.25, 0.5, 1.0f),
        vec4(0.25, 0.25, 0.5, 1.0f),
    );

    gl_Position = vertices[gl_VertexID];
}

glDrawArrays(GL_TRIANGLES, 0, 3);

```

## 跟着渲染管线走


顶点拾取 ->


### 传递数据给顶点着色器
顶点着色器是OpenGL渲染管线第一个可编程阶段，可编程阶段和其他强制阶段是有区别的
在顶点着色器运行之前，一个可修改函数阶段叫做 vertex fetching 有时候也叫 vertex pulling，他们自动提供了顶点着色器的输入

#### 顶点属性(vertex attribute)
在GLSL中，着色器输入输出数据是通过`in`和`out`限定符存储的

```
layout (location = 0) in vec4 offset
```
glVertexAttrib*() 更新 vertex attribute 输入到顶点染色器

```cpp
// index 对应location
// v 参数
void glVertexAttrib4fv(	GLuint index,
 	                    const GLfloat *v);
```
```cpp

GLfloat attrib[] = { 
    (float) sin(currentTime) * 0.5f,
    (float) cos(currentTime) * 0.6f,
    0.0f,
    0.0f };

glVertexAttrib4fv(0, attrb);
```

### shader之前传递数据
用out关键字创建输出变量，会被送到下一个阶段用in关键字声明同名字的变量

#### interface block(接口块？？)

匹配interface block是通过block名字匹配的，但是允许block实例在不同的阶段拥有不同的名字有两个目的
一是避免疑惑的变量命名，二是可能在不同的阶段从单项变长数组
```cpp

layout (location 0) in vec4 color;

out VS_OUT{
    vec4 color;
} vs_out;


void main()
{
    vs_out.color = color;
}

```

```cpp

in VS_OUT{
    vec4 color;
} fs_in;

void main()
{
    color = fs_in.color;
}

```

### 细分曲面(镶嵌化处理技术)
[百度百科](https://baike.baidu.com/item/%E6%9B%B2%E9%9D%A2%E7%BB%86%E5%88%86/4965377)

细分曲面是一个打破高级基本元素成很多细小的简单的元素（例如三角形）的渲染过程
OpenGL 包括一个可修改的函数，可配置的细分曲面引擎。他可以打破四边形，三角形和线条，打破成大量的小的点，线条或者三角形。这些元素可以被渲染管线下的正常光栅化设备直接消耗
细分曲面位于OpenGL顶点着色器阶段之后, 它由三部分组成：

- 细分曲面控制着色器
- the fixed-function tessellation engine
- the tessellation evaluation shader

#### 细分曲面控制着色器 (TCS)
有时候也叫控制着色器, 这个着色器从顶点着色器获取输入，主要负责两件事：
- 检测被送到细分曲面引擎的细分曲面等级  
- 产生要被送到细分曲面之后的细分曲面评价着色器数据

细分曲面的工作方式是通过打碎高级平面(patchs),打碎成点线或者三角形, 每个patch是通过一定数量的控制点形成的。 这个数量可以通过`glPatchParameteri()`,设置参数`pname`为`GL_PATCH_VERTICES`, `value` 设置成控制点的数量。

```cpp
// pname: GL_PATCH_VERTICES, GL_PATCH_DEFAULT_OUTER_LEVEL, GL_PATCH_DEFAULT_INNER_LEVEL 
// value: 参数的值
void glPatchParameteri(	GLenum pname,
 	GLint value);
```

每个patch默认的控制点是3，最大控制点数量由实现定义，但保证至少为32。

当曲面细分被激活的时候, 顶点着色器在每个控制点运行一次，而曲面细分着色器在一组相同数量的顶点控制点运行。也就是说顶点被用作控制点，并且顶点着色器的结果被成批的传递给曲面细分着色器作为输入。每个碎片的控制点数目可以改变使得曲面细分着色器输出的控制点数目可以与它消耗的控制点数目不同。控制着色器生产的控制点的数量由out layout限定符设置。
```cpp
layout (vertices = N) out;
```
这里，N是每个碎片的控制点的数量，控制着色器是负责计算 控制点的输出 和 设置结果碎片的曲面细分因子，该因子会被送到固定函数细分曲面引擎。细分曲面因子被写成`gl_TessLevelInner` 和 `gl_TessLevelOuter`内置的输出变量，然而其他数据传递下去，一般用户定义的输出变量（使用`out`关键字，或者专用的内置`gl_out`数组）写。

```cpp
#version 450 core
layout (vertices = 3) out; // 输出控制点的数量
void main(void)
{
    // 细分曲面等级设置成5.0
    // 数值越高，会产生越密集的细分曲面输出
    // 数值越低，会产生越粗糙的细分曲面输出
    // 数值为0，patch会被丢掉
    if(gl_InvocationID == 0){
        gl_TessLevelInner[0] = 5.0;
        gl_TessLevelOuter[0] = 5.0;
        gl_TessLevelOuter[1] = 5.0;
        gl_TessLevelOuter[2] = 5.0;
    }
    
    // gl_InvocationID是 gl_in 和 gl_out 数组的 index
    // 值是0 到 当前调用的细分曲面控制着色器的patch的控制点的数量
    gl_out[gl_InvocationID].gl_Position = gl_in[gl_InvocationID].gl_position;
}
```
#### 细分曲面引擎
细分曲面引擎是OpenGL渲染管线的一个固定函数，它把很多碎片表示的高级曲面打碎成简单的元素，例如点线三角形。在细分曲面引擎接收到碎片之前，细分曲面控制着色器处理将要到来的控制点，和设置用于打破碎片的细分曲面因子。在曲面细分引擎产生输出的元素之后，代表他们的顶点会被细分曲面评价着色器拾起，细分曲面引擎主要负责生产提供给细分曲面评价着色器的参数。

#### 细分曲面评价着色器（TES）
当固定函数细分曲面引擎运行的时候，它产生一些表示元素已经生成的输出顶点。他们传递给细分曲面评价着色器。细分曲面评价着色器（也叫做评价着色器）运行的时候，每个从细分曲面器生产的顶点都会调用它。当细分曲面等级很高的时候，细分曲面评价着色器会运行大量次数，因此，你需要小心复杂的细分曲面评价着色器和高级的细分曲面等级。

```cpp
#version 450 core
// triangles 代表三角形模型
// equal_spacing 是指相等的距离
// cw 生成的三角形按顶点按照顺时针顺序
layout (triangles, equal_spacing, cw) in;
void main(void)
{
    // gl_TestCoord 细分曲面器生成的顶点的质心坐标
    // gl_in 是 细分曲面控制着色器的写着gl_out的结构体
    gl_Position =   (gl_TestCoord.x * gl_in[0].gl_Position +
                     gl_TestCoord.y * gl_in[1].gl_Position +
                     gl_TestCoord.z * gl_in[2].gl_Position);
}
```

```cpp
// 绘画轮廓
// face : GL_FRONT_AND_BACK
// mode : 表示多边形怎么光栅化. 接受的值为GL_POINT, GL_LINE, GL_FILL.
void glPolygonMode(	GLenum face,
 	GLenum mode);
```

### 几何着色器
几何着色器逻辑上来说是前端着色器的最后阶段，发生在顶点和细分曲面阶段之后，在光栅化之前。几何着色器针对每个图元运行一次，并且正在处理的图元的所有的顶点数据都有访问权限。几何着色器也是着色器阶段中唯一可以编程控制数据流总量增减的着色器。虽然说细分曲面着色器也可以增减管线的工作量，但是它只通过设置碎片细分曲面的级别来隐式的影响工作量。而几何着色器包含两个函数`EmitVertex()`和`EndPrimitive`他们能显示地产生顶点到图元组装和光栅化。
几何着色器的另一个独特的功能是他们在渲染管线中途改变变图元模式，例如他们以三角形作为输入，然后产生一些点或者线作为输出。或者以独立点创建三角形。

```cpp

#version 450 core
layout (triangles) in;
layout (points, max_vertices=3) out;

// 作用： 转换三角形为顶点。
void main(void){
    int i;
    for( i = 0; i < gl_in.length(); i++)
    {
        gl_Position = gl_in[i].gl_Position;
        EmitVertex(); // 几何着色器的输出中产生一个顶点
    }
    // 结束的时候自动调用 EndPrimitive();
}

```

 













