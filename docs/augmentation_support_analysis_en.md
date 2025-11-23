# Dart SDK Augmentation Feature Support Analysis

## Overview

This document provides a detailed analysis of the Dart SDK's support for the `augment` enhancement feature. Augmentation is an experimental Dart language feature that allows developers to enhance and extend existing declarations from outside their original definition, providing foundational support for static metaprogramming and macros.

## Feature Status

### Experimental Flag

According to `tools/experimental_features.yaml`:

- **Feature Name**: `augmentations`
- **Experimental Release Version**: 3.6.0
- **Current Status**: Experimental (not enabled by default)
- **Config Path**: `tools/experimental_features.yaml` lines 157-159

```yaml
augmentations:
  experimentalReleaseVersion: '3.6.0'
  help: "Augmentations - enhancing declarations from outside"
```

### Related Experimental Features

1. **Macros**
   - Experimental Release Version: 3.3.0
   - Status: Experimental
   - Description: Static metaprogramming
   - Tests require `--enable-experiment=macros` flag

2. **Enhanced Parts**
   - Experimental Release Version: 3.6.0
   - Status: Experimental
   - Description: Generalized parts with nesting and exports/imports

## Core Component Support

### 1. Scanner & Parser

**Keyword Definition** (`pkg/_fe_analyzer_shared/lib/src/scanner/token.dart`):

```dart
static const Keyword AUGMENT = const Keyword(
  /* index = */ 86,
  "augment",
  "AUGMENT",
  KeywordStyle.builtIn,
  isModifier: true,
);
```

**Parser Support** (`pkg/_fe_analyzer_shared/lib/src/parser/parser_impl.dart`):
- `augment super` expression parsing
- Library augmentation declaration parsing
- Import augmentation parsing

### 2. Front-end Compiler

**Key Files**:
- `augmentation_iterator.dart` - Iterator for traversing origin and all augmentations
- `augmentation_lowering.dart` - Generates synthesized names for augmented members
- `internal_ast.dart` - AST nodes: `AugmentSuperInvocation`, `AugmentSuperGet`, `AugmentSuperSet`
- `expression_generator.dart` - `AugmentSuperAccessGenerator` class

**Naming Scheme**:
```dart
// origin.dart
void method() {}  // Index 0, name: _#method#augment0

// augment1.dart
augment void method() {}  // Index 1, name: _#method#augment1

// augment2.dart
augment void method() {}  // Final augmentation uses original name 'method'
```

### 3. Analyzer

**Element Model** (`pkg/analyzer/lib/dart/element/element.dart`):

New element kinds:
- `ElementKind.AUGMENTATION_IMPORT`
- `ElementKind.CLASS_AUGMENTATION`
- `ElementKind.LIBRARY_AUGMENTATION`

Key APIs:
```dart
bool get isAugmentation;  // Whether element is an augmentation
Fragment get nextFragment;  // Next fragment in augmentation chain
Fragment get previousFragment;  // Previous fragment in augmentation chain
```

**Fragment Concept**: Introduced to represent individual declaration sites, supporting multiple declarations per element through augmentations.

**Diagnostic Tests**: 8+ augmentation-related diagnostic tests covering:
- Augmentation modifier validation
- Type parameter consistency
- Extends/implements clause restrictions
- Declaration existence requirements

### 4. LSP & Tool Support

**Code Lens**: `AugmentationCodeLensProvider` for augmentation-specific code lens features

**Custom Handlers**:
- `AugmentationHandler` - Handles augmentation requests
- `AugmentedHandler` - Handles augmented expression requests

## Test Coverage

### Language Tests

**Directory**: `tests/language/augmentation_libraries/`

Core test files:
- `class_augmentation_test.dart` (92 lines)
- `class_augmentation.dart` (77 lines)

**Syntax Examples**:

```dart
// Main file
import augment "class_augmentation.dart";

// Augmentation file
library augment 'class_augmentation_test.dart';

augment class A extends B implements I {
  augment List<int> get ints => augmented..add(2);
  augment String get str => '$augmented world';
}
```

### Parser Test Cases

**Directory**: `pkg/front_end/parser_testcases/augmentation/`

Test files with expectations:
- `top_level_declarations.dart` - Top-level augmentations
- `member_declarations.dart` - Member augmentations
- `augment_super.dart` - Augment super expressions
- `top_level_errors.dart`, `member_errors.dart` - Error cases

## Supported Syntax

### Library-level Augmentation

```dart
// Augmentation file declaration
library augment 'target_file.dart';

// Import augmentation
import augment 'augmentation_file.dart';
```

### Top-level Declaration Augmentation

```dart
augment void method() {}
augment int get getter => 0;
augment void set setter(value) {}
augment int field;
augment class Class {}
augment mixin Mixin {}
```

### Class Member Augmentation

