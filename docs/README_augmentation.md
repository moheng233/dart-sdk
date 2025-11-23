# Augmentation Support Analysis Documentation

This directory contains comprehensive documentation analyzing the Dart SDK's support for the augmentation feature.

## Documents

### Chinese Version (中文版本)
- **File**: [augmentation_support_analysis.md](./augmentation_support_analysis.md)
- **Size**: ~18KB (666 lines)
- **Language**: Chinese (中文)
- **Description**: 详细分析了 Dart SDK 对 `augment` 增强特性的全面支持，包括实验性标志、核心组件、语法支持、文件结构、使用示例等。

### English Version
- **File**: [augmentation_support_analysis_en.md](./augmentation_support_analysis_en.md)
- **Size**: ~10KB (388 lines)
- **Language**: English
- **Description**: Comprehensive analysis of Dart SDK's support for the `augment` enhancement feature, including experimental flags, core components, syntax support, file structure, usage examples, and more.

## Key Findings

### Feature Status
- **Experimental Feature**: Available since Dart SDK 3.6.0
- **Not Enabled by Default**: Requires `--enable-experiment=macros` or `--enable-experiment=augmentations`
- **Current SDK Version**: 3.11.0

### Supported Components

#### Full Toolchain Support
1. **Scanner & Parser** - Complete syntax support for `augment` keyword and related constructs
2. **Front-end Compiler** - Augmentation lowering, iterator, and AST nodes
3. **Analyzer** - Element model with Fragment concept, comprehensive diagnostics
4. **LSP Tools** - Code Lens, custom handlers for augmentation navigation

#### Test Coverage
- Language tests in `tests/language/augmentation_libraries/`
- Parser test cases in `pkg/front_end/parser_testcases/augmentation/`
- Analyzer diagnostic tests (8+ test files)

### Syntax Overview

```dart
// Import augmentation
import augment "augmentation.dart";

// Declare augmentation file
library augment 'main.dart';

// Augment a class
augment class MyClass {
  // Augment a method
  augment void myMethod() {
    augmented(); // Call original method
    // Additional logic
  }
}
```

## Quick Reference

### Core Files Referenced

| Component | Key Files |
|-----------|-----------|
| Scanner | `pkg/_fe_analyzer_shared/lib/src/scanner/token.dart` |
| Parser | `pkg/_fe_analyzer_shared/lib/src/parser/parser_impl.dart` |
| Experiments | `pkg/_fe_analyzer_shared/lib/src/experiments/flags.dart` |
| Augmentation Iterator | `pkg/front_end/lib/src/builder/augmentation_iterator.dart` |
| Augmentation Lowering | `pkg/front_end/lib/src/kernel/augmentation_lowering.dart` |
| Element Model | `pkg/analyzer/lib/dart/element/element.dart` |
| LSP Support | `pkg/analysis_server/lib/src/lsp/handlers/` |
| Feature Config | `tools/experimental_features.yaml` |

### Related Features
- **Macros** (3.3.0) - Static metaprogramming
- **Enhanced Parts** (3.6.0) - Generalized parts with nesting

### Official Resources
- [Augmentation Libraries Specification](https://github.com/dart-lang/language/blob/main/working/augmentation-libraries/feature-specification.md)
- [Enhanced Parts Specification](https://github.com/dart-lang/language/blob/main/working/augmentation-libraries/parts_with_imports.md)
- [Element Model Migration Guide](../pkg/analyzer/doc/element_model_migration_guide.md)

## Usage

To enable augmentations in your Dart code:

```bash
# Run with experimental flag
dart --enable-experiment=macros your_file.dart

# Or in analysis options
# analysis_options.yaml
analyzer:
  enable-experiment:
    - macros
```

## Development Status

### ✅ Completed
- Basic syntax support
- Front-end compiler implementation
- Analyzer element model
- LSP tool support
- Basic test coverage
- Diagnostics and error handling

### 🔄 In Progress
- Macro system refinement
- Stability and performance optimization
- Broader test coverage

### 📋 Planned
- Official feature release
- Complete documentation and examples
- IDE integration improvements

## Notes

- Both documents contain the same core information in different languages
- The Chinese version is more detailed (666 lines vs 388 lines)
- Documentation is based on analysis of SDK version 3.11.0
- All information is derived from actual codebase analysis, not external sources

---

**Created**: 2025-11-23  
**SDK Version Analyzed**: 3.11.0  
**Maintainer**: Dart SDK Analysis Project
