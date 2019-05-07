# CMake学习笔记
[TOC]

## 一、初次使用

### 1. 初次见面

编译`CMakeLists.txt`文件需要有以下三行:

```cmake
# 指定最低版本号
cmake_minimum_required(VERSION 3.4.1) 

# 指定项目名称
project(Project_CHello)

# 添加目标可执行文件
add_executable(CHello main.cpp )
```

**注意:**

*   cmake的函数调用大小写不敏感, 但参数、字符串大小写敏感, 以下写法均正确:
    -   `cmake_minimum_required(VERSION 3.4.1)`
    -   `cmake_MINIMUM_required(VERSION 3.4.1)`
    -   `CMAKE_MINIMUM_REQUIRED(VERSION 3.4.1)`

### 2. 初次尝鲜

与`CMakeLists.txt`文件在同一目录下, 编写一个c++文件`main.cpp`, 如下:

```c++
#include <iostream>

int main() {

    std::cout << "hello world" << std::endl;

    return 0;
}
```

执行cmake命令:

```shell
cmake ./CMakeLists.txt && make
```

目录文件生成:

```shell
-rwxr-xr-x 1 root staff  9216 May  5 18:17 CHello*
-rw-r--r-- 1 root staff 11507 May  5 18:17 CMakeCache.txt
drwxr-sr-x 5 root staff  4096 May  5 18:17 CMakeFiles/
-rw-r--r-- 1 root staff  1357 May  5 18:17 cmake_install.cmake
-rw-r--r-- 1 root staff   196 May  5 18:16 CMakeLists.txt
-rw-r--r-- 1 root staff   110 May  5 18:16 main.cpp
-rw-r--r-- 1 root staff  4702 May  5 18:17 Makefile
```

`cmake`的作用就是生成`Makefile`之类的文件; `make`的作用就是利用`Makefile`进行编译

生成的文件中包含`CHello`文件, 这个就是打包好的可执行文件

## 二、常用操作

### 1. 添加源文件

#### 1.1方式一: 手动添加每个文件

```cmake
add_executable(
	CHello
	main.h #这里指定多个文件
	main.cpp
)
```

#### 1.2 方式二: 添加某个/多个目录下的源文件

```cmake
# 查找某个目录下的源文件, 不会查询子目录, 源文件里面的依赖会自动添加
# 并将查询到的原文件保存到DIR_SRCS变量
aux_source_directory(. dir_cur)
aux_source_directory(./math dir_math)
add_executable(
	CHello
	${dir_cur}
	${dir_math}
)
# 可以打印消息查看目录
messgae(cur     ${dir_cur})
messgae(math ${dir_math})
```

#### 1.3 方式三: 添加子目录模块

```cmake
# 添加子目录下的模块
add_subdirectory(math)
# 生成目标文件
add_executable(CHello main.cpp)
# 添加链接库, Math是子目录模块打包出的库
target_link_libraries(CHello Math)
```

子目录下也要有一个CMakeLists.txt文件, 这条语句工作时, 会将子目录下的`CMakeLists.txt`运行一次, 那么子目录下的源代码也就被处理为一个单独的模块, 接下来要做的就是将子目录下的模块链接下就可以了

子模块`CMakeLists.txt`编写如下:

```cmake
aux_source_directory(. DIR_LIB_SRCS)
add_library (Math ${DIR_LIB_SRCS})
```

### 2. 添加头文件目录

```cmake
include_directories(/usr/lib/jvm/java-8-openjdk-amd64/include/jni.h)
```

作用相当于g++选项中的`-I`参数，也相当于环境变量中增加路径到`CPLUS_INCLUDE_PATH`变量

### 3. 添加链接库目录

```cmake
link_directories(/home/server/third/lib)
```

作用相当于g++命令的`-L`选项的，也相当于环境变量中增加路径到`LD_LIBRARY_PATH`变量

### 4. 添加链接库到目标

```cmake
# 连接libmath.so库，默认优先链接动态库
target_link_libraries(CHello math)
# 链接静态链接库
target_link_libraries(CHello libmath.a)
# 链接动态链接库
target_link_libraries(CHello libmath.so)
```

### 5. 设置变量

```cmake
# 给MY_VAR变量设置值
set(
	MY_VAR
	hello
)
# 将MY_VAR的值打印出来
message(MY_VAR)
```

### 6. 设置C++标准

```cmake
# 设置C++标注为11
set(CMAKE_CXX_STANDARD 11)
```

### 7. 设置输出目录

```cmake
# 设置执行文件输出目录
set(
	EXECUTABLE_OUTPUT_PATH 
	"/shared/Projects/CHello"
)

 # 设置库输出目录
set(
	LIBRARY_OUTPUT_PATH
	"/shared/Projects/CHello"
)
```

### 8. 分支语句(if/else)

```cmake
set(MY_VAR 0)
set(MY_VAR_1 2)

# 变量设置为false、0或者空字符串为假, 其他为真
if (MY_VAR)
    message("1")
elseif (MY_VAR_1)
    message("2")
else ()
    message("3")
endif ()

# 字符串比较, 建议加上{}和"", 避免没有定义AAA变量时出现问题
if ("${AAA}" STREQUAL "")
    message("aaa")
endif ()
```

### 9. 定义选项(option)

```cmake
# 这里定义选项, 值为ON或者OFF, 非ON时默认为OFF
option(my_option "this is my option" ON)

# 这里会输出ON
message(${my_option})
```

### 10. 重新生成缓存

cmake没有自带清除缓存功能, 如果之前用cmake生成过, 再次使用cmake时, CMakeLists.txt的修改并不会生效.

需要先删除缓存文件`CMakeCache.txt`, 再使用cmake生成即可

```shell
cd build
rm -f CMakeCache.txt
cmake ..
```

### 11. 添加其他CMakeLists.txt文件

```cmake
include(option.txt)
```

### 12. 动态生成宏定义文件

需要一个辅助文件`config.h.in`, 并生成一个`config.h`目标文件.

```cmake
set(my_var_1 hello)
option(my_option_1 "this is my option" on)
option(my_option_2 "this is my option" off)
configure_file(
        ${PROJECT_SOURCE_DIR}/config.h.in
        ${PROJECT_SOURCE_DIR}/config.h
)
```

如上所示, 在CMakeLists.txt文件中定义`my_var`, `my_option`变量, 使用`configure_file`配置输入文件和输出文件

```c++
#cmakedefine my_var_1 @my_var_1@
#cmakedefine my_var_2 @my_var_2@
#cmakedefine my_option_1 @my_option_1@
#cmakedefine my_option_2 @my_option_2@

#define MY_VAR_1 @my_var_1@
#define MY_VAR_2 @my_var_2@
#define MY_OPTION_1 @my_option_1@
#define MY_OPTION_2 @my_option_2@
```

如上所示, 在源文件目录创建`config.h.in`文件:

```c++
#define my_var_1 hello
/* #undef my_var_2 */
#define my_option_1 ON
/* #undef my_option_2 OFF*/

#define MY_VAR_1 hello
#define MY_VAR_2 
#define MY_OPTION_1 ON
#define MY_OPTION_2 OFF
```

如上所示, 生成的`config.h`文件. 生成具体细节如下:

*   `#cmakedefine`: 如果在CMakeLists.txt里面用`set`定义了值(值不为空/false/off)或者`option`的值为`on`, 那么就生成`#define 变量名`格式, 否则就是`/* #undef 变量名*/`格式
*   `#define`: 不管值何变量是否定义过, 始终生成`#define 变量名格式`
*   `@变量名@`: 取变量的值