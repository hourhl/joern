# Joern Documentation

This directory contains comprehensive documentation about the Joern code analysis platform.

## Available Documents

### Joern完整介绍与开发指南.md (Chinese)
**Comprehensive Guide to Joern for Thesis Writing**

A complete guide to understanding and developing with Joern, written in Chinese for academic thesis purposes. This document covers:

1. **Introduction to Joern** (第1章)
   - What is Joern?
   - Working principles and architecture
   - Usage scenarios (security analysis, program understanding, code quality, research)

2. **CPG Generation Process** (第2章)
   - Overall architecture with multi-layer pass pipeline
   - Detailed flow from source code to CPG
   - Language frontend → AST creation → Base layer → Optional overlays → FlatGraph storage
   - Key technical details: DiffGraph mechanism, language-agnostic design

3. **CPG Elements** (第3章)
   - 50+ node types (structural, statement, parameter, type, etc.)
   - Edge types (AST, CFG, call, data flow, type relations)
   - Node properties (name, fullName, code, lineNumber, typeFullName, etc.)
   - Example CPG structure for simple C code
   - Value of graph representation

4. **Custom Development Guide** (第4章)
   - Development environment setup (JDK 21, SBT, IDE configuration)
   - Creating custom language frontends (x2cpg)
   - Adding custom CPG passes
   - Writing custom security queries
   - Extending CPG schema with custom nodes and edges
   - Best practices and common SBT commands

5. **Practical Case Study** (第5章)
   - Detecting unsafe Java deserialization vulnerabilities
   - Complete implementation with query code and tests

6. **Learning Resources** (第6章)
   - Official documentation links
   - Key academic papers by Fabian Yamaguchi
   - Related technologies (static analysis, graph databases, ML in code analysis)
   - Community resources

7. **Summary and Appendices** (第7章)
   - Core concepts recap
   - Joern's advantages
   - Application prospects
   - Thesis writing suggestions
   - FAQ and glossary

**File Size**: 41KB (1252 lines)  
**Language**: Chinese (中文)  
**Target Audience**: Graduate students writing thesis on code analysis, Joern developers  
**Joern Version**: v2.0+ (FlatGraph architecture)

---

### C代码解析流程详解.md (Chinese)
**Detailed C Code Parsing Flow in Joern**

An in-depth technical guide explaining how Joern parses C code, written in Chinese. This document provides a complete walkthrough of the parsing pipeline by examining the actual source code. It covers:

1. **Overall Architecture** (第1章)
   - Entry points and core classes (Main, C2Cpg, AstCreationPass, CdtParser, AstCreator)
   - Architecture components and their responsibilities
   - File locations and module organization

2. **Detailed Parsing Flow** (第2章)
   - Flow overview with ASCII diagrams
   - Code path from file discovery to CPG generation
   - Parallel processing with ForkJoinParallelCpgPass

3. **AST Creation Phase** (第3章)
   - CdtParser: Eclipse CDT parsing to IASTTranslationUnit
   - AstCreator: CDT AST → Joern CPG AST conversion
   - Detailed node and edge generation for:
     - FILE, NAMESPACE_BLOCK nodes
     - Function definitions (METHOD, METHOD_PARAMETER_IN, METHOD_RETURN, BLOCK)
     - Variable declarations (LOCAL, CALL for assignment, LITERAL)
     - Control flow statements (CONTROL_STRUCTURE, BLOCK)
     - Function calls (CALL, arguments, LITERAL)
   - Complete code examples from source showing exact node/edge creation

4. **Base Layer Processing** (第4章)
   - All 10 Base layer passes explained:
     - FileCreationPass, NamespaceCreator, TypeDeclStubCreator, MethodStubCreator
     - ParameterIndexCompatPass, MethodDecoratorPass, AstLinkerPass
     - ContainsEdgePass, TypeRefPass, TypeEvalPass
   - What nodes and edges each pass generates
   - Examples of stub creation for external functions/types

5. **CFG Layer Processing** (第5章)
   - CfgCreationPass: Control flow graph construction
   - Examples of CFG for basic blocks, branches, loops
   - CfgDominatorPass: Dominator and post-dominator relationships
   - CdgPass: Control dependence graph

