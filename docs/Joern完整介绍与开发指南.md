# Joern完整介绍与开发指南

本文档旨在为撰写毕业论文提供对Joern项目的全面认识，涵盖工作原理、使用场景、CPG生成流程、图元素结构以及自定义开发方法。

---

## 1. Joern简介

### 1.1 什么是Joern？

**Joern是一个用于分析源代码、字节码和二进制可执行文件的开源代码分析平台**。它由Fabian Yamaguchi博士在2014年创建，目前由ShiftLeft公司和开源社区共同维护，是安全研究和漏洞发现领域的重要工具。

Joern的核心特性：
- **跨语言支持**：支持C/C++、Java、JavaScript、Python、Go、PHP、C#、Ruby、Swift等多种编程语言
- **代码属性图（CPG）**：将代码转换为图数据库表示，融合AST、CFG、DDG等多种图结构
- **高性能分析**：使用FlatGraph图数据库，相比传统方案内存占用减少40%，查询速度提升40%
- **丰富的查询语言**：基于Scala的领域特定语言（DSL），支持复杂的程序分析查询
- **可扩展架构**：支持自定义前端解析器、分析pass和查询插件

### 1.2 工作原理

Joern的工作原理可以概括为以下三个核心阶段：

#### 阶段1：代码解析（Parsing）
```
源代码 → 语言前端（Frontend） → 抽象语法树（AST）
```
- 针对不同编程语言，Joern提供了专门的前端解析器（如c2cpg、javasrc2cpg、pysrc2cpg等）
- 每个前端使用对应语言的解析器将源代码转换为AST
- 例如：C语言使用Eclipse CDT解析器，Java使用JavaParser

#### 阶段2：CPG生成（CPG Creation）
```
AST → CPG Pass管道 → 代码属性图（CPG）
```
- **AstCreationPass**：将AST转换为CPG的基础节点
- **Base层Pass**：添加基础结构（命名空间、类型、方法等）
- **ControlFlow层**：生成控制流图（CFG）
- **TypeRelations层**：建立类型继承关系
- **CallGraph层**：构建方法调用图
- 使用**DiffGraph机制**批量更新图结构，提高性能

#### 阶段3：分析查询（Analysis & Querying）
```
CPG → 查询引擎 → 分析结果
```
- 使用**Traversal DSL**进行图遍历查询
- 支持**数据流分析**（污点追踪、可达性分析）
- 内置**QueryDB**查询库，包含50+常见漏洞检测查询

### 1.3 使用场景

Joern在以下场景中被广泛应用：

#### 1. 安全漏洞发现
- **静态应用安全测试（SAST）**：检测缓冲区溢出、注入漏洞、使用后释放等
- **污点分析**：追踪不可信数据从输入到危险函数的流动路径
- **密码学问题**：发现弱加密算法、硬编码密钥等安全问题

#### 2. 程序理解与逆向工程
- **代码审计**：快速理解大型代码库的结构和逻辑
- **调用关系分析**：可视化函数调用链和依赖关系
- **二进制分析**：通过ghidra2cpg支持对编译后程序的分析

#### 3. 代码质量分析
- **代码度量**：计算圈复杂度、函数参数过多、过长函数等
- **代码异味检测**：识别不良编程实践
- **依赖分析**：检测未使用的导入、循环依赖等

#### 4. 学术研究
- **程序分析研究**：作为研究平台开发新的分析算法
- **机器学习**：将CPG作为特征输入进行漏洞预测
- **软件演化研究**：对比不同版本代码的变化

---

## 2. Joern解析源码并生成CPG图的基本流程

### 2.1 整体架构

Joern采用**多层Pass管道架构**，将CPG生成过程分解为多个独立的转换步骤：

```
源代码文件
    ↓
┌─────────────────────────────────────────┐
│  语言前端（Language Frontend）          │
│  - c2cpg, javasrc2cpg, pysrc2cpg等      │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  AST创建Pass（AstCreationPass）         │
│  - 将源代码解析为抽象语法树              │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  Base层Pass（基础结构层）               │
│  - FileCreationPass                     │
│  - NamespaceCreator                     │
│  - TypeDeclStubCreator                  │
│  - MethodStubCreator                    │
│  - AstLinkerPass                        │
│  - TypeRefPass, TypeEvalPass            │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  可选增强层（Optional Overlays）         │
│  - ControlFlow: 控制流图生成             │
│  - CallGraph: 调用图生成                 │
│  - TypeRelations: 类型层次关系           │
│  - DataFlow: 数据流分析                  │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  FlatGraph图数据库                      │
│  - 持久化为cpg.bin或cpg.bin.zip         │
└─────────────────────────────────────────┘
    ↓
   CPG
```

### 2.2 详细流程说明

#### 步骤1：选择语言前端

根据待分析代码的语言，选择相应的前端工具：

```bash
# C/C++代码
./c2cpg.sh <source_dir> -o cpg.bin

# Java源代码
./javasrc2cpg <source_dir> -o cpg.bin

# Python代码
./pysrc2cpg <source_dir> -o cpg.bin

# JavaScript/TypeScript
./jssrc2cpg <source_dir> -o cpg.bin
```

#### 步骤2：语法解析（Parsing）

每个前端内部会：
1. 扫描输入目录，识别源代码文件
2. 调用语言特定的解析器（如ANTLR、JavaParser、Tree-sitter等）
3. 生成抽象语法树（AST）
4. 处理预处理指令、宏展开（C/C++特有）

#### 步骤3：AST转CPG（AstCreationPass）

