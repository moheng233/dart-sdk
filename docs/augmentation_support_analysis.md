# Dart SDK 增强（Augmentation）特性支持分析

## 概述

本文档详细分析了 Dart SDK 对 `augment` 增强特性的支持情况。增强（Augmentation）是 Dart 语言的一个实验性特性，允许开发者从外部增强和扩展现有的声明，为静态元编程和宏（Macros）功能提供基础支持。

## 特性状态

### 实验性标志

根据 `tools/experimental_features.yaml` 配置：

- **特性名称**: `augmentations`
- **实验性发布版本**: 3.6.0
- **当前状态**: 实验性特性（未默认启用）
- **配置路径**: `tools/experimental_features.yaml` 第 157-159 行

```yaml
augmentations:
  experimentalReleaseVersion: '3.6.0'
  help: "Augmentations - enhancing declarations from outside"
```

### 相关的实验性特性

增强特性与以下特性密切相关：

1. **Macros（宏）**
   - 实验性发布版本: 3.3.0
   - 状态: 实验性特性
   - 描述: 静态元编程支持
   - 测试中需要使用 `--enable-experiment=macros` 标志

2. **Enhanced Parts（增强型 Parts）**
   - 实验性发布版本: 3.6.0
   - 状态: 实验性特性
   - 描述: 泛化 parts，支持嵌套以及导出/导入

## 核心组件支持

### 1. 扫描器与解析器（Scanner & Parser）

#### 关键字定义

位置: `pkg/_fe_analyzer_shared/lib/src/scanner/token.dart`

```dart
static const Keyword AUGMENT = const Keyword(
  /* index = */ 86,
  "augment",
  "AUGMENT",
  KeywordStyle.builtIn,
  isModifier: true,
);
```

`augment` 被定义为：
- 内置关键字（built-in keyword）
- 修饰符类型
- 索引为 86

#### 解析器支持

位置: `pkg/_fe_analyzer_shared/lib/src/parser/parser_impl.dart`

解析器实现了对以下语法的支持：
- `augment super` 表达式解析（第 7742, 8035, 8040 行）
- 库增强声明解析
- 导入增强（import augment）解析

相关方法：
```dart
Token parseAugmentSuperExpression(Token token, IdentifierContext context)
listener.handleAugmentSuperExpression(augmentToken, superToken, context)
```

### 2. 前端编译器（Front-end Compiler）

#### 增强迭代器

文件: `pkg/front_end/lib/src/builder/augmentation_iterator.dart`

实现了用于遍历原始声明及其所有增强的迭代器：

```dart
class AugmentationIterator<T> implements Iterator<T> {
  final T _origin;
  final List<T>? _augmentations;
  // ...
}
```

功能：
- 按顺序遍历原始声明和所有增强
- 支持增强链（augmentation chain）的完整迭代

#### 增强降级（Lowering）

文件: `pkg/front_end/lib/src/kernel/augmentation_lowering.dart`

核心功能：
- 为增强成员生成合成名称
- 命名规则：`_#<name>#augment<index>`
- 支持增强层索引管理

示例命名方案：
```dart
// origin.dart
void method() {}  // 索引 0，名称: _#method#augment0

// augment1.dart
augment void method() {}  // 索引 1，名称: _#method#augment1

// augment2.dart
augment void method() {}  // 最后一个增强使用原始名称 'method'
```

#### 内部 AST 节点

文件: `pkg/front_end/lib/src/kernel/internal_ast.dart`

定义了增强相关的内部 AST 节点：
- `AugmentSuperInvocation` - augment super 调用
- `AugmentSuperGet` - augment super 获取操作
- `AugmentSuperSet` - augment super 设置操作

#### 表达式生成器

文件: `pkg/front_end/lib/src/kernel/expression_generator.dart`

实现了 `AugmentSuperAccessGenerator` 类，用于生成 `augment super` 访问表达式。

### 3. 分析器（Analyzer）

#### 元素模型支持

位置: `pkg/analyzer/lib/dart/element/element.dart`

关键 API：

1. **元素类型扩展**
   - `ElementKind.AUGMENTATION_IMPORT` - 增强导入类型
   - `ElementKind.CLASS_AUGMENTATION` - 类增强类型
   - `ElementKind.LIBRARY_AUGMENTATION` - 库增强类型

2. **isAugmentation 属性**（第 1097-1101 行）
   ```dart
   /// Whether the element is an augmentation.
   /// Executable elements are augmentations if they are explicitly marked as
   /// such using the 'augment' modifier.
   bool get isAugmentation;
   ```

