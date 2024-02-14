---
title: CMakeList语法
date: 2024-02-14 15:26:41
categories:
- 编译
---



```cmake
cmake_minimum_required(VERSION 3.10)
project(szjcmk)

#cmakedefine AUTOMATIC_CAPACITY 

set(IS_MAKE_LIB ON) # 是否编译动态库

set(TARGET usrApp)  #生成可执行文件的名称

set(SRC_DIR src)    #源代码文件目录
set(LDIR lib)       #动态库依赖目录

# 设置编译器路径
set(COMPILER_PATH /home/daylong/project/10S-SPSJJLMK/SF/CPU/os/gcc-linaro-7.3.1-2018.05-x86_64_arm-linux-gnueabihf/bin)

# 设置编译器
set(CMAKE_COMPILER arm-linux-gnueabihf)

# 设置windows共享路径
set(WIN_PATH /home/daylong/windows/szjcImage)

# 设置项目编译选项
# add_compile_options(-Werror) # 警告当成错误
add_compile_options(-Wall -Wextra ) # 开启大部分警告
# add_compile_options(-pedantic)      # 开启严格的C++编译选项 
add_compile_options(-Warray-bounds) # 开启数组越界检查
add_compile_options(-Waddress)      # 开启地址越界检查
# add_compile_options(-Wconversion-extra) # 开启转换警告
# add_compile_options(-Wfloat-equal)  # 开启浮点数相等警告
add_compile_options(-Wuninitialized)    # 开启未初始化变量检查
add_compile_options(-Wunused-parameter) # 开启未使用参数检查
add_compile_options(-Wnonnull)          # 开启空指针检查

add_compile_options(-Wno-psabi)         # 忽略ABI更改
add_compile_options(-Wno-error=deprecated-declarations -Wno-deprecated-declarations)# 忽略已弃用的声明
add_compile_options(-Wno-type-limits)       # 忽略类型限制警告
# add_compile_options(-Wno-unused-variable)   # 忽略未使用的变量
add_compile_options(-Wno-format-overflow)   # 忽略格式溢出警告

add_compile_options(-D_FILE_OFFSET_BITS=64) # 在32位系统中使用64位文件大小

# 设置全局C++编译选项
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wold-style-cast") # 开启C++的旧式类型转换警告
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17") 

# 设置默认构建类型为 Release
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O0 -g")# 设置Debug版本的编译选项
set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -O0 -g")    # 设置Debug版本的编译选项

set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -Og")# 设置Release版本的编译选项
set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -O3")    # 设置Release版本的编译选项

# set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-O2 -g")  # RelWithDebInfo 构建类型
# set(CMAKE_CXX_FLAGS_MINSIZEREL "-Os -DNDEBUG")  # MinSizeRel 构建类型


message(STATUS "Build type:                                       ${CMAKE_BUILD_TYPE}")
message(STATUS "C flags, Debug configuration:                     ${CMAKE_C_FLAGS_DEBUG}")
message(STATUS "C flags, Release configuration:                   ${CMAKE_C_FLAGS_RELEASE}")
# message(STATUS "C flags, Release configuration with Debug info:   ${CMAKE_C_FLAGS_RELWITHDEBINFO}")
# message(STATUS "C flags, minimal Release configuration:           ${CMAKE_C_FLAGS_MINSIZEREL}")
message(STATUS "C++ flags, Debug configuration:                   ${CMAKE_CXX_FLAGS_DEBUG}")
message(STATUS "C++ flags, Release configuration:                 ${CMAKE_CXX_FLAGS_RELEASE}")
# message(STATUS "C++ flags, Release configuration with Debug info: ${CMAKE_CXX_FLAGS_RELWITHDEBINFO}")
# message(STATUS "C++ flags, minimal Release configuration:         ${CMAKE_CXX_FLAGS_MINSIZEREL}")

# 设置交叉编译器路径
if(COMPILER_PATH)
    set(CMAKE_C_COMPILER ${COMPILER_PATH}/${CMAKE_COMPILER}-gcc)
    set(CMAKE_CXX_COMPILER ${COMPILER_PATH}/${CMAKE_COMPILER}-g++)
    set(CMAKE_STRIP ${COMPILER_PATH}/${CMAKE_COMPILER}-strip)
else()
    set(CMAKE_C_COMPILER ${CMAKE_COMPILER}-gcc)
    set(CMAKE_CXX_COMPILER ${CMAKE_COMPILER}-g++)
    set(CMAKE_STRIP ${CMAKE_COMPILER}-strip)
endif()

# 设置源代码文件
aux_source_directory(${SRC_DIR} SRC_FILES)

# 设置头文件路径
include_directories(
    inc
    lib/inc
    lib/inc/spdlog
    )

# 设置库文件路径
link_directories(${LDIR})

# 添加可执行文件
add_executable(${TARGET} ${SRC_FILES})

if(IS_MAKE_LIB)
    add_subdirectory(
        ${CMAKE_SOURCE_DIR}/user_so # 添加子目录
        ${CMAKE_BINARY_DIR}/user_so # 添加子目录编译后的目标文件目录
        )

    add_dependencies(${TARGET} fpga) # 添加依赖关系
endif()

# 添加链接库
target_link_libraries(${TARGET}
        fpga  # 添加libfpga.so链接库
        pthread  # 添加libpthread.so链接库
        )

# 设置输出目录
if(CMAKE_BUILD_TYPE MATCHES "Debug")
    set(EXECUTABLE_OUTPUT_PATH ./debug)
else()
    set(EXECUTABLE_OUTPUT_PATH ./release)
endif()
# set(EXECUTABLE_OUTPUT_PATH .)

# 添加链接前的自定义命令
if(CMAKE_BUILD_TYPE MATCHES "Debug")
    add_custom_command(TARGET ${TARGET}  # 指定目标
        PRE_BUILD  # 指定在编译之前执行
        COMMENT $<$<CONFIG:Debug> : "正在执行 ${EXECUTABLE_OUTPUT_PATH}/${TARGET} (Debug模式)..."
    )
else()
    add_custom_command(TARGET ${TARGET} PRE_BUILD
        COMMENT $<$<CONFIG:Release> : "正在执行 ${EXECUTABLE_OUTPUT_PATH}/${TARGET} (Release模式)..."
    )
endif()

# 编译完成后执行Shell命令
if(CMAKE_BUILD_TYPE MATCHES "Debug")
    add_custom_command(TARGET ${TARGET} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different ${EXECUTABLE_OUTPUT_PATH}/${TARGET} ${WIN_PATH}
        COMMAND echo "[100%] COPY ${EXECUTABLE_OUTPUT_PATH}/${TARGET} to ${WIN_PATH}")
else()
    add_custom_command(TARGET ${TARGET} POST_BUILD
        COMMAND ${CMAKE_STRIP} ${EXECUTABLE_OUTPUT_PATH}/${TARGET}
        COMMAND echo "[100%] Strip ${EXECUTABLE_OUTPUT_PATH}/${TARGET}"
        COMMAND ${CMAKE_COMMAND} -E copy_if_different ${EXECUTABLE_OUTPUT_PATH}/${TARGET} ${WIN_PATH}
        COMMAND echo "[100%] COPY ${EXECUTABLE_OUTPUT_PATH}/${TARGET} to ${WIN_PATH}")
endif()

# ${CMAKE_COMMAND} -E copy_if_different 是一个 CMake 提供的命令行工具，用于复制文件。这个命令会在编译过程中执行。

# ${CMAKE_COMMAND} 是一个 CMake 变量，它包含了 CMake 的执行路径。

# -E copy_if_different 是一个选项，表示执行复制操作，并且只会在源文件和目标文件不同时才进行复制。
# 这意味着如果源文件和目标文件相同，则不会执行复制操作，以避免重复复制。
cmake_minimum_required(VERSION 3.10)

set(TARGET fpga)

aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR}/src FPGA_SRCS)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=armv7") # -march=armv7

#生成动态库
add_library(${TARGET} SHARED ${FPGA_SRCS})

#设置输出目录
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY  ${CMAKE_CURRENT_SOURCE_DIR}/lib)

# 编译完成后执行Shell命令
if(CMAKE_BUILD_TYPE MATCHES "Debug")
    add_custom_command(TARGET ${TARGET} POST_BUILD
        # 复制动态库及头文件到指定目录
        COMMAND ${CMAKE_COMMAND} -E copy_if_different lib${TARGET}.so ${CMAKE_SOURCE_DIR}/lib
        # COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_SOURCE_DIR}/src/fpga.h ${CMAKE_SOURCE_DIR}/lib/inc

        # 复制动态库到共享目录
        COMMAND ${CMAKE_COMMAND} -E copy_if_different lib${TARGET}.so ${WIN_PATH}
        COMMAND echo "[----] COPY lib${TARGET}.so to ${WIN_PATH}")
else()
    add_custom_command(TARGET ${TARGET} POST_BUILD
    
        COMMAND ${CMAKE_STRIP} lib${TARGET}.so
        COMMAND echo "[----] Strip lib${TARGET}.so"
        # 复制动态库及头文件到指定目录
        COMMAND ${CMAKE_COMMAND} -E copy_if_different lib${TARGET}.so ${CMAKE_SOURCE_DIR}/lib
        # COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_SOURCE_DIR}/src/fpga.h ${CMAKE_SOURCE_DIR}/lib/inc        

        # 复制动态库到共享目录
        COMMAND ${CMAKE_COMMAND} -E copy_if_different lib${TARGET}.so ${WIN_PATH}
        COMMAND echo "[----] COPY lib${TARGET}.so to ${WIN_PATH}")
endif()
```