这是最关键的一步，将语言特定的AST转换为Joern统一的CPG表示：

```scala
// 示例：创建方法节点
val methodNode = NewMethod()
  .name("myFunction")
  .fullName("com.example.MyClass.myFunction")
  .signature("int myFunction(String param)")
  .lineNumber(42)
  .columnNumber(5)
  .code("int myFunction(String param) { ... }")

// 创建参数节点
val paramNode = NewMethodParameterIn()
  .name("param")
  .typeFullName("java.lang.String")
  .order(1)

// 创建调用节点
val callNode = NewCall()
  .name("println")
  .methodFullName("java.io.PrintStream.println")
  .code("System.out.println(param)")

// 批量添加到图中
diffGraph.addNode(methodNode)
diffGraph.addNode(paramNode)
diffGraph.addNode(callNode)
diffGraph.addEdge(methodNode, paramNode, EdgeTypes.AST)
```

#### 步骤4：Base层处理

Base层包含一系列跨语言通用的Pass：

| Pass名称 | 功能描述 |
|---------|---------|
| **FileCreationPass** | 为每个源文件创建FILE节点 |
| **NamespaceCreator** | 创建命名空间/包结构 |
| **TypeDeclStubCreator** | 为类/接口创建类型声明节点 |
| **MethodStubCreator** | 创建方法声明的存根节点 |
| **AstLinkerPass** | 链接AST父子关系，确保树结构完整 |
| **ContainsEdgePass** | 添加CONTAINS边（文件包含类型，类型包含方法） |
| **TypeRefPass** | 解析类型引用（如变量类型、返回类型） |
| **TypeEvalPass** | 类型推断，为表达式节点添加类型信息 |

#### 步骤5：增强层（Overlays）

根据分析需求，可以选择性地应用增强层：

**ControlFlow层（控制流图）**
```scala
// 为每个方法生成CFG
cpg.method.foreach { method =>
  val cfg = new CfgCreationPass(method).createCfg()
  // CFG边连接基本块：if条件、循环、跳转等
}
```

**CallGraph层（调用图）**
```scala
// 解析方法调用，建立CALL节点到METHOD节点的链接
cpg.call.methodFullName("com.example.*").foreach { call =>
  val targetMethod = resolveCallTarget(call)
  diffGraph.addEdge(call, targetMethod, EdgeTypes.CALL)
}
```

**TypeRelations层（类型关系）**
```scala
// 建立继承关系：Subclass -> INHERITS_FROM -> Superclass
cpg.typeDecl.filter(_.inheritsFromTypeFullName.nonEmpty).foreach { typeDecl =>
  val baseTypes = resolveBaseTypes(typeDecl)
  baseTypes.foreach { baseType =>
    diffGraph.addEdge(typeDecl, baseType, EdgeTypes.INHERITS_FROM)
  }
}
```

#### 步骤6：持久化

生成的CPG通过FlatGraph序列化为二进制格式：

```scala
// 保存CPG到磁盘
cpg.close()  // 写入cpg.bin

// 之后可以重新加载
val cpg = io.shiftleft.codepropertygraph.Cpg.withStorage("cpg.bin")
```

### 2.3 关键技术点

#### DiffGraph机制
为了提高性能，Joern使用批量更新机制：
```scala
val diffGraph = DiffGraphBuilder()
// 批量添加节点和边
diffGraph.addNode(node1)
diffGraph.addNode(node2)
diffGraph.addEdge(node1, node2, EdgeTypes.AST)
// 一次性应用所有变更
DiffGraphApplier.applyDiff(cpg.graph, diffGraph)
```

这避免了频繁的单点更新，提升了CPG构建速度。

#### 语言无关设计
尽管不同语言的语法差异很大，Joern通过以下方式实现统一表示：
- **共享节点类型**：METHOD、CALL、IDENTIFIER等对所有语言通用
- **语言特定属性**：通过dynamicTypeHintFullName、astParentType等扩展属性处理特殊情况
- **语义归一化**：例如将Python的list comprehension和Java的for循环都表示为CONTROL_STRUCTURE节点

---

## 3. 生成的CPG图包含的元素

### 3.1 CPG核心概念

代码属性图（Code Property Graph）是一种**融合多种程序表示的超图结构**，它将以下几种图统一在一起：
- **抽象语法树（AST）**：代码的语法结构
- **控制流图（CFG）**：程序执行路径
- **程序依赖图（PDG）**：数据和控制依赖关系
- **调用图（Call Graph）**：函数调用关系
- **类型层次图（Type Hierarchy）**：继承和实现关系

### 3.2 节点类型（Node Types）

CPG定义了50+种节点类型，主要分为以下几类：

#### 3.2.1 结构性节点（Structural Nodes）

| 节点类型 | 说明 | 示例 |
|---------|------|------|
| **FILE** | 源代码文件 | `src/main/java/App.java` |
| **NAMESPACE_BLOCK** | 命名空间/包的实现块 | `package com.example` |
| **NAMESPACE** | 命名空间引用 | `com.example` |
| **TYPE_DECL** | 类、接口、结构体声明 | `class User { ... }` |
| **TYPE** | 类型定义 | `java.lang.String` |
| **METHOD** | 方法/函数声明 | `void login(String user) { ... }` |
| **METHOD_RETURN** | 方法返回值 | `return value;` |
| **MEMBER** | 类成员变量 | `private String name;` |
| **MODIFIER** | 访问修饰符 | `public`, `static`, `final` |

#### 3.2.2 语句和表达式节点（Statement & Expression Nodes）