3. **Fragment（片段）概念**
   
   引入了 Fragment 概念来表示单个声明站点：
   - `Fragment.nextFragment` - 增强链中的下一个片段
   - `Fragment.previousFragment` - 增强链中的前一个片段
   - `Fragment.isAugmentation` - 片段是否为增强

#### 元素模型迁移

文档: `pkg/analyzer/doc/element_model_migration_guide.md`

分析器在 7.4 版本引入了新的元素模型以支持增强特性：

主要变更：
- 引入 `Fragment` 类表示单个声明站点
- 支持元素的多重声明（通过增强）
- 元素可以分布在多个 parts 中
- 类名添加 `2` 后缀以支持渐进迁移（如 `LibraryElement2`、`ClassElement2`）

相关说明（第 14-32 行）：
- 支持增强库（augmentation libraries）特性规范
- 增强允许从外部增强现有声明
- 每个元素可以有一个或多个 fragments

#### 诊断支持

分析器实现了 8+ 个增强相关的诊断测试：

测试文件（`pkg/analyzer/test/src/diagnostics/`）：
1. `augmentation_modifier_extra_test.dart` - 额外的增强修饰符
2. `augmentation_type_parameter_bound_test.dart` - 类型参数边界
3. `augmentation_extends_clause_already_present_test.dart` - extends 子句已存在
4. `augmentation_type_parameter_count_test.dart` - 类型参数数量
5. `augmentation_type_parameter_name_test.dart` - 类型参数名称
6. `augmentation_of_different_declaration_kind_test.dart` - 不同声明类型的增强
7. `augmentation_modifier_missing_test.dart` - 缺少增强修饰符
8. `augmentation_without_declaration_test.dart` - 没有声明的增强

#### Analyzer Changelog

关键更新记录（`pkg/analyzer/CHANGELOG.md`）：

**7.4.1 版本**:
- 恢复 `InstanceElement.augmented` getter
- 注意：PropertyAccessor 作为增强时可能没有对应的 variable

**7.4.0 版本**:
- 弃用 `PropertyInducingElement get variable`，改用 `variable2`
- 原因：当属性访问器是增强且没有对应声明时，没有对应的变量
- Extension 增强不允许有 `onClause`

### 4. LSP 和工具支持

#### Code Lens 支持

文件: `pkg/analysis_server/lib/src/lsp/handlers/code_lens/augmentations.dart`

提供了 `AugmentationCodeLensProvider` 类，为增强提供 Code Lens 功能。

#### 自定义处理器

文件位置：
- `pkg/analysis_server/lib/src/lsp/handlers/custom/handler_augmentation.dart`
- `pkg/analysis_server/lib/src/lsp/handlers/custom/handler_augmented.dart`

实现了：
- `AugmentationHandler` - 处理增强请求
- `AugmentedHandler` - 处理 augmented 表达式请求

### 5. 测试覆盖

#### 语言测试

目录: `tests/language/augmentation_libraries/`

核心测试文件：
- `class_augmentation_test.dart` (92 行) - 类增强测试
- `class_augmentation.dart` (77 行) - 增强实现

测试文件使用 `--enable-experiment=macros` 标志。

**关键测试特性**：

1. **库声明**
   ```dart
   // 主文件
   import augment "class_augmentation.dart";
   
   // 增强文件
   library augment 'class_augmentation_test.dart';
   ```

2. **类增强**
   ```dart
   augment class A extends B implements I {
     augment List<int> get ints => augmented..add(2);
     augment String get str => '$augmented world';
     // ...
   }
   ```

3. **多重增强**
   - 支持链式增强（同一成员多次增强）
   - 使用 `augmented` 关键字引用前一个增强

4. **支持的增强类型**
   - Getters/Setters
   - 字段
   - 方法
   - 构造函数
   - 运算符
   - Mixins
   - 继承和实现

#### 解析器测试

目录: `pkg/front_end/parser_testcases/augmentation/`

测试文件：
1. `top_level_declarations.dart` - 顶层声明增强
2. `member_declarations.dart` - 成员声明增强
3. `augment_super.dart` - augment super 表达式
4. `top_level_errors.dart` - 顶层错误
5. `member_errors.dart` - 成员错误

每个测试文件都有对应的期望文件：
- `.expect` - 解析结果
- `.parser.expect` - 解析器输出
- `.scanner.expect` - 扫描器输出
- `.intertwined.expect` - 交织输出

#### 宏测试

目录: `pkg/front_end/parser_testcases/macros/`

包含 `augment_class.dart` 及其期望文件，测试在宏上下文中的增强功能。

## 语法支持

### 支持的增强语法

#### 1. 库级增强

```dart
// 增强文件声明
library augment 'target_file.dart';

// 导入增强
import augment 'augmentation_file.dart';
```