```dart
class Class {
  augment void method() {}
  augment int get getter => 0;
  augment void set setter(value) {}
  augment int field;
  
  // Static members
  augment static void method() {}
  augment static int field;
}
```

### `augmented` Expression

Access the original augmented declaration:

```dart
// Field augmentation
augment String field = '${augmented}b';

// Getter augmentation
augment String get str => '$augmented world';

// Method augmentation
augment String func() => 'b${augmented()}';

// Constructor augmentation
augment MyClass() {
  augmented();  // Call original constructor
  // Additional initialization
}
```

## File Organization

```
dart-sdk/
├── pkg/
│   ├── _fe_analyzer_shared/          # Shared code
│   │   ├── scanner/token.dart        # AUGMENT keyword
│   │   ├── parser/parser_impl.dart   # Augmentation parsing
│   │   └── experiments/flags.dart    # Experimental flags
│   │
│   ├── front_end/                    # Front-end compiler
│   │   └── src/
│   │       ├── builder/augmentation_iterator.dart
│   │       └── kernel/augmentation_lowering.dart
│   │
│   ├── analyzer/                     # Analyzer
│   │   ├── dart/element/element.dart # Element model API
│   │   └── doc/element_model_migration_guide.md
│   │
│   └── analysis_server/              # LSP server
│       └── lsp/handlers/
│           ├── code_lens/augmentations.dart
│           └── custom/handler_augmentation.dart
│
├── tests/language/augmentation_libraries/
└── tools/experimental_features.yaml
```

## Current Limitations

1. **Experimental Feature**: Not enabled by default, requires `--enable-experiment=macros` or `--enable-experiment=augmentations`
2. **Version**: Earliest support in 3.6.0 (experimental), not yet officially released
3. **API Stability**: May change in future versions
4. **Relationship with Macros**: Tightly integrated but can be used independently

## Tool Support Status

- ✅ Syntax highlighting (via scanner)
- ✅ Syntax parsing
- ✅ Semantic analysis (via analyzer element model)
- ✅ LSP support (Code Lens, navigation)
- ✅ Compiler support (front-end and kernel layers)

## Usage Example

**Main file (main.dart)**:
```dart
// SharedOptions=--enable-experiment=macros

import augment "augmentation.dart";

class Counter {
  int count = 0;
  
  void increment() {
    count++;
  }
}

void main() {
  var counter = Counter();
  counter.increment();
  print(counter.count); // Output: 1
}
```

**Augmentation file (augmentation.dart)**:
```dart
library augment 'main.dart';

augment class Counter {
  // Augment increment method with logging
  augment void increment() {
    print('Incrementing...');
    augmented(); // Call original method
    print('Incremented!');
  }
}
```

## Development Roadmap

### Completed
- ✅ Basic syntax support (scanner and parser)
- ✅ Front-end compiler implementation
- ✅ Analyzer element model
- ✅ LSP tool support
- ✅ Basic test coverage
- ✅ Diagnostics and error handling

### In Progress
- 🔄 Macro system refinement
- 🔄 Stability and performance optimization
- 🔄 Broader test coverage

### Planned
- 📋 Official feature release (remove experimental flag)
- 📋 Complete documentation and examples
- 📋 IDE integration improvements
- 📋 Deep integration with macros

## Related Resources

### Official Documentation

1. **Augmentation Libraries Specification**
   - GitHub: https://github.com/dart-lang/language/blob/main/working/augmentation-libraries/feature-specification.md

2. **Enhanced Parts**
   - GitHub: https://github.com/dart-lang/language/blob/main/working/augmentation-libraries/parts_with_imports.md

### SDK Internal Documentation

1. **Element Model Migration Guide**
   - Path: `pkg/analyzer/doc/element_model_migration_guide.md`

2. **Experimental Features Configuration**
   - Path: `tools/experimental_features.yaml`

### Code Generation

After modifying experimental features configuration, run:

```bash
# Update analyzer
dart pkg/analyzer/tool/experiments/generate.dart
dart pkg/analyzer/tool/api/generate.dart

# Update kernel/front_end
dart pkg/front_end/tool/cfe.dart generate-experimental-flags
dart pkg/front_end/tool/update_expectations.dart

# Update VM
dart tools/generate_experimental_flags.dart
```

## Summary

The Dart SDK provides comprehensive support for the augmentation feature across the entire toolchain, from scanner and parser to compiler and analyzer. Currently in experimental stage, this feature provides a powerful foundation for static metaprogramming and macros.

**Key Advantages**:
- Complete language support (scanning, parsing, analysis, compilation)
- Rich LSP tool integration
- Detailed diagnostics and error handling
- Deep integration with macro system

Developers can try this feature by enabling the experimental flag, but should note that this is an evolving feature and APIs may change in future versions.

---

**Document Version**: 1.0  
**SDK Version**: 3.11.0  
**Last Updated**: 2025-11-23  
**Author**: Dart SDK Analysis