6. **Other Overlay Layers** (第6章)
   - CallGraph layer: CALL → METHOD edge resolution
   - TypeRelations layer: INHERITS_FROM, ALIAS_OF, field access linking

7. **Complete Example** (第7章)
   - Full C program with calculate() and main() functions
   - Step-by-step parsing from source to final CPG
   - Node count: ~60 nodes (FILE, METHOD, LOCAL, CALL, IDENTIFIER, etc.)
   - Edge count: ~150 edges (AST, CFG, REF, EVAL_TYPE, CONTAINS, etc.)
   - Detailed breakdown of every node and edge generated
   
8. **Summary** (第8章)
   - Parsing flow hierarchy table
   - Key design principles (parallelism, two-stage parsing, global context, trait composition)
   - Debugging commands with Joern REPL
   - Resources for further learning

**File Size**: 28KB (1109 lines)  
**Language**: Chinese (中文)  
**Target Audience**: Developers and researchers who need to understand Joern's C parsing internals  
**Joern Version**: v2.0+ (FlatGraph architecture)  
**Source Code References**: Includes line numbers and file paths for all referenced code

---

### Joern数据流分析详解.md (Chinese)
**Comprehensive Guide to Data Flow Analysis in Joern**

A detailed technical guide clarifying which data flow analyses Joern implements, written in Chinese. This document addresses common questions about Joern's analysis capabilities:

1. **Data Flow Analysis Overview** (第1章)
   - Implementation status summary table
   - Architecture and module locations
   - Reaching Definitions ✅ (Fully implemented)
   - Taint Analysis ✅ (Fully implemented)
   - Pointer Analysis ❌ (Not implemented)
   - Constant Propagation ⚠️ (Partially implemented)

2. **Reaching Definitions Analysis** (第2章)
   - MOP (Meet Over all Paths) algorithm implementation
   - ReachingDefPass: Parallel processing with ForkJoinParallelCpgPass
   - DataFlowSolver: Fixed-point iteration algorithm
   - DdgGenerator: Data dependence graph (DDG) edge generation
   - Code examples from ReachingDefPass.scala, DataFlowSolver.scala
   - Usage examples with ddgIn API

3. **Taint Analysis** (第3章)
   - Backward path exploration engine
   - Engine.scala: Core query engine with parallel task solving
   - EngineContext: Configurable context (maxCallDepth, caching)
   - DSL API: reachableBy(), reachableByFlows()
   - Practical examples: buffer overflow, SQL injection, command injection detection

4. **Pointer Analysis** (第4章)
   - Clarification: Not implemented as standalone module
   - Related functionality in DefaultSemantics (addressOf, indirection operators)
   - Why not implemented: Complexity, language-agnostic design, alternative approaches

5. **Constant Propagation** (第5章)
   - Status: Only partially implemented in specific frontends
   - JavaScriptImportResolverPass example: Simple string constant propagation
   - Alternative approaches for constant-related analysis

6. **Semantics System** (第6章)
   - Semantics trait and FlowSemantic
   - DefaultSemantics: 100+ built-in function/operator semantics
   - operatorFlows: Assignment, addition, field access, etc.
   - cFlows: C standard library (strcpy, malloc, sprintf, etc.)
   - javaFlows: Java common methods
   - Custom semantics: NilSemantics, NoCrossTaintSemantics
   - Flow mapping notation: (src, dst) where -1 = return value

7. **Practical Examples** (第7章)
   - Command injection detection with full code
   - SQL injection detection in Java
   - Cross-procedural taint tracking
   - Custom semantics for sanitization functions
   - DDG visualization

8. **Summary** (第8章)
   - Implementation status table
   - Advantages: Parallel efficiency, extensible semantics, cross-language
   - Limitations: No pointer analysis, path explosion, call depth limits
   - Best practices: Engine configuration, custom semantics, staged analysis

**File Size**: 7KB (303 lines, simplified version)  
**Language**: Chinese (中文)  
**Target Audience**: Developers and researchers needing to understand Joern's data flow analysis capabilities  
**Joern Version**: v2.0+ (FlatGraph architecture)  
**Key Clarification**: Addresses the question "Does Joern implement Reaching Definitions, Taint Analysis, Pointer Analysis, and Constant Propagation?" with detailed evidence from source code