| 节点类型 | 说明 | 示例 |
|---------|------|------|
| **BLOCK** | 代码块 | `{ stmt1; stmt2; }` |
| **CONTROL_STRUCTURE** | 控制结构 | `if`, `for`, `while`, `switch` |
| **CALL** | 函数/方法调用 | `println("Hello")` |
| **RETURN** | 返回语句 | `return x + y;` |
| **IDENTIFIER** | 标识符引用 | `username` |
| **LITERAL** | 字面量 | `"string"`, `42`, `true` |
| **FIELD_IDENTIFIER** | 字段访问 | `obj.field` |
| **UNKNOWN** | 未识别的表达式 | 复杂的宏展开结果 |

#### 3.2.3 参数和局部变量节点

| 节点类型 | 说明 | 示例 |
|---------|------|------|
| **METHOD_PARAMETER_IN** | 输入参数 | `void func(int x)` 中的 `x` |
| **METHOD_PARAMETER_OUT** | 输出参数（仅C/C++） | 指针或引用参数 |
| **LOCAL** | 局部变量声明 | `int count = 0;` |

#### 3.2.4 类型和注解节点

| 节点类型 | 说明 | 示例 |
|---------|------|------|
| **TYPE_PARAMETER** | 泛型参数 | `<T>` in `List<T>` |
| **TYPE_ARGUMENT** | 泛型实参 | `String` in `List<String>` |
| **ANNOTATION** | 注解/装饰器 | `@Override`, `@Deprecated` |
| **ANNOTATION_PARAMETER** | 注解参数 | `@RequestMapping(value="/api")` |

#### 3.2.5 其他节点

| 节点类型 | 说明 | 示例 |
|---------|------|------|
| **COMMENT** | 注释 | `// This is a comment` |
| **IMPORT** | 导入语句 | `import java.util.*` |
| **BINDING** | 变量绑定 | 闭包捕获的变量 |
| **META_DATA** | 元数据 | CPG版本、语言信息 |

### 3.3 边类型（Edge Types）

边用于连接节点，表示不同类型的关系：

#### 3.3.1 结构性边（Structural Edges）

| 边类型 | 说明 | 示例 |
|-------|------|------|
| **AST** | 抽象语法树父子关系 | METHOD --AST--> BLOCK --AST--> CALL |
| **SOURCE_FILE** | 节点属于哪个源文件 | METHOD --SOURCE_FILE--> FILE |
| **CONTAINS** | 包含关系 | FILE --CONTAINS--> TYPE_DECL --CONTAINS--> METHOD |

#### 3.3.2 控制流边（Control Flow Edges）

| 边类型 | 说明 | 示例 |
|-------|------|------|
| **CFG** | 控制流图边 | Statement1 --CFG--> Statement2 |
| **CONDITION** | 条件表达式 | IF --CONDITION--> Expression |
| **DOMINATES** | 支配关系 | BasicBlock1 --DOMINATES--> BasicBlock2 |
| **POST_DOMINATES** | 后支配关系 | Exit --POST_DOMINATES--> Block |

#### 3.3.3 调用和引用边（Call & Reference Edges）

| 边类型 | 说明 | 示例 |
|-------|------|------|
| **CALL** | 调用边 | CALL --CALL--> METHOD |
| **ARGUMENT** | 参数传递 | CALL --ARGUMENT--> Expression |
| **RECEIVER** | 方法接收者 | CALL --RECEIVER--> Object |
| **REF** | 类型引用 | LOCAL --REF--> TYPE |
| **EVAL_TYPE** | 表达式类型 | Expression --EVAL_TYPE--> TYPE |

#### 3.3.4 数据流边（Data Flow Edges）

| 边类型 | 说明 | 示例 |
|-------|------|------|
| **REACHING_DEF** | 到达定义 | Definition --REACHING_DEF--> Use |
| **PARAMETER_LINK** | 参数链接 | PARAM_IN --PARAMETER_LINK--> PARAM_OUT |

#### 3.3.5 类型关系边（Type Relation Edges）

| 边类型 | 说明 | 示例 |
|-------|------|------|
| **INHERITS_FROM** | 继承关系 | SubClass --INHERITS_FROM--> SuperClass |
| **ALIAS_OF** | 类型别名 | TypeAlias --ALIAS_OF--> OriginalType |
| **BINDS_TO** | 绑定到变量 | Binding --BINDS_TO--> LOCAL |

### 3.4 节点属性（Node Properties）

每个节点都有一组属性，常见属性包括：

| 属性名 | 类型 | 说明 | 示例 |
|-------|------|------|------|
| **name** | String | 节点名称 | `"myFunction"` |
| **fullName** | String | 完全限定名 | `"com.example.MyClass.myFunction"` |
| **code** | String | 源代码文本 | `"int x = 42;"` |
| **lineNumber** | Int | 行号 | `42` |
| **columnNumber** | Int | 列号 | `10` |
| **order** | Int | AST子节点顺序 | `0`, `1`, `2` |
| **typeFullName** | String | 类型全名 | `"java.lang.String"` |
| **signature** | String | 方法签名 | `"void func(int, String)"` |
| **isExternal** | Boolean | 是否外部定义 | `true`/`false` |
| **filename** | String | 文件路径 | `"src/App.java"` |
| **astParentType** | String | 父节点类型 | `"METHOD"` |
| **astParentFullName** | String | 父节点完整名称 | `"com.example.MyClass.method"` |

### 3.5 CPG示例

