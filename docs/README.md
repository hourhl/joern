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

## Official Documentation

For official Joern documentation, please visit:
- Website: https://joern.io
- Docs: https://docs.joern.io/
- CPG Spec: https://cpg.joern.io
- GitHub: https://github.com/joernio/joern

## Contributing

If you'd like to contribute additional documentation, please follow the Joern contribution guidelines and submit a pull request.
