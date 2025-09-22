# The DefinitionProcessor: A Deep Dive

This document provides a detailed analysis of the `DefinitionProcessor`, explaining its workflow and how it leverages `tree-sitter` to extract code definitions, using C/C++ as a concrete example.

### 1. Introduction

The `DefinitionProcessor` is a central component of the graph building pipeline. It is responsible for the second pass of the analysis, where it parses individual source files to identify and ingest code definitions like classes, functions, and methods into the knowledge graph. Its primary goal is to populate the graph with the core code entities and to build the necessary in-memory data structures (like the `function_registry`) that are required for the subsequent call-resolution pass.

### 2. Core Responsibilities

The `DefinitionProcessor` has several key responsibilities:

*   **File Processing**: It takes a source file, reads its content, and uses the appropriate `tree-sitter` parser to generate an Abstract Syntax Tree (AST).
*   **Definition Extraction**: It traverses the AST and uses language-specific `tree-sitter` queries to find and extract all code definitions.
*   **Graph Ingestion**: It creates `Module`, `Class`, `Function`, and `Method` nodes in the graph, along with the structural `DEFINES` relationships.
*   **State Management**: It populates the `function_registry` and `simple_name_lookup` dictionaries, which are critical for the later stages of the analysis.
*   **Import Handling**: It works with the `ImportProcessor` to parse and resolve import statements within each file.
*   **Dependency Analysis**: It can parse project dependency files (like `pyproject.toml`, `package.json`, etc.) to identify and record external dependencies.

### 3. The `process_file` Workflow

The `process_file` method is the main entry point for analyzing a single source file. Here is a step-by-step breakdown of its workflow:

1.  **Path and Module Name Resolution**: It takes the file path, calculates its relative path within the repository, and generates a unique `qualified_name` for the module (e.g., `my_project.src.utils`).

2.  **Parsing with Tree-sitter**: It reads the file's content and selects the correct `tree-sitter` parser based on the file's language (which is determined by its extension). It then calls the parser's `parse()` method to generate a complete AST for the file.

3.  **Module Creation**: It creates a `Module` node in the graph for the file and links it to its parent `Package` or `Folder`.

4.  **Import Processing**: It delegates to the `ImportProcessor` to analyze all the import statements in the file. The `ImportProcessor` builds a mapping of imported names to their fully qualified names, which is essential for resolving symbols later on.

5.  **Definition Extraction with Tree-sitter Queries**: This is the core of the process. The `DefinitionProcessor` uses pre-compiled `tree-sitter` queries to find all the definitions within the AST. The workflow is as follows:
    *   It retrieves the set of queries for the specific language from the `queries` dictionary.
    *   It executes the `functions` query against the AST to find all function nodes.
    *   It executes the `classes` query to find all class, struct, enum, and interface nodes.
    *   For each captured node, it extracts key information like the name, start and end line numbers, and docstrings.

6.  **Node and Relationship Creation**: For each definition found, it creates the corresponding node in the graph (`Function`, `Class`, etc.) and creates a `DEFINES` relationship from the parent `Module` (or `Class` in the case of a method) to the new node.

7.  **State Update**: It registers each new function, method, and class in the `function_registry` with its fully qualified name. This registry is the central lookup table for the entire codebase.

8.  **Return AST**: Finally, it returns the parsed AST, which is then stored in the `BoundedASTCache` for use in the next pass (call processing).

### 4. Concrete Example: C/C++ File Processing

Let's walk through how the `DefinitionProcessor` handles a C++ file.

#### a. File Identification and Language Configuration

When the `GraphUpdater` encounters a file like `src/math/vector.cpp`, it identifies it as a C++ file based on its `.cpp` extension. It then retrieves the C++ configuration from `language_config.py`. This configuration is crucial and looks something like this:

```python
"cpp": create_lang_config(
    file_extensions=[".cpp", ".h", ".hpp", ...],
    function_node_types=["function_definition", "template_declaration", ...],
    class_node_types=["class_specifier", "struct_specifier", ...],
    # Pre-formatted queries for more precise matching
    function_query="""
    (function_definition) @function
    (template_declaration (function_definition)) @function
    ...""",
    class_query="""
    (class_specifier) @class
    (struct_specifier) @class
    ...""",
)
```

This configuration tells the `DefinitionProcessor` exactly which `tree-sitter` node types correspond to functions and classes in C++.

#### b. Parsing and Querying

The `DefinitionProcessor` uses the `tree-sitter-cpp` parser to generate an AST from the content of `vector.cpp`. It then executes the `function_query` and `class_query` against this AST.

For example, if `vector.cpp` contains:

```cpp
namespace math {
  class Vector {
  public:
    Vector(float x, float y);
    float length() const;
  };
}
```

*   The `class_query` `(class_specifier) @class` will match the `class Vector {...}` block.
*   The `function_query` `(function_definition) @function` will match the definitions of the constructor and the `length` method if they are defined in this file.

#### c. Definition Ingestion

For each match, the `DefinitionProcessor` extracts the relevant information:

*   **For the `Vector` class**: It calls `_extract_cpp_class_name` to get the name "Vector". It then uses `build_cpp_qualified_name` to construct the qualified name, taking into account the `math` namespace, resulting in something like `my_project.src.math.vector.math.Vector`.
*   **For the `length` method**: It extracts the name "length" and, because it's inside the `Vector` class, it creates the qualified name `my_project.src.math.vector.math.Vector.length`.

It then creates a `Class` node for `Vector` and a `Method` node for `length` in the graph, and establishes a `DEFINES_METHOD` relationship between them.

### 5. Handling Method Overrides

After all files have been processed, the `GraphUpdater` calls `process_all_method_overrides`. This method iterates through all the methods that have been registered. For each method, it checks the inheritance hierarchy of its class (which was also recorded during the definition processing pass). If it finds a method with the same name in a parent class, it creates an `OVERRIDES` relationship in the graph. This is done in a separate, final pass to ensure that all class and method definitions are available before trying to resolve overrides.

### 6. Conclusion

The `DefinitionProcessor` is a powerful and flexible component that forms the backbone of the graph construction process. Its intelligent use of `tree-sitter` and its language-specific configurations allows it to parse a wide variety of languages in a consistent and structured way. By separating the concerns of parsing, definition extraction, and relationship creation, it provides a clean and maintainable architecture for translating complex source code into a queryable knowledge graph.