以下C代码：
```c
int add(int a, int b) {
    int result = a + b;
    return result;
}
```

对应的CPG结构（简化表示）：
```
FILE(name="example.c")
  |--CONTAINS-->
  |
  METHOD(name="add", fullName="add", signature="int(int,int)")
    |--AST-->
    |
    +-- METHOD_PARAMETER_IN(name="a", typeFullName="int", order=1)
    |
    +-- METHOD_PARAMETER_IN(name="b", typeFullName="int", order=2)
    |
    +-- METHOD_RETURN(typeFullName="int")
    |
    +-- BLOCK(code="{ int result = a + b; return result; }")
          |--AST-->
          |
          +-- LOCAL(name="result", typeFullName="int")
          |
          +-- CALL(name="<operator>.assignment", code="result = a + b")
          |     |--ARGUMENT-->
          |     |
          |     +-- IDENTIFIER(name="result", order=1)
          |     |
          |     +-- CALL(name="<operator>.addition", code="a + b", order=2)
          |           |--ARGUMENT-->
          |           |
          |           +-- IDENTIFIER(name="a", order=1)
          |           |
          |           +-- IDENTIFIER(name="b", order=2)
          |
          +-- RETURN(code="return result;")
                |--AST-->
                |
                +-- IDENTIFIER(name="result")
```

### 3.6 CPG的价值

通过将代码表示为图结构，Joern能够：
1. **统一表示**：不同语言的代码转换为相同的图模式
2. **关系推理**：轻松查询"哪些函数调用了X"、"变量Y在哪里被修改"
3. **路径分析**：查找从输入到危险函数的完整数据流路径
4. **模式匹配**：使用图查询语言快速发现代码模式和反模式
5. **可扩展性**：可以添加自定义节点、边和属性来支持特定分析需求

---

## 4. 如何对Joern进行自定义开发

### 4.1 开发环境搭建

#### 4.1.1 系统要求
- **JDK 21**（推荐使用OpenJDK或Oracle JDK）
- **SBT**（Scala Build Tool）
- **Git**
- **可选**：gcc/g++（用于C/C++头文件自动发现）
- **可选**：IDE（IntelliJ IDEA + Scala插件 或 VSCode + Metals插件）

#### 4.1.2 克隆和构建
```bash
# 克隆Joern仓库
git clone https://github.com/joernio/joern.git
cd joern

# 编译Joern
sbt stage

# 运行Joern
./joern
```

#### 4.1.3 IDE设置

**IntelliJ IDEA：**
1. 安装Scala插件
2. 在SBT中运行`sbt compile`并保持运行
3. 在IntelliJ中选择"Open as BSP project"（不要选择sbt项目）
4. 等待索引完成

**VSCode：**
1. 安装Docker和`ms-vscode-remote.remote-containers`插件
2. 打开Joern项目文件夹
3. 选择"Reopen in Container"
4. 在Metals侧边栏选择"Import build"

### 4.2 创建自定义语言前端

#### 4.2.1 前端结构

创建一个新的语言前端需要实现以下组件：

```
mylang2cpg/
├── src/main/scala/io/joern/mylang2cpg/
│   ├── Main.scala                    # CLI入口
│   ├── Config.scala                  # 配置选项
│   ├── MyLang2Cpg.scala             # 前端主类
│   ├── parser/
│   │   └── MyLangParser.scala       # 语言解析器封装
│   ├── astcreation/
│   │   └── AstCreator.scala         # AST转CPG转换器
│   └── passes/
│       └── MyLangTypePass.scala     # 语言特定的Pass
└── src/test/scala/io/joern/mylang2cpg/
    └── MyLangPassTests.scala         # 测试
```

#### 4.2.2 实现配置类

```scala
package io.joern.mylang2cpg

import io.joern.x2cpg.X2CpgConfig

case class Config(
  inputPath: String = "",
  outputPath: String = "cpg.bin"
) extends X2CpgConfig[Config] {
  // 自定义选项
  var includeTests: Boolean = false
  var maxDepth: Int = 100
}
```

#### 4.2.3 实现主类

```scala
package io.joern.mylang2cpg

import io.joern.x2cpg.X2CpgFrontend
import io.joern.x2cpg.passes.frontend.XTypeRecoveryConfig
import io.shiftleft.codepropertygraph.Cpg

class MyLang2Cpg extends X2CpgFrontend[Config] {
  
  override def createCpg(config: Config): Try[Cpg] = {
    // 1. 初始化CPG
    val cpg = Cpg.empty
    
    // 2. 执行解析和AST创建
    val astCreator = new AstCreator(config, cpg)
    astCreator.createAst()
    
    // 3. 运行Base层Pass
    new Base().run(new LayerCreatorContext(cpg))
    
    // 4. 返回CPG
    Success(cpg)
  }
}
```

#### 4.2.4 实现AST创建器