#### 2. 顶层声明增强

```dart
augment method() {}
augment void method() {}
augment get getter => null;
augment int get getter => 0;
augment set setter(value) {}
augment void set setter(value) {}
augment var field;
augment final field = 0;
augment const field = 0;
augment int field;
augment late var field;
augment late final field;
augment late int field;
augment class Class {}
augment abstract class Class {}
augment class Class = Object with Mixin;
augment abstract class Class = Object with Mixin;
augment mixin Mixin {}
```

#### 3. 类成员增强

```dart
class Class {
  augment method() {}
  augment void method() {}
  augment get getter => null;
  augment int get getter => 0;
  augment set setter(value) {}
  augment void set setter(value) {}
  augment var field;
  augment final field = 0;
  augment const field = 0;
  augment int field;
  augment late var field;
  augment late final field;
  augment late int field;
  
  // 静态成员
  augment static method() {}
  augment static void method() {}
  augment static get getter => null;
  augment static int get getter => 0;
  augment static set setter(value) {}
  augment static void set setter(value) {}
  augment static var field;
  augment static final field = 0;
  augment static const field = 0;
  augment static int field;
  augment static late var field;
  augment static late final field;
  augment static late int field;
}
```

#### 4. Mixin 成员增强

```dart
mixin Mixin {
  augment method() {}
  augment void method() {}
  // ... (语法同类成员)
}
```

#### 5. `augmented` 表达式

在增强中访问被增强的原始声明：

```dart
// 字段增强
augment String fieldWithInitializer = '${augmented}b';

// Getter 增强
augment String get str => '$augmented world';

// Setter 增强
augment set str(String value) => augmented = '2$value';

// 方法增强
augment String funcWithBody() => 'b${augmented()}';

// 构造函数增强
augment A() : augmentationInitializerInitialized = true {
  augmented();  // 调用原始构造函数
  augmentationConstructorInitialized = true;
}
```

## 文件组织结构

### 核心实现文件

```
dart-sdk/
├── pkg/
│   ├── _fe_analyzer_shared/          # 共享代码
│   │   ├── lib/src/scanner/
│   │   │   └── token.dart            # AUGMENT 关键字定义
│   │   ├── lib/src/parser/
│   │   │   ├── parser_impl.dart      # 增强语法解析
│   │   │   ├── listener.dart         # 增强事件监听
│   │   │   └── modifier_context.dart # 增强修饰符上下文
│   │   └── lib/src/experiments/
│   │       └── flags.dart            # 实验性标志定义
│   │
│   ├── front_end/                    # 前端编译器
│   │   └── lib/src/
│   │       ├── builder/
│   │       │   └── augmentation_iterator.dart  # 增强迭代器
│   │       ├── kernel/
│   │       │   ├── augmentation_lowering.dart  # 增强降级
│   │       │   ├── internal_ast.dart           # 增强 AST 节点
│   │       │   └── expression_generator.dart   # 表达式生成器
│   │       └── util/
│   │           └── parser_ast_helper.dart      # 解析辅助
│   │
│   ├── analyzer/                     # 分析器
│   │   ├── lib/dart/element/
│   │   │   └── element.dart          # 元素模型 API
│   │   ├── lib/src/summary2/         # 摘要生成
│   │   └── doc/
│   │       └── element_model_migration_guide.md  # 迁移指南
│   │
│   └── analysis_server/              # LSP 服务器
│       └── lib/src/lsp/handlers/
│           ├── code_lens/augmentations.dart      # Code Lens
│           └── custom/
│               ├── handler_augmentation.dart     # 增强处理器
│               └── handler_augmented.dart        # augmented 处理器
│
├── tests/
│   └── language/
│       └── augmentation_libraries/   # 语言特性测试
│           ├── class_augmentation_test.dart
│           └── class_augmentation.dart
│
└── tools/
    └── experimental_features.yaml    # 实验性特性配置
```

## 当前限制和注意事项

### 1. 实验性特性

- 增强特性目前为实验性，未默认启用
- 需要通过 `--enable-experiment=macros` 或 `--enable-experiment=augmentations` 启用
- API 可能会在未来版本中发生变化

### 2. 版本要求

- 最早支持版本：3.6.0（实验性）
- 当前 SDK 版本：3.11.0
- 尚未正式发布（enabledIn 字段未设置）

### 3. 与宏的关系

- 增强特性是宏功能的基础
- 测试中常使用 `--enable-experiment=macros` 标志
- 两个特性紧密集成但可以独立使用

### 4. 工具支持