---

### reachableByFlows算法详解.md (Chinese)
**Deep Dive into Joern's Taint Analysis Algorithm: reachableByFlows**

A comprehensive technical guide explaining Joern's core taint analysis algorithm in detail, written in Chinese. This document provides an in-depth walkthrough of the `reachableByFlows` implementation:

1. **Overview** (第1章)
   - What is reachableByFlows?
   - API signature and basic usage
   - Backward search strategy

2. **Algorithm Principles** (第2章)
   - Why backward search? (sink → source vs source → sink)
   - Core algorithm idea with pseudocode
   - Key design decisions table

3. **Core Architecture** (第3章)
   - Component relationship diagram (ExtendedCfgNode → Engine → TaskSolver/TaskCreator)
   - Execution model with work-stealing thread pool
   - Parallel task processing flow

4. **Data Structures** (第4章)
   - `TaskFingerprint`: Unique task identifier (node, callSiteStack, callDepth)
   - `PathElement`: Path node with metadata (node, visible, isOutputArg, outEdgeLabel)
   - `ReachableByResult`: Intermediate result (taskStack, path, partial flag)
   - `ReachableByTask`: Work unit with task chain and initial path
   - `TableEntry`: Final result entry
   - `TaskSummary`: Task completion summary

5. **Algorithm Flow** (第5章)
   - Top-level `reachableByFlows()` flow with filtering and deduplication
   - `Engine.backwards()` main loop: submit tasks → process completions → create new tasks
   - Task submission logic: started tracking and held queue
   - Complete flowchart with all steps

6. **Key Components** (第6章)
   - **TaskSolver**: Recursive backward expansion via `results()` method
     - expandIn(): DDG edge traversal with cycle detection
     - Visibility determination based on semantics
     - Task-level caching mechanism
   - **TaskCreator**: New task generation from partial results
     - tasksForParams(): Parameter expansion (Case 1: context-sensitive, Case 2: context-insensitive)
     - tasksForUnresolvedOutArgs(): Return value and output argument expansion
   - **HeldTaskCompletion**: Iterative completion of held tasks

7. **Performance Optimizations** (第7章)
   - Parallel execution with work-stealing pool
   - 3-level caching: task-internal, global result table, optional cross-query
   - 3-level deduplication: within-task, held-task, final-result
   - Cycle detection: path-level and task-chain-level
   - Configuration limits: maxCallDepth, maxArgsToAllow, maxOutputArgsExpansion

8. **Complete Example** (第8章)
   - C code: readInput() → processData() → executeCommand() chain
   - Step-by-step execution trace through all 7 phases
   - Final complete path from fgets (source) to system (sink)
   - Visual path representation with arrows

9. **Configuration Parameters** (第9章)
   - EngineContext and EngineConfig options
   - Tuning recommendations table (simple programs vs complex call chains)

10. **Summary** (第10章)
    - Key features table
    - Algorithm complexity analysis (worst-case O(N × D × B^D))
    - Advantages: precision, scalability, flexibility
    - Limitations: path explosion, depth limits, pointer analysis gaps
    - Best practices checklist

**File Size**: 24KB (1020+ lines)  
**Language**: Chinese (中文)  
**Target Audience**: Developers and researchers needing deep understanding of Joern's taint analysis internals  
**Joern Version**: v2.0+ (FlatGraph architecture)  
**Code References**: Detailed explanations with actual code snippets from:
- ExtendedCfgNode.scala (reachableByFlows entry point)
- Engine.scala (backwards search orchestration)
- TaskSolver.scala (recursive DDG expansion)
- TaskCreator.scala (task generation logic)
- HeldTaskCompletion.scala (held task processing)

**Includes**: Architecture diagrams, execution flow charts, complete worked example with 7-phase trace

---

## Official Documentation

For official Joern documentation, please visit:
- Website: https://joern.io
- Docs: https://docs.joern.io/
- CPG Spec: https://cpg.joern.io
- GitHub: https://github.com/joernio/joern

## Contributing

If you'd like to contribute additional documentation, please follow the Joern contribution guidelines and submit a pull request.