```scala
package io.joern.mylang2cpg.astcreation

import io.joern.x2cpg.Ast
import io.joern.x2cpg.AstCreatorBase
import io.shiftleft.codepropertygraph.generated.nodes._

class AstCreator(config: Config, cpg: Cpg) 
  extends AstCreatorBase(config.inputPath) {
  
  def createAst(): Unit = {
    // 1. 解析源文件
    val sourceFiles = scanInputPath(config.inputPath)
    
    // 2. 为每个文件创建AST
    sourceFiles.foreach { file =>
      val fileNode = createFileNode(file)
      diffGraph.addNode(fileNode)
      
      // 3. 解析文件内容
      val parsedAst = parseFile(file)
      
      // 4. 转换为CPG AST
      val cpgAst = convertToCpgAst(parsedAst, fileNode)
      
      // 5. 添加到图中
      Ast.storeInDiffGraph(cpgAst, diffGraph)
    }
  }
  
  private def convertToCpgAst(node: MyLangNode, parentFile: NewFile): Ast = {
    node match {
      case FunctionDef(name, params, body) =>
        val methodNode = NewMethod()
          .name(name)
          .fullName(s"${parentFile.name}:$name")
          .signature(createSignature(params))
          .lineNumber(node.lineNumber)
          .code(node.sourceText)
        
        val paramAsts = params.map(convertParam)
        val bodyAst = convertBody(body)
        
        methodAst(methodNode, paramAsts, bodyAst)
      
      case CallExpr(funcName, args) =>
        val callNode = NewCall()
          .name(funcName)
          .methodFullName(funcName)
          .code(node.sourceText)
        
        val argAsts = args.map(convertToCpgAst)
        callAst(callNode, argAsts)
      
      // ... 处理其他节点类型
    }
  }
}
```

### 4.3 添加自定义CPG Pass

#### 4.3.1 Pass基类

所有Pass都继承自`CpgPass`：

```scala
package io.joern.x2cpg.passes

import io.shiftleft.codepropertygraph.Cpg
import io.shiftleft.passes.CpgPass
import io.shiftleft.codepropertygraph.generated.nodes._

class MyCustomPass(cpg: Cpg) extends CpgPass(cpg) {
  
  override def run(builder: DiffGraphBuilder): Unit = {
    // 遍历CPG并应用转换
    cpg.method.foreach { method =>
      // 分析逻辑
      if (needsModification(method)) {
        addCustomAnnotation(method, builder)
      }
    }
  }
  
  private def needsModification(method: Method): Boolean = {
    // 自定义检查逻辑
    method.parameter.size > 5
  }
  
  private def addCustomAnnotation(
    method: Method, 
    builder: DiffGraphBuilder
  ): Unit = {
    val annotation = NewAnnotation()
      .name("TooManyParameters")
      .fullName("custom.TooManyParameters")
      .code(s"@TooManyParameters(${method.parameter.size})")
    
    builder.addNode(annotation)
    builder.addEdge(method, annotation, EdgeTypes.AST)
  }
}
```

### 4.4 编写自定义查询

#### 4.4.1 查询库结构

查询放在`querydb/src/main/scala/io/joern/scanners/`：

```scala
package io.joern.scanners.mylang

import io.joern.scanners._
import io.joern.console.scripting.Query
import io.shiftleft.semanticcpg.language._

object MyLangQueries extends QueryBundle {
  
  @q
  def findHardcodedPasswords(): Query = Query.make(
    name = "hardcoded-passwords",
    author = Crew.yourName,
    title = "Hardcoded passwords in source code",
    description = """
      Detects string literals that appear to be passwords
      assigned to variables or passed to authentication functions.
    """,
    score = 8.0,
    withStrRep({ cpg =>
      cpg.assignment
        .where(_.target.isIdentifier)
        .where(_.target.name(".*(?i)(password|passwd|pwd).*"))
        .where(_.source.isLiteral)
        .filter { assignment =>
          val value = assignment.source.code.head
          value.length > 5 && !value.contains("$")
        }
    }),
    tags = List(QueryTags.badfn, QueryTags.default),
    codeExamples = CodeExamples(
      List("""
        String password = "admin123";  // BAD
        auth.login(username, "secret");  // BAD
      """),
      List("""
        String password = System.getenv("DB_PASSWORD");  // GOOD
        auth.login(username, getUserInput());  // GOOD
      """)
    )
  )
  
  @q
  def findSqlInjection(): Query = Query.make(
    name = "sql-injection",
    author = Crew.yourName,
    title = "Potential SQL injection vulnerability",
    description = "Detects SQL queries constructed with user input",
    score = 9.0,
    withStrRep({ cpg =>
      cpg.method.name(".*execute.*", ".*query.*").call
        .whereNot(_.argument.isLiteral)
        .where(_.argument.reachableBy(cpg.method.isPublic.parameter))
    }),
    tags = List(QueryTags.badfn, QueryTags.default)
  )
}
```

#### 4.4.2 高级查询示例

**数据流分析查询：**
```scala
@q
def findPathTraversal(): Query = Query.make(
  name = "path-traversal",
  author = Crew.yourName,
  title = "Path traversal vulnerability",
  description = "User input flows to file system operations",
  score = 8.5,
  withStrRep({ cpg =>
    val sources = cpg.method.name("getParameter", "readLine").callIn
    val sinks = cpg.method.name(".*File.*", ".*Path.*").callIn
    
    sinks.filter { sink =>
      sink.argument.reachableBy(sources).nonEmpty
    }
  }),
  tags = List(QueryTags.badfn)
)
```

**控制流分析查询：**
```scala
@q
def findMissingNullChecks(): Query = Query.make(
  name = "missing-null-check",
  author = Crew.yourName,
  title = "Potential null pointer dereference",
  description = "Variables used without null checks",
  score = 6.0,
  withStrRep({ cpg =>
    cpg.method.internal.flatMap { method =>
      method.call.name(".*get.*", ".*find.*")
        .whereNot(_.cfgNext.isCall.name(".*null.*", ".*isEmpty.*"))
        .filter { call =>
          // 检查返回值是否直接被解引用
          val nextUse = call.cfgNext.isCall
            .where(_.argument.isIdentifier.code(call.code))
          nextUse.nonEmpty
        }
    }
  }),
  tags = List(QueryTags.badfn)
)
```

