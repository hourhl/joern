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

## Official Documentation

For official Joern documentation, please visit:
- Website: https://joern.io
- Docs: https://docs.joern.io/
- CPG Spec: https://cpg.joern.io
- GitHub: https://github.com/joernio/joern

## Contributing

If you'd like to contribute additional documentation, please follow the Joern contribution guidelines and submit a pull request.
