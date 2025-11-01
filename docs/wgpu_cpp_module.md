# 将 Dawn C++ 包装器封装成 C++ Module

本文档说明如何在使用 Dawn 提供的 `webgpu.hpp` 时，正确地在 C++20 Modules 中处理 `WEBGPU_CPP_IMPLEMENTATION` 宏。

## 背景

`webgpu.hpp` 使用 `WEBGPU_CPP_IMPLEMENTATION` 宏来控制实现段落的生成。按照传统头文件用法，需要在 **恰好一个** 翻译单元中在包含头文件前定义该宏，以生成辅助函数、回调适配器等实现代码。若改用 C++ Modules，需要将这一约定映射到模块接口单元与实现单元之间。

## 模块结构

可以将模块拆分成一个接口单元与一个实现单元：

1. **接口单元**：负责导出 Dawn C++ API，不定义 `WEBGPU_CPP_IMPLEMENTATION`。
2. **实现单元**：只负责生成实现代码，在包含前定义 `WEBGPU_CPP_IMPLEMENTATION`。

这两个单元既可以各自作为独立的模块单元，也可以采用 **内部模块分区（module partition）** 的形式来组织。

### 为什么不直接在接口单元里定义宏？

理论上确实可以在接口单元的全局模块片段中 `#define WEBGPU_CPP_IMPLEMENTATION` 后立即包含 `webgpu.hpp`，这样同一个文件即可同时提供声明与实现。但在实践中将实现放进单独单元更有优势：

- **减小 BMI（编译后的模块接口文件）影响面**：`WEBGPU_CPP_IMPLEMENTATION` 会生成大量非内联的函数体与适配器。如果它们出现在接口单元里，接口变动将迫使所有导入该模块的目标重新编译；把实现移到实现单元/分区后，接口 BMI 只包含符号声明，实现细节的修改不会触发级联重编译。
- **保持实现细节私有**：开启该宏会生成像 `createInstance`、回调桥接等自由函数，它们只是模块内部的粘合代码，不需要暴露给使用者。通过内部实现单元或分区，它们只在模块内部可见，接口层仍旧只导出 `wgpu` 命名空间下的 C++ API。
- **兼容传统“单翻译单元”约定**：项目中可能还有旧的 `.cpp` 文件以头文件方式包含 `webgpu.hpp`。维持“只有实现单元定义该宏”的结构，更容易检查并避免因为额外的 `#define` 而产生重复定义的混用问题。

因此本文推荐继续沿用“接口声明 + 单一实现单元”的布局，同时说明如果项目规模较小、确实希望只用一个模块文件，也可以在全局模块片段里定义该宏。

### 接口单元示例

```cpp
// file: src/wgpu_cpp.ixx
export module wgpu_cpp;

module;  // 全局模块片段，用于包含 C 头与 Dawn 头
#include <webgpu/webgpu.h>
#include <webgpu/webgpu.hpp>

export namespace wgpu {
    using namespace ::wgpu; // 直接导出 Dawn 提供的符号
}
```

关键点：

- 接口单元中**不要**定义 `WEBGPU_CPP_IMPLEMENTATION`。
- `module;` 全局片段允许在模块内部继续使用 Dawn 头文件的宏定义。
- 通过 `using namespace ::wgpu;` 将 Dawn 的命名空间内容整体导出，避免重新包装。

### 实现单元示例

```cpp
// file: src/wgpu_cpp_impl.cppm
module wgpu_cpp;

#define WEBGPU_CPP_IMPLEMENTATION
#include <webgpu/webgpu.hpp>
```

这里实现单元只负责生成实现代码，不需要 `export`。由于模块实现会与接口单元共享同一个命名空间实例，它会满足 "恰好一个编译单元定义宏" 的要求。

### 使用内部模块分区的写法

如果希望把实现放进模块的内部分区，可以将实现单元声明为 `module wgpu_cpp:impl;`，并在接口单元中 `export module wgpu_cpp;` 时引入该分区：

```cpp
// file: src/wgpu_cpp.ixx
export module wgpu_cpp;

module;
#include <webgpu/webgpu.h>
#include <webgpu/webgpu.hpp>

export namespace wgpu {
    using namespace ::wgpu;
}

import :impl; // 导入内部实现分区，但不对外导出
```

```cpp
// file: src/wgpu_cpp_impl.cppm
module wgpu_cpp:impl;

#define WEBGPU_CPP_IMPLEMENTATION
#include <webgpu/webgpu.hpp>
```

在这种布局下，接口单元最后的 `import :impl;` 会把实现分区编译进同一个模块，但不会导出给使用者；依然只有实现分区定义了 `WEBGPU_CPP_IMPLEMENTATION`，满足 Dawn 的要求。

## CMake 集成示例

以下示例假设使用 CMake 3.28+，并且 `wgpu` 目标已经由本仓库提供的 CMake 脚本配置好：

```cmake
add_library(wgpu_cpp STATIC
    src/wgpu_cpp.ixx
    src/wgpu_cpp_impl.cppm
)

target_link_libraries(wgpu_cpp PUBLIC wgpu)

target_sources(wgpu_cpp
    PUBLIC FILE_SET cxx_modules TYPE CXX_MODULES FILES src/wgpu_cpp.ixx
    PRIVATE FILE_SET cxx_modules TYPE CXX_MODULES FILES src/wgpu_cpp_impl.cppm
)
```

- 接口单元放入 `PUBLIC` 的 module file set 中，供依赖方导入。
- 实现单元保持 `PRIVATE`，但依旧属于同一个模块名称 `wgpu_cpp`。
- 没有任何地方需要在传统意义上的 `translation unit` 里重复定义宏。

## 验证建议

- 在模块实现单元中加入 `static_assert(&wgpu::Instance::Create == nullptr);` 等引用，以确认实现确实被编译。
- 若链接器提示重复定义，检查是否有额外的 `.cpp` 文件定义了 `WEBGPU_CPP_IMPLEMENTATION`。
- 如果工程仍旧以传统头文件方式包含 `webgpu.hpp`，务必确保旧的定义点被移除，否则会出现重复符号。

通过上述结构，就可以在保留 Dawn 官方头文件的前提下，把其封装进现代 C++ Module，同时继续遵守 `WEBGPU_CPP_IMPLEMENTATION` 的约束。