#### 4.4.3 测试查询

```scala
package io.joern.scanners.mylang

import io.joern.suites.Suite

class MyLangQueryTests extends Suite {
  
  override val code = """
    void processRequest(String userInput) {
      String query = "SELECT * FROM users WHERE id = " + userInput;
      db.execute(query);  // 应该被检测到
    }
  """
  
  "findSqlInjection" should {
    "detect SQL injection vulnerability" in {
      val results = MyLangQueries.findSqlInjection()(cpg)
      results should have size 1
      results.head.evidence.head.asInstanceOf[Call].name shouldBe "execute"
    }
  }
  
  "not flag safe queries" in {
    val safeCode = """
      void processRequest(int userId) {
        String query = "SELECT * FROM users WHERE id = ?";
        db.execute(query, userId);  // 参数化查询，安全
      }
    """
    cpg = code2Cpg(safeCode)
    MyLangQueries.findSqlInjection()(cpg) shouldBe empty
  }
}
```

#### 4.4.4 运行查询

```bash
# 方式1：使用joern-scan命令行工具
./joern-scan <source_dir> --list-query-names  # 列出所有查询
./joern-scan <source_dir> --query sql-injection  # 运行特定查询

# 方式2：在Joern REPL中
./joern
joern> importCpg("cpg.bin")
joern> runQuery("sql-injection")
joern> cpg.method.name("execute").call.l  # 手动查询
```

### 4.5 扩展CPG Schema

#### 4.5.1 Schema扩展概述

如果需要添加新的节点类型或属性，可以使用Schema Extender：

```scala
// 位置：schema-extender/src/main/scala/MySchemaExtension.scala
import flatgraph.schema._

object MySchemaExtension {
  
  def addCustomNodes(builder: SchemaBuilder): Unit = {
    // 添加自定义属性
    val securityLevel = builder.addProperty(
      name = "SECURITY_LEVEL",
      valueType = ValueTypes.STRING,
      comment = "Security classification of the code element"
    )
    
    val riskScore = builder.addProperty(
      name = "RISK_SCORE",
      valueType = ValueTypes.INT,
      comment = "Calculated risk score (0-10)"
    )
    
    // 添加自定义节点类型
    val securityAnnotation = builder.addNodeType(
      name = "SECURITY_ANNOTATION",
      comment = "Custom security annotation node"
    )
      .addProperty(securityLevel)
      .addProperty(riskScore)
    
    // 扩展现有节点
    builder.getNodeType("METHOD")
      .addProperty(securityLevel)
      .addProperty(riskScore)
    
    // 添加自定义边类型
    val hasSecurityIssue = builder.addEdgeType(
      name = "HAS_SECURITY_ISSUE",
      comment = "Links code elements to security issues"
    )
      .addSourceNode(builder.getNodeType("METHOD"))
      .addDestinationNode(securityAnnotation)
  }
}
```

### 4.6 开发最佳实践

#### 4.6.1 代码风格
```bash
# 在提交前格式化代码
sbt scalafmt Test/scalafmt
```

#### 4.6.2 测试驱动开发
- 为每个Pass编写单元测试
- 测试边界情况和错误输入
- 使用`EmptyGraphFixture`和`MockCpg`简化测试

#### 4.6.3 性能优化
- 使用`DiffGraphBuilder`批量更新图
- 避免在循环中频繁查询CPG
- 考虑Pass的执行顺序，避免重复遍历

#### 4.6.4 文档
- 为自定义查询提供清晰的描述和代码示例
- 注释复杂的AST转换逻辑
- 更新README记录新功能

#### 4.6.5 贡献回社区
- 遵循贡献指南
- 提供有意义的PR描述
- 包含测试和文档更新

### 4.7 常用SBT命令

```bash
# 构建相关
sbt compile                          # 编译项目
sbt stage                            # 构建可执行文件
sbt <project>/stage                  # 构建特定项目
sbt createDistribution               # 创建完整发布包

# 测试相关
sbt test                             # 运行所有测试
sbt <project>/test                   # 测试特定项目
sbt testOnly *MyTest                 # 运行特定测试类

# 代码质量
sbt scalafmt Test/scalafmt          # 格式化代码

# 交互式开发
sbt console                          # 启动Scala REPL
sbt ~compile                         # 监视文件变化自动编译
```

---

## 5. 实战案例：检测不安全的反序列化

### 5.1 问题背景

Java反序列化漏洞是一种严重的安全问题，攻击者可以通过构造恶意序列化对象执行任意代码。我们要创建一个Joern查询来检测不安全的反序列化操作。

### 5.2 实现查询

