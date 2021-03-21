# 利用CMake生成代码中的常量字符串

在实现材质系统的时候，往往会有不少的shader代码需要维护，根据具体的材质属性，在运行时进行shader的拼装。而出于解耦考虑和维护的方便，我们通常会把shader的实现分好多个文本文件来存储，而不会直接写在c++的源文件中。如果我们期望在运行时去读取这些文本文件，可以直接打开文件读取内容到string中就行啦。这次主要记录的是在编译之前，就提前将这些shader中的内容转换成常量字符串，并提供字符串常量给源代码中使用，这些shader内容会直接带在可执行文件中。

## CMake的能力

话说有一句“名言”叫cmake is cmake, the others are bullshit，这话虽然有点过分，但是确实有了cmake之后，在项目代码的构建任务中真的是“cmake在手，天下我有”的感觉。这次干的骚操作，也是利用了cmake的能力。

## configure_file

这是高阶CMake中内置的函数，先简要说说用法（详细文档可参考cmake手册或google一下）：
第一个参数是输入的文件，第二个参数是输出的文件。我们指定一个输入的文件，然后cmake会读取这个文件中的内容，并将当前cmake中的变量的值替换成输入文件的内容中的变量。例如

```cmake
configure_file(Shaders.h Shaders.generated.h)
```

那么cmake在构建时，会读取Shaders.h中的内容，然后将Shaders.h中两个“@”中间的字符串找出来，看是否能匹配上当前cmake中的变量，如果匹配上，就会将其替换成cmake中该变量的值，如果没有匹配的变量，就替换成空字符串，最后将这段内容输出到Shaders.generated.h文件里。

有点绕，直接看例子。

我们在Shader.h文件中写了这些代码：

```c++
#ifndef _SHADERS_H_
#define _SHADERS_H_

#include <string>

const std::string SHADER_VS
(R"SHADER(
@SHADER_VS@
)SHADER");

const std::string SHADER_FS
(R"SHADER(
@SHADER_FS@
)SHADER");

#endif
```

如果我们在cmake中设置了两个变量：

```cmake
set(SHADER_VS "shader vs")
set(SHADER_FS "shader fs")
```

那么configure_file之后，会生成Shaders.generated.h文件，并在这个文件中，“@SHADER_VS@”将被替换成"shader vs"，“@SHADER_FS@”将被替换成"shader fs"。最终的Shaders.generated.h将是这样的：

```c++
#ifndef _SHADERS_H_
#define _SHADERS_H_

#include <string>

const std::string SHADER_VS
(R"SHADER(
shader vs
)SHADER");

const std::string SHADER_FS
(R"SHADER(
shader fs
)SHADER");

#endif
```

我们在源代码中，就能直接使用SHADER_VS和SHADER_FS这两个字符串常量做后面的事情啦。

## file

到这还没有结束，还需要从文件中读取shader代码的字符串。

假设我们写了两段shader代码，保存在shader.vs和shader.fs文件中，我们得在构建时自动读取它们，然后转变成当前cmake中的变量。

shader.vs的内容：

```glsl
void main()
{
    gl_Position = vec4(0.0, 0.0, 0.0, 1.0);
}
```

shader.fs的内容：

```glsl
layout (location = 0) out vec4 fragColor;

void main()
{
    fragColor = vec4(1.0, 0.0, 0.0, 1.0);
}
```

我们写一个cmake中的宏函数来做这件事情

```cmake
macro(generate_shader shader_file shader_content)
file(READ ${shader_file} ${shader_content})
endmacro()
```

这样我们就有了一个generate_shader宏函数，第一个参数是shader文件，第二个参数是指定的cmake变量，这个宏函数实现了读取shader文件中的内容到给定的当前cmake变量中。

```cmake
generate_shader(shader.vs SHADER_VS) # 不在需要set(SHADER_VS "shader.vs中的内容")
generate_shader(shader.fs SHADER_FS) # 不在需要set(SHADER_FS "shader.fs中的内容")
```

## 完整示例

完整的CMakeLists如下：

```cmake
cmake_minimum_required(VERSION 3.10)
project(project_configure_file)

macro(generate_shader shader_file shader_content)
file(READ ${shader_file} ${shader_content})
endmacro()

generate_shader(shader.vs SHADER_VS)
generate_shader(shader.fs SHADER_FS)

configure_file(Shaders.h Shaders.generated.h)
```

运行CMake我们将得到生成的带有shader代码常量字符串的头文件Shaders.generated.h，结合前面的Shaders.h和shader.vs & shader.fs，生成的内容将如下：

```c++
#ifndef _SHADERS_H_
#define _SHADERS_H_

#include <string>

const std::string SHADER_VS
(R"SHADER(
void main()
{
    gl_Position = vec4(0.0, 0.0, 0.0, 1.0);
}
)SHADER");

const std::string SHADER_FS
(R"SHADER(
layout (location = 0) out vec4 fragColor;

void main()
{
    fragColor = vec4(1.0, 0.0, 0.0, 1.0);
}
)SHADER");

#endif
```

在源代码中尽情使用吧。

## 协议

本文以上内容遵循CC BY-ND 4.0协议，署名-禁止演绎。

本文中所提供的源代码遵循MIT开源协议。
代码托管于：<https://github.com/KondeU/ShaderStringGeneratedByCmake>

转载请注明出处：<https://tis.ac.cn/blog/kongdeyou/use_cmake_to_generate_constant_string/>
并署名：[kongdeyou(https://tis.ac.cn/blog/author/kongdeyou/)](https://tis.ac.cn/blog/author/kongdeyou/)