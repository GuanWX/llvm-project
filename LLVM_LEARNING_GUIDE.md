# LLVM 入门学习指南

本文档基于当前仓库（LLVM 14.x）的实际代码结构，系统整理了 LLVM 的仓库结构与入门学习思路，帮助初学者快速建立对 LLVM 项目的整体认识。

---

## 目录

1. [仓库顶层结构总览](#1-仓库顶层结构总览)
2. [LLVM 核心架构概述](#2-llvm-核心架构概述)
3. [LLVM Core（llvm/ 目录）详解](#3-llvm-core-llvm-目录详解)
4. [Clang 前端（clang/ 目录）详解](#4-clang-前端-clang-目录详解)
5. [其他重要子项目简介](#5-其他重要子项目简介)
6. [构建方法](#6-构建方法)
7. [入门学习路线](#7-入门学习路线)
8. [推荐阅读资源](#8-推荐阅读资源)

---

## 1. 仓库顶层结构总览

```
llvm-project/
├── llvm/                  # LLVM 核心：IR、优化 Pass、代码生成、工具链
├── clang/                 # C/C++/ObjC 前端编译器
├── clang-tools-extra/     # 基于 Clang 的额外工具（clang-tidy、clangd 等）
├── lld/                   # LLVM 链接器
├── lldb/                  # LLVM 调试器
├── compiler-rt/           # 运行时库（sanitizer、builtins 等）
├── libcxx/                # C++ 标准库（libc++）实现
├── libcxxabi/             # C++ ABI 库
├── libunwind/             # 栈展开库
├── mlir/                  # 多层中间表示框架（面向 DSL 和机器学习编译器）
├── flang/                 # Fortran 前端
├── openmp/                # OpenMP 运行时
├── polly/                 # 多面体优化框架
├── bolt/                  # 二进制优化和布局工具
├── libc/                  # LLVM 的 C 标准库实现
├── libclc/                # OpenCL C 库
├── cross-project-tests/   # 跨项目集成测试
├── pstl/                  # 并行 STL 实现
├── cmake/                 # 公共 CMake 模块
├── runtimes/              # 运行时库统一构建入口
├── third-party/           # 第三方依赖
├── utils/                 # 公共工具脚本
├── README.md              # 项目说明
├── CONTRIBUTING.md        # 贡献指南
└── SECURITY.md            # 安全策略
```

> **核心关系**：源代码 → **Clang（前端）** → LLVM IR → **LLVM（中端优化）** → 机器码 → **LLD（链接器）** → 可执行文件

---

## 2. LLVM 核心架构概述

LLVM 的编译流程可以抽象为三段式架构：

```
┌──────────┐     ┌──────────────┐     ┌────────────┐
│  前端     │     │  中端（优化） │     │  后端       │
│ (Clang)  │ ──→ │  (LLVM IR)   │ ──→ │ (CodeGen)  │
│ 源码→AST │     │  Pass 管道    │     │ IR→机器码   │
└──────────┘     └──────────────┘     └────────────┘
```

**关键概念**：

| 概念 | 说明 |
|------|------|
| **LLVM IR** | LLVM 的中间表示，是一种强类型、SSA（静态单赋值）形式的低级语言，有三种表示：内存中的 C++ 对象、磁盘上的 bitcode（.bc）、人类可读的文本（.ll） |
| **Pass** | 对 IR 进行分析或变换的模块化单元，是 LLVM 优化的核心机制 |
| **Module** | LLVM IR 的顶层容器，对应一个编译单元（一个 .c/.cpp 文件） |
| **Function** | Module 中的函数 |
| **BasicBlock** | 函数中的基本块，以终结指令（br、ret 等）结尾 |
| **Instruction** | 基本块中的单条 IR 指令 |
| **Value / Use** | LLVM 中几乎所有东西都是 Value；Use 表示值的使用关系（def-use 链） |
| **SelectionDAG** | 后端代码生成中将 IR 转换为目标相关机器指令的 DAG 表示 |
| **TableGen** | LLVM 自有的领域特定语言，用于描述目标架构（指令集、寄存器等） |

---

## 3. LLVM Core（llvm/ 目录）详解

```
llvm/
├── include/llvm/           # 公共头文件（API 定义）
│   ├── IR/                 # IR 相关类的声明（Module, Function, Instruction...）
│   ├── Transforms/         # 优化 Pass 的头文件
│   ├── Analysis/           # 分析 Pass 的头文件
│   ├── CodeGen/            # 代码生成的头文件
│   ├── Target/             # 目标架构抽象接口
│   ├── Support/            # 基础支持库（字符串、文件、命令行等）
│   ├── ADT/                # 抽象数据类型（SmallVector, StringRef, DenseMap 等）
│   ├── MC/                 # 机器码层（汇编/反汇编）
│   ├── TableGen/           # TableGen 相关
│   └── Pass.h / PassManager.h  # Pass 基础设施
│
├── lib/                    # 核心实现
│   ├── IR/                 # ★ IR 实现（Value, Module, Function, BasicBlock, Instruction...）
│   ├── Transforms/         # ★ 优化 Pass 实现
│   │   ├── Scalar/         #   标量优化（循环优化、GVN、SROA 等）
│   │   ├── IPO/            #   过程间优化（内联、死函数消除等）
│   │   ├── Vectorize/      #   向量化优化
│   │   ├── InstCombine/    #   指令合并优化
│   │   ├── Utils/          #   优化工具函数
│   │   └── Hello/          #   ★ Hello World Pass 示例（入门必看！）
│   ├── Analysis/           # 分析 Pass（别名分析、循环分析、支配树等）
│   ├── CodeGen/            # 代码生成（指令选择、寄存器分配、指令调度等）
│   │   ├── SelectionDAG/   #   SelectionDAG 指令选择
│   │   └── GlobalISel/     #   全局指令选择（新一代）
│   ├── Target/             # ★ 各目标架构后端实现
│   │   ├── X86/            #   x86/x86-64 后端
│   │   ├── AArch64/        #   ARM 64 位后端
│   │   ├── RISCV/          #   RISC-V 后端
│   │   ├── ARM/            #   ARM 32 位后端
│   │   ├── Mips/           #   MIPS 后端
│   │   └── ...             #   更多目标架构
│   ├── AsmParser/          # LLVM IR 文本解析
│   ├── Bitcode/            # Bitcode 读写
│   ├── MC/                 # 机器码层（汇编器、反汇编器）
│   ├── Linker/             # IR 级别链接
│   ├── LTO/                # 链接时优化
│   ├── ExecutionEngine/    # JIT 执行引擎（ORC JIT）
│   ├── Support/            # 基础支持库实现
│   ├── TableGen/           # TableGen 解析器
│   └── Passes/             # Pass 管道构建
│
├── tools/                  # ★ 命令行工具（理解 LLVM 功能的最佳入口）
│   ├── opt/                #   IR 优化工具：运行指定的 Pass 对 .ll/.bc 进行优化
│   ├── llc/                #   IR → 汇编/目标文件
│   ├── lli/                #   IR 解释执行 / JIT 执行
│   ├── llvm-as/            #   .ll → .bc（文本 IR 转 bitcode）
│   ├── llvm-dis/           #   .bc → .ll（bitcode 转文本 IR）
│   ├── llvm-link/          #   IR 级别链接多个 .bc 文件
│   ├── llvm-mc/            #   汇编器/反汇编器
│   ├── llvm-objdump/       #   目标文件查看工具
│   ├── llvm-nm/            #   符号表查看
│   ├── llvm-ar/            #   归档工具
│   ├── llvm-readobj/       #   ELF/COFF/MachO 文件读取
│   ├── bugpoint/           #   自动化 bug 缩减工具
│   └── ...
│
├── examples/               # ★ 示例代码（入门必看！）
│   ├── Kaleidoscope/       #   ★ 官方教程：实现一个简单语言的完整编译器
│   │   ├── Chapter2/       #     词法分析 + 语法分析
│   │   ├── Chapter3/       #     生成 LLVM IR
│   │   ├── Chapter4/       #     添加 JIT 和优化
│   │   ├── Chapter5/       #     控制流（if/then/else, for）
│   │   ├── Chapter6/       #     用户自定义运算符
│   │   ├── Chapter7/       #     可变变量
│   │   ├── Chapter8/       #     编译到目标文件
│   │   ├── Chapter9/       #     添加调试信息
│   │   └── BuildingAJIT/   #     构建 JIT 编译器的系列教程
│   ├── IRTransforms/       #   IR 变换示例（SimplifyCFG）
│   ├── ModuleMaker/        #   如何用 C++ API 创建 LLVM Module
│   ├── HowToUseJIT/        #   JIT 使用示例
│   └── HowToUseLLJIT/      #   LLJIT 使用示例
│
├── docs/                   # 文档（.rst 格式，可用 Sphinx 构建为 HTML）
│   ├── tutorial/           #   ★ 教程文档
│   │   ├── LangImpl01-10.rst   # Kaleidoscope 教程各章节
│   │   └── BuildingAJIT*.rst   # JIT 构建教程
│   ├── LangRef.rst         #   ★ LLVM IR 语言参考手册（最重要的参考文档）
│   ├── ProgrammersManual.rst   ★ 程序员手册（LLVM C++ API 使用指南）
│   ├── WritingAnLLVMPass.rst   ★ 编写 LLVM Pass（旧 Pass 管理器）
│   ├── WritingAnLLVMNewPMPass.rst  ★ 编写 LLVM Pass（新 Pass 管理器）
│   ├── WritingAnLLVMBackend.rst    编写 LLVM 后端
│   ├── GettingStarted.rst  #   快速入门
│   ├── CodingStandards.rst #   编码规范
│   ├── CodeGenerator.rst   #   代码生成器文档
│   └── ...
│
├── test/                   # 回归测试（lit 测试框架）
├── unittests/              # 单元测试（Google Test）
└── utils/                  # 开发工具脚本
```

---

## 4. Clang 前端（clang/ 目录）详解

```
clang/
├── include/clang/          # 公共头文件
├── lib/                    # 核心实现
│   ├── Lex/                # 词法分析（预处理器、Token 流）
│   ├── Parse/              # 语法分析（递归下降解析器）
│   ├── AST/                # 抽象语法树（AST）定义与操作
│   ├── Sema/               # 语义分析（类型检查、重载决议等）
│   ├── CodeGen/            # AST → LLVM IR 的代码生成
│   ├── Analysis/           # 静态分析框架
│   ├── StaticAnalyzer/     # Clang 静态分析器
│   ├── Driver/             # 编译器驱动（命令行参数处理、工具链调用）
│   ├── Frontend/           # 前端基础设施
│   ├── Format/             # clang-format 实现
│   └── Tooling/            # LibTooling（构建 Clang 工具的库）
├── tools/                  # Clang 工具
│   ├── driver/             # clang 可执行文件入口
│   └── ...
├── examples/               # 示例
├── test/                   # 测试
└── docs/                   # 文档
```

> **Clang 编译流程**：源代码 → 预处理 → 词法分析(Lex) → 语法分析(Parse) → AST → 语义分析(Sema) → LLVM IR(CodeGen) → 交给 LLVM 后端

---

## 5. 其他重要子项目简介

| 子项目 | 路径 | 说明 |
|--------|------|------|
| **LLD** | `lld/` | LLVM 链接器，支持 ELF、COFF、MachO、WebAssembly 等格式 |
| **LLDB** | `lldb/` | 基于 LLVM/Clang 的调试器 |
| **compiler-rt** | `compiler-rt/` | 运行时库：sanitizer（ASan, MSan, TSan）、builtins、profile 等 |
| **libc++** | `libcxx/` | LLVM 的 C++ 标准库实现 |
| **MLIR** | `mlir/` | 多层 IR 框架，用于构建可扩展的编译器基础设施，广泛应用于 AI 编译器 |
| **Polly** | `polly/` | 基于多面体模型的循环优化器 |
| **BOLT** | `bolt/` | 二进制级别的优化工具，通过重排布局提升性能 |
| **Flang** | `flang/` | LLVM 的 Fortran 前端 |

---

## 6. 构建方法

### 最小构建（仅 LLVM Core）

```bash
# 克隆代码
git clone https://github.com/llvm/llvm-project.git
cd llvm-project

# 创建构建目录
cmake -S llvm -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DLLVM_ENABLE_ASSERTIONS=ON

# 构建
cmake --build build -j$(nproc)
```

### 构建 LLVM + Clang

```bash
cmake -S llvm -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DLLVM_ENABLE_PROJECTS="clang" \
  -DLLVM_ENABLE_ASSERTIONS=ON

cmake --build build -j$(nproc)
```

### 运行测试

```bash
# 运行所有 LLVM 回归测试
cmake --build build --target check-llvm

# 运行所有 Clang 测试
cmake --build build --target check-clang
```

### 推荐的 Debug 构建选项

```bash
cmake -S llvm -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DLLVM_ENABLE_PROJECTS="clang" \
  -DLLVM_ENABLE_ASSERTIONS=ON \
  -DLLVM_TARGETS_TO_BUILD="X86" \   # 只构建 X86 后端，大幅加速
  -DLLVM_USE_LINKER=lld \            # 使用 lld 链接加速
  -DCMAKE_EXPORT_COMPILE_COMMANDS=ON # 生成编译数据库，方便 IDE
```

---

## 7. 入门学习路线

### 阶段一：理解 LLVM IR（1-2 周）

**目标**：理解 LLVM IR 是什么、能表达什么、怎么读写。

1. **阅读入门文档**
   - `llvm/docs/GettingStarted.rst` — 了解 LLVM 项目全貌
   - `llvm/docs/LangRef.rst` — LLVM IR 语言参考（先浏览框架，遇到具体指令再查）

2. **动手实验 LLVM IR**
   ```bash
   # 写一个简单的 C 程序 hello.c
   echo 'int add(int a, int b) { return a + b; }' > /tmp/hello.c

   # 用 Clang 生成 LLVM IR
   build/bin/clang -S -emit-llvm /tmp/hello.c -o /tmp/hello.ll

   # 查看 IR
   cat /tmp/hello.ll

   # 文本 IR → bitcode
   build/bin/llvm-as /tmp/hello.ll -o /tmp/hello.bc

   # bitcode → 文本 IR
   build/bin/llvm-dis /tmp/hello.bc -o /tmp/hello2.ll

   # 用 opt 运行优化
   build/bin/opt -O2 -S /tmp/hello.ll -o /tmp/hello_opt.ll

   # 用 llc 编译为汇编
   build/bin/llc /tmp/hello.ll -o /tmp/hello.s
   ```

3. **阅读关键源码**
   - `llvm/include/llvm/IR/Value.h` — 理解 Value，LLVM 中最基础的类
   - `llvm/include/llvm/IR/Instruction.h` — 指令类
   - `llvm/include/llvm/IR/BasicBlock.h` — 基本块
   - `llvm/include/llvm/IR/Function.h` — 函数
   - `llvm/include/llvm/IR/Module.h` — 模块（顶层容器）

### 阶段二：跟随 Kaleidoscope 教程（2-3 周）

**目标**：通过实现一个小型语言编译器，理解前端到 LLVM IR 的完整流程。

1. **按章节跟做教程**
   - 教程文档：`llvm/docs/tutorial/LangImpl01.rst` 到 `LangImpl10.rst`
   - 配套代码：`llvm/examples/Kaleidoscope/Chapter2/` 到 `Chapter9/`

   | 章节 | 内容 | 核心知识点 |
   |------|------|-----------|
   | Ch1 | 概述 | LLVM 生态介绍 |
   | Ch2 | 词法/语法分析 | Lexer、Parser、AST |
   | Ch3 | **生成 LLVM IR** | IRBuilder、Module、Function、BasicBlock |
   | Ch4 | **添加 JIT 和优化** | Pass Manager、FPM、JIT 执行 |
   | Ch5 | 控制流 | Phi 节点、SSA 形式 |
   | Ch6 | 用户定义运算符 | 运算符优先级解析 |
   | Ch7 | 可变变量 | alloca + mem2reg 优化 |
   | Ch8 | 编译到目标文件 | TargetMachine、文件输出 |
   | Ch9 | 调试信息 | DWARF、DIBuilder |

2. **同时阅读**
   - `llvm/docs/ProgrammersManual.rst` — LLVM C++ API 使用指南
   - 了解 `SmallVector`、`StringRef`、`DenseMap` 等 LLVM 自有的 ADT 数据结构

### 阶段三：学习编写 LLVM Pass（1-2 周）

**目标**：学习 LLVM 的 Pass 机制，编写自己的分析和变换 Pass。

1. **从 Hello World Pass 开始**
   - 源码：`llvm/lib/Transforms/Hello/` — 最简单的 Pass 示例
   - 文档：`llvm/docs/WritingAnLLVMNewPMPass.rst`（新 Pass 管理器）
   - 旧文档：`llvm/docs/WritingAnLLVMPass.rst`（旧 Pass 管理器，了解即可）

2. **学习 IR 变换示例**
   - `llvm/examples/IRTransforms/` — SimplifyCFG 示例

3. **阅读现有 Pass 源码**
   - `llvm/lib/Transforms/Scalar/` — 标量优化（从简单的开始读，如 DCE、ADCE）
   - `llvm/lib/Transforms/InstCombine/` — 指令合并
   - `llvm/lib/Transforms/IPO/` — 过程间优化
   - `llvm/lib/Analysis/` — 分析 Pass（DominatorTree、LoopInfo 等）

4. **动手用 opt 工具调试 Pass**
   ```bash
   # 查看可用的 Pass
   build/bin/opt --print-passes

   # 运行特定 Pass 并查看 IR 变化
   build/bin/opt -passes="instcombine" -S /tmp/hello.ll -o /tmp/hello_ic.ll

   # 对比优化前后的 IR
   diff /tmp/hello.ll /tmp/hello_ic.ll
   ```

### 阶段四：深入后端 / 代码生成（2-4 周）

**目标**：理解 LLVM 如何将 IR 转换为目标机器码。

1. **阅读文档**
   - `llvm/docs/CodeGenerator.rst` — 代码生成器架构
   - `llvm/docs/WritingAnLLVMBackend.rst` — 编写后端指南
   - `llvm/docs/TableGen/` — TableGen 文档

2. **阅读源码**
   - `llvm/lib/CodeGen/` — 代码生成通用框架
   - `llvm/lib/CodeGen/SelectionDAG/` — SelectionDAG 指令选择
   - `llvm/lib/Target/X86/` 或 `llvm/lib/Target/RISCV/` — 具体后端实现
   - 后端的 `.td` 文件 — TableGen 定义（指令集、寄存器、调度模型）

3. **用 llc 观察代码生成过程**
   ```bash
   # 查看 SelectionDAG
   build/bin/llc -view-dag-combine1-dags /tmp/hello.ll

   # 查看指令选择后的结果
   build/bin/llc -print-after-all /tmp/hello.ll 2>&1 | less

   # 指定输出目标
   build/bin/llc -march=x86-64 -mcpu=help  # 查看支持的 CPU
   build/bin/llc -march=riscv64 /tmp/hello.ll -o /tmp/hello_riscv.s
   ```

### 阶段五：进阶方向（按兴趣选择）

| 方向 | 关键路径 | 说明 |
|------|----------|------|
| **Clang 前端** | `clang/lib/Lex/`, `Parse/`, `Sema/`, `CodeGen/` | 深入 C++ 编译前端 |
| **静态分析** | `clang/lib/StaticAnalyzer/`, `clang/lib/Analysis/` | Clang 静态分析器 |
| **Clang 工具开发** | `clang/lib/Tooling/`, `clang-tools-extra/` | 开发代码分析/重构工具 |
| **JIT 编译** | `llvm/lib/ExecutionEngine/`, `llvm/examples/BuildingAJIT/` | ORC JIT 框架 |
| **MLIR** | `mlir/` | 多层 IR，AI 编译器方向的热门选择 |
| **链接器** | `lld/` | 了解链接过程 |
| **Sanitizer** | `compiler-rt/lib/asan/`, `msan/`, `tsan/` | 内存安全工具的实现 |
| **新增后端** | `llvm/lib/Target/`, TableGen | 为新架构编写完整后端 |

---

## 8. 推荐阅读资源

### 仓库内文档（按推荐阅读顺序）

| 优先级 | 文档 | 说明 |
|--------|------|------|
| ★★★ | `llvm/docs/tutorial/LangImpl01-10.rst` | Kaleidoscope 教程，入门必读 |
| ★★★ | `llvm/docs/LangRef.rst` | IR 语言参考手册，随时查阅 |
| ★★★ | `llvm/docs/ProgrammersManual.rst` | LLVM C++ API 使用指南 |
| ★★☆ | `llvm/docs/WritingAnLLVMNewPMPass.rst` | 编写 Pass（新 Pass 管理器） |
| ★★☆ | `llvm/docs/GettingStarted.rst` | 入门与构建 |
| ★★☆ | `llvm/docs/CodingStandards.rst` | 编码规范（贡献代码前必读） |
| ★★☆ | `llvm/docs/CodeGenerator.rst` | 代码生成器架构 |
| ★☆☆ | `llvm/docs/WritingAnLLVMBackend.rst` | 编写后端（进阶） |
| ★☆☆ | `llvm/docs/AliasAnalysis.rst` | 别名分析（优化方向） |

### 仓库内示例代码（按推荐顺序）

| 优先级 | 路径 | 说明 |
|--------|------|------|
| ★★★ | `llvm/examples/Kaleidoscope/` | 教程配套的完整代码 |
| ★★★ | `llvm/lib/Transforms/Hello/` | 最简单的 Pass 示例 |
| ★★☆ | `llvm/examples/ModuleMaker/` | 用 API 构建 LLVM Module |
| ★★☆ | `llvm/examples/IRTransforms/` | IR 变换示例 |
| ★★☆ | `llvm/examples/HowToUseJIT/` | JIT 使用示例 |
| ★☆☆ | `llvm/examples/HowToUseLLJIT/` | LLJIT 使用示例 |

### 外部推荐资源

- [LLVM 官方文档](https://llvm.org/docs/) — 在线版文档
- [LLVM Doxygen API 文档](https://llvm.org/doxygen/) — C++ API 在线浏览
- [LLVM 官方博客](https://blog.llvm.org/) — 项目动态
- [LLVM Dev Meeting 演讲视频](https://www.youtube.com/c/LLVMPROJ) — 年度开发者会议录像

---

> **学习建议**：LLVM 代码量庞大，不要试图一次读完所有源码。建议以具体任务为导向（如"写一个 Pass"、"理解某个优化"），结合文档和工具（opt、llc）交叉学习。动手实验 > 阅读文档 > 阅读源码，循环迭代，逐步加深理解。