```scala
package io.joern.scanners.java

import io.joern.scanners._
import io.joern.console.scripting.Query
import io.shiftleft.semanticcpg.language._

object JavaSecurityQueries extends QueryBundle {
  
  @q
  def unsafeDeserialization(): Query = Query.make(
    name = "unsafe-deserialization",
    author = Crew.yourName,
    title = "Unsafe Java Deserialization",
    description = """
      Detects deserialization of untrusted data which can lead to
      remote code execution. The query finds:
      1. Calls to ObjectInputStream.readObject()
      2. That take input from untrusted sources (HTTP requests, sockets, etc.)
      3. Without prior validation or whitelisting
    """,
    score = 9.5,
    withStrRep({ cpg =>
      // 定义source：不可信数据源
      val untrustedSources = cpg.call
        .methodFullName(".*HttpServletRequest.*getInputStream")
        .methodFullName(".*Socket.*getInputStream")
        .methodFullName(".*readLine.*")
      
      // 定义sink：反序列化操作
      val deserializationSinks = cpg.call
        .methodFullName(".*ObjectInputStream.*readObject")
      
      // 查找从source到sink的数据流
      deserializationSinks.where { sink =>
        sink.argument.reachableBy(untrustedSources).nonEmpty
      }
    }),
    tags = List(QueryTags.badfn, QueryTags.default),
    codeExamples = CodeExamples(
      List("""
        // BAD: 直接反序列化HTTP输入
        protected void doPost(HttpServletRequest request) {
          InputStream is = request.getInputStream();
          ObjectInputStream ois = new ObjectInputStream(is);
          Object obj = ois.readObject();  // 漏洞！
        }
      """),
      List("""
        // GOOD: 使用白名单验证
        protected void doPost(HttpServletRequest request) {
          InputStream is = request.getInputStream();
          ObjectInputStream ois = new ValidatingObjectInputStream(is);
          ois.setWhitelist(Arrays.asList("com.example.SafeClass"));
          Object obj = ois.readObject();  // 安全
        }
      """)
    )
  )
}
```

### 5.3 测试查询

```scala
class JavaSecurityQueryTests extends Suite {
  
  val vulnerableCode = """
    import javax.servlet.http.*;
    import java.io.*;
    
    public class VulnerableServlet extends HttpServlet {
      protected void doPost(HttpServletRequest request) {
        try {
          InputStream is = request.getInputStream();
          ObjectInputStream ois = new ObjectInputStream(is);
          UserData userData = (UserData) ois.readObject();
          processData(userData);
        } catch (Exception e) {
          e.printStackTrace();
        }
      }
    }
  """
  
  "unsafeDeserialization" should {
    "detect vulnerable deserialization" in {
      cpg = code2Cpg(vulnerableCode, "VulnerableServlet.java")
      val results = JavaSecurityQueries.unsafeDeserialization()(cpg)
      
      results should have size 1
      results.head.evidence.head match {
        case call: Call =>
          call.methodFullName should include("readObject")
        case _ => fail("Expected Call node")
      }
    }
  }
}
```

---

## 6. 学习资源和参考文献

### 6.1 官方文档
- **Joern官网**：https://joern.io
- **完整文档**：https://docs.joern.io/
- **CPG规范**：https://cpg.joern.io
- **GitHub仓库**：https://github.com/joernio/joern

### 6.2 学术论文

1. Yamaguchi, F., Golde, N., Arp, D., & Rieck, K. (2014). **"Modeling and Discovering Vulnerabilities with Code Property Graphs"**. IEEE Symposium on Security and Privacy. 
   - 这篇论文是CPG的开创性工作，介绍了如何将代码表示为属性图并用于漏洞发现。

2. Yamaguchi, F., Wressnegger, C., Gascon, H., & Rieck, K. (2013). **"Chucky: Exposing Missing Checks in Source Code for Vulnerability Discovery"**. ACM CCS.
   - 提出了一种基于机器学习的方法，通过分析CPG来发现缺失的安全检查。

3. Yamaguchi, F., Maier, A., Gascon, H., & Rieck, K. (2015). **"Automatic Inference of Search Patterns for Taint-Style Vulnerabilities"**. IEEE S&P.
   - 展示了如何自动学习污点分析模式来发现安全漏洞。

4. Yamaguchi, F., Lottmann, M., & Rieck, K. (2012). **"Generalized Vulnerability Extrapolation using Abstract Syntax Trees"**. ACSAC.
   - 介绍了使用AST进行漏洞泛化的技术。

### 6.3 相关技术

#### 静态程序分析
- **数据流分析**：追踪变量值在程序中的传播
- **控制流分析**：分析程序的执行路径
- **指针分析**：确定指针可能指向的内存位置
- **类型推断**：自动推导变量和表达式的类型

#### 图数据库技术
- **Neo4j**：流行的图数据库，Joern早期版本使用
- **TinkerPop**：图计算框架
- **FlatGraph**：Joern v4.0+使用的高性能图存储

#### 代码分析工具
- **LLVM**：编译器基础设施，提供丰富的分析能力
- **Soot**：Java字节码分析框架
- **WALA**：IBM的程序分析库
- **CodeQL**：GitHub的代码查询语言

#### 机器学习在代码分析中的应用
- **Graph Neural Networks (GNN)**：在代码图上进行深度学习
- **Code2Vec**：将代码片段嵌入到向量空间
- **Transformer模型**：用于代码理解和生成

### 6.4 社区资源
- **Discord频道**：https://discord.com/invite/vv4MH284Hc
- **博客文章**：ShiftLeft博客关于Joern的系列文章
- **视频教程**：YouTube上的Joern workshops和演示
- **Stack Overflow**：标签[joern]下的问题和答案

### 6.5 实用工具和库
- **sbt**：Scala构建工具
- **scalafmt**：Scala代码格式化工具
- **ScalaTest**：Scala测试框架
- **ANTLR**：强大的解析器生成器
- **Tree-sitter**：增量解析库

---

## 7. 总结

### 7.1 核心要点回顾

本文档详细介绍了Joern代码分析平台的方方面面：

1. **Joern的本质**：一个跨语言的代码分析平台，通过代码属性图（CPG）统一表示不同语言的代码，为静态分析提供强大基础。