已实现的工具支持：
- ✅ 语法高亮（通过扫描器）
- ✅ 语法解析
- ✅ 语义分析（通过分析器元素模型）
- ✅ LSP 支持（Code Lens、导航等）
- ✅ 编译器支持（前端和 Kernel 层）

### 5. 已知问题和限制

根据诊断测试，已知的验证规则包括：
- 增强修饰符的正确使用
- 类型参数一致性检查
- extends/implements 子句的限制
- 增强必须对应已存在的声明
- 不同声明类型之间的增强限制

## 使用示例

### 基本使用

**主文件 (main.dart)**:
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
  print(counter.count); // 输出: 1
}
```

**增强文件 (augmentation.dart)**:
```dart
library augment 'main.dart';

augment class Counter {
  // 增强 increment 方法，添加日志
  augment void increment() {
    print('Incrementing...');
    augmented(); // 调用原始方法
    print('Incremented!');
  }
}
```

### 链式增强

```dart
// 原始声明
String get message => 'Hello';

// 第一次增强
augment String get message => '$augmented, World';

// 第二次增强
augment String get message => '$augmented!';

// 最终结果: "Hello, World!"
```

### 构造函数增强

```dart
class MyClass {
  late final bool initialized;
  
  MyClass() {
    // 原始构造函数
  }
}

augment class MyClass {
  augment MyClass() {
    augmented(); // 调用原始构造函数
    initialized = true; // 添加额外初始化
  }
}
```

## 技术细节

### 增强链处理

1. **原始声明**（index 0）
   - 第一个声明，作为增强链的起点
   - 生成名称: `_#<name>#augment0`

2. **中间增强**（index 1, 2, ...）
   - 每个增强都有唯一索引
   - 生成名称: `_#<name>#augment<index>`

3. **最终增强**（无索引）
   - 使用原始声明名称
   - 作为外部访问的入口点

### `augmented` 关键字

`augmented` 在不同上下文中的含义：

- **字段**: 引用被增强字段的值
- **Getter**: 引用被增强 getter 的返回值
- **Setter**: 引用被增强 setter 的赋值目标
- **方法**: 作为函数调用，执行被增强的方法
- **构造函数**: 作为函数调用，执行被增强的构造函数

## 开发路线图

### 已完成

- ✅ 基础语法支持（扫描器和解析器）
- ✅ 前端编译器实现
- ✅ 分析器元素模型
- ✅ LSP 工具支持
- ✅ 基础测试覆盖
- ✅ 诊断和错误处理

### 进行中

- 🔄 完善宏系统
- 🔄 增强的稳定性和性能优化
- 🔄 更广泛的测试覆盖

### 计划中

- 📋 特性正式发布（移除实验性标志）
- 📋 完整的文档和示例
- 📋 IDE 集成改进
- 📋 与宏的深度集成

## 相关资源

### 官方文档

1. **增强库规范**
   - GitHub: https://github.com/dart-lang/language/blob/main/working/augmentation-libraries/feature-specification.md
   - 详细的语言规范和语义

2. **增强型 Parts**
   - GitHub: https://github.com/dart-lang/language/blob/main/working/augmentation-libraries/parts_with_imports.md
   - Parts 的增强和扩展

### SDK 内部文档

1. **元素模型迁移指南**
   - 路径: `pkg/analyzer/doc/element_model_migration_guide.md`
   - 分析器 API 迁移说明

2. **实验性特性配置**
   - 路径: `tools/experimental_features.yaml`
   - 特性标志和版本信息

### 代码生成说明

修改实验性特性配置后，需要运行以下命令更新代码：

```bash
# 更新 analyzer
dart pkg/analyzer/tool/experiments/generate.dart
dart pkg/analyzer/tool/api/generate.dart

# 更新 kernel/front_end
dart pkg/front_end/tool/cfe.dart generate-experimental-flags
dart pkg/front_end/tool/update_expectations.dart

# 更新 VM
dart tools/generate_experimental_flags.dart
```

## 总结

Dart SDK 对增强（Augmentation）特性提供了全面的支持，涵盖了从扫描器、解析器到编译器和分析器的完整工具链。该特性目前处于实验性阶段，为静态元编程和宏功能提供了强大的基础。

主要优势：
- 完整的语言支持（扫描、解析、分析、编译）
- 丰富的 LSP 工具集成
- 详细的诊断和错误处理
- 与宏系统的深度集成

开发者可以通过启用实验性标志来试用该特性，但需要注意这是一个正在发展中的特性，API 可能会在未来版本中发生变化。

---

**文档版本**: 1.0  
**SDK 版本**: 3.11.0  
**最后更新**: 2025-11-23  
**作者**: Dart SDK Analysis