2. **工作流程**：从源代码到CPG的完整流程包括：
   - 语言前端解析（c2cpg、javasrc2cpg等）
   - AST创建和转换
   - 多层Pass管道（Base、ControlFlow、CallGraph等）
   - FlatGraph持久化存储

3. **CPG结构**：
   - 50+种节点类型（METHOD、CALL、IDENTIFIER等）
   - 丰富的边类型（AST、CFG、CALL、REF等）
   - 详细的节点属性（name、fullName、code、typeFullName等）
   - 融合AST、CFG、PDG、调用图的超图结构

4. **自定义开发**：
   - 创建新语言前端的完整流程
   - 编写自定义Pass扩展分析能力
   - 开发安全查询检测漏洞
   - 扩展CPG schema添加自定义节点和属性

### 7.2 Joern的优势

1. **跨语言统一**：支持10+种编程语言，提供统一的分析接口
2. **高性能**：FlatGraph架构，内存效率提升40%，查询速度提升40%
3. **可扩展性**：模块化设计，易于添加新语言和分析功能
4. **强大的查询能力**：基于Scala的DSL，支持复杂的图遍历和数据流分析
5. **开源社区**：活跃的社区支持，持续更新和改进

### 7.3 应用前景

Joern在以下领域有广阔的应用前景：

1. **软件安全**：自动化漏洞发现、安全审计、合规性检查
2. **DevSecOps**：集成到CI/CD流水线，实现持续安全检测
3. **代码理解**：帮助开发者快速理解大型代码库
4. **学术研究**：作为研究平台探索新的程序分析技术
5. **智能编程助手**：结合机器学习，提供智能代码建议和bug修复

### 7.4 毕业论文建议

如果您在撰写关于Joern的毕业论文，建议关注以下方向：

1. **理论研究**：
   - CPG的形式化定义和理论基础
   - 图查询语言的表达能力分析
   - 数据流分析算法的正确性和完整性证明

2. **技术改进**：
   - 提出新的Pass或分析算法
   - 优化CPG构建和查询性能
   - 扩展对新编程语言的支持

3. **应用研究**：
   - 使用Joern进行大规模代码库的漏洞研究
   - 开发针对特定漏洞类型的检测查询
   - 比较Joern与其他静态分析工具的效果

4. **跨学科结合**：
   - 将机器学习应用于CPG分析
   - 结合符号执行和CPG进行混合分析
   - 探索CPG在软件演化研究中的应用

### 7.5 进一步学习路径

1. **入门阶段**（1-2周）：
   - 安装Joern并运行基本示例
   - 学习Scala基础语法
   - 理解CPG的基本概念

2. **实践阶段**（2-4周）：
   - 分析真实项目的代码
   - 编写简单的自定义查询
   - 研究现有查询的实现

3. **进阶阶段**（1-2个月）：
   - 实现自定义Pass
   - 为新语言开发前端
   - 贡献代码回社区

4. **专家阶段**（持续）：
   - 研究高级分析技术
   - 发表学术论文
   - 参与Joern核心开发

---

## 附录

### A. 常见问题（FAQ）

**Q: Joern支持哪些编程语言？**
A: Joern支持C/C++、Java、JavaScript/TypeScript、Python、Go、PHP、C#、Ruby、Swift、Kotlin等主流编程语言，还支持通过ghidra2cpg分析二进制文件。

**Q: Joern和其他SAST工具（如SonarQube、Checkmarx）有什么区别？**
A: Joern是一个开源的代码分析平台，提供了灵活的查询语言和可扩展的架构，适合研究和定制。商业SAST工具通常提供开箱即用的规则库和企业级支持。

**Q: CPG文件有多大？**
A: 取决于代码库大小。对于Linux内核这样的大型项目，使用FlatGraph后，压缩的CPG文件约400MB，在内存中约2.6GB（比旧版本减少40%）。

**Q: 如何在Joern中进行污点分析？**
A: 使用`reachableBy`方法：
```scala
val sources = cpg.method.name("getParameter").callIn
val sinks = cpg.method.name(".*execute.*").callIn
sinks.where(_.argument.reachableBy(sources))
```

**Q: Joern的性能如何？**
A: Joern v4.0使用FlatGraph后，性能显著提升。生成Linux内核的CPG约需10-30分钟，查询通常在毫秒到秒级完成。

### B. 术语表

- **CPG (Code Property Graph)**：代码属性图，融合AST、CFG、PDG等的图表示
- **AST (Abstract Syntax Tree)**：抽象语法树，代码的语法结构
- **CFG (Control Flow Graph)**：控制流图，程序的执行路径
- **PDG (Program Dependence Graph)**：程序依赖图，数据和控制依赖关系
- **Pass**：CPG转换步骤，每个Pass执行特定的分析或增强
- **DiffGraph**：批量更新机制，用于高效修改CPG
- **FlatGraph**：Joern v4.0+使用的高性能图存储引擎
- **Overlay**：可选的分析层，如ControlFlow、CallGraph等
- **Frontend**：语言前端，负责将特定语言代码转换为CPG

### C. 引用本文档

如果您在学术论文中使用了本文档的内容，建议使用以下引用格式：

```
Joern Project (2026). Joern完整介绍与开发指南. 
GitHub: https://github.com/joernio/joern
```

---

**文档版本**：1.0  
**最后更新**：2026年2月5日  
**适用Joern版本**：v2.0+（FlatGraph架构）  
**作者**：根据Joern官方文档和源代码编写  

祝您的毕业论文写作顺利！如有任何问题，请参考官方文档或在Joern社区提问。
