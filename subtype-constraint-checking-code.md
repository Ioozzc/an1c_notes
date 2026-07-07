# asn1c Subtype 校验代码位置说明

> 配套文档：[fno-constraints-with-uper-analysis.md](./fno-constraints-with-uper-analysis.md)  
> 说明 asn1c 中 **subtype 约束校验**（constraint checking）相关代码在哪里，以及与 UPER 编码约束、`-fno-constraints` 的关系。

---

## 1. 什么是 Subtype 校验代码

ASN.1 里可以为类型添加子类型约束，例如：

```asn1
Age ::= INTEGER (0..150)
Name ::= OCTET STRING (SIZE(1..32))
```

**Subtype 校验代码**是在**运行时**检查 C 结构体中的值是否满足这些约束的代码，例如判断 `age == 200` 是否非法。

它与 **PER/UPER 编码约束表**（`asn_per_constraints_t`）不同：

| 概念 | 类型/符号 | 作用 | `-fno-constraints` |
|------|-----------|------|---------------------|
| Subtype 校验 | `general_constraints` / `memb_*_constraint_*` | 运行时验证值是否合法 | 应禁用 |
| UPER 编码约束 | `asn_PER_*_constr_*` / `per_constraints` | 决定 UPER 比特布局 | 不应禁用 |

---

## 2. 三层结构概览

```
应用层调用 asn_check_constraints()
        ↓
[1] constraints.c          — 统一入口
        ↓
[2] constr_*.c (skeleton) — 通用框架（SEQUENCE/CHOICE/… 递归检查）
        ↓
[3] 编译器生成的 *_constraint() — 具体约束逻辑（(0..150)、SIZE(1..32) 等）
```

---

## 3. 第一层：统一入口（Runtime，固定代码）

| 文件 | 内容 |
|------|------|
| `skeletons/constraints.h` | `asn_check_constraints()` 声明；`asn_constr_check_f` 回调类型 |
| `skeletons/constraints.c` | `asn_check_constraints()` 实现 |

核心逻辑：调用类型描述符上的 `general_constraints` 回调。

```c
// constraints.c
ret = type_descriptor->encoding_constraints.general_constraints(
    type_descriptor, struct_ptr, _asn_i_ctfailcb, &arg);
```

应用层用法：

```c
#include <constraints.h>

char errbuf[128];
size_t errlen = sizeof(errbuf);
if (asn_check_constraints(&asn_DEF_MyPDU, &my_struct, errbuf, &errlen) != 0) {
    /* 约束校验失败 */
}
```

**调用示例**：`skeletons/converter-example.c` 在编解码前后调用 `asn_check_constraints()`。

---

## 4. 第二层：通用校验框架（Runtime Skeleton，固定代码）

各 ASN.1 构造类型和基础类型在 skeleton 中实现 `*_constraint()`，负责按类型结构**递归**检查成员。

### 构造类型

| 文件 | 函数 |
|------|------|
| `skeletons/constr_SEQUENCE.c` | `SEQUENCE_constraint()` |
| `skeletons/constr_SET.c` | `SET_constraint()` |
| `skeletons/constr_CHOICE.c` | `CHOICE_constraint()` |
| `skeletons/constr_SET_OF.c` | `SET_OF_constraint()` |

以 `SEQUENCE_constraint()` 为例：遍历每个成员，调用成员自己的 `general_constraints`：

```c
// constr_SEQUENCE.c
if (elm->encoding_constraints.general_constraints) {
    ret = elm->encoding_constraints.general_constraints(
        elm->type, memb_ptr, ctfailcb, app_key);
    if (ret) return ret;
} else {
    return elm->type->encoding_constraints.general_constraints(...);
}
```

### 基础类型 / 字符串类型

| 文件 | 函数 |
|------|------|
| `skeletons/BIT_STRING.c` | `BIT_STRING_constraint()` |
| `skeletons/BMPString.c` | `BMPString_constraint()` |
| `skeletons/UTF8String.c` | `UTF8String_constraint()` |
| `skeletons/VisibleString.c` | `VisibleString_constraint()` |
| `skeletons/PrintableString.c` | `PrintableString_constraint()` |
| `skeletons/NumericString.c` | `NumericString_constraint()` |
| `skeletons/IA5String.c` | `IA5String_constraint()` |
| `skeletons/UniversalString.c` | `UniversalString_constraint()` |
| `skeletons/UTCTime.c` | `UTCTime_constraint()` |
| `skeletons/GeneralizedTime.c` | `GeneralizedTime_constraint()` |
| `skeletons/OBJECT_IDENTIFIER.c` | `OBJECT_IDENTIFIER_constraint()` |

### 类型描述符中的挂接点

`general_constraints` 定义在 `skeletons/constr_TYPE.h`：

```c
typedef struct asn_encoding_constraints_s {
    const struct asn_oer_constraints_s *oer_constraints;
    const struct asn_per_constraints_s *per_constraints;
    asn_constr_check_f *general_constraints;   /* ← subtype 校验入口 */
} asn_encoding_constraints_t;
```

---

## 5. 第三层：编译器生成的具体校验代码

这是 **`-fno-constraints` 真正要禁用的部分**：针对每个 ASN.1 类型/成员，根据 `INTEGER (0..150)`、`SIZE(1..32)` 等约束生成的 C 函数。

### 5.1 编译器源码（生成逻辑）

| 文件 | 作用 |
|------|------|
| `libasn1compiler/asn1c_constraint.c` | `asn1c_emit_constraint_checking_code()`：生成范围判断、SIZE 检查等 C 代码 |
| `libasn1compiler/asn1c_constraint.h` | 上述函数声明 |
| `libasn1compiler/asn1c_C.c` ~1341 | 生成**类型级** `Foo_constraint()` |
| `libasn1compiler/asn1c_C.c` ~2902–2924 | 生成**成员级** `memb_bar_constraint_N()` |
| `libasn1compiler/asn1c_C.c` ~2872–2886 | 将校验函数挂到 `encoding_constraints.general_constraints` |
| `libasn1compiler/asn1c_C.c` ~3072 | 类型 descriptor 的 `general_constraints` 赋值 |

类型级生成条件（`asn1c_C.c`）：

```c
if (!(arg->flags & A1C_NO_CONSTRAINTS) && expr->combined_constraints) {
    OUT("int\n");
    OUT("%s_constraint(const asn_TYPE_descriptor_t *td, const void *sptr, ...)", p);
    asn1c_emit_constraint_checking_code(arg);
}
```

成员级生成条件：在 `emit_member_table()` 中，当 `expr->constraints` 存在且**未**设置 `A1C_NO_CONSTRAINTS` 时生成 `memb_*_constraint_*`。

### 5.2 生成物（编译 ASN.1 后的输出 `.c` 文件）

对 `MyModule.asn1` 运行 `asn1c` 后，输出目录中会出现类似代码：

```c
static int
memb_age_constraint_1(const asn_TYPE_descriptor_t *td, const void *sptr,
        asn_app_constraint_failed_f *ctfailcb, void *app_key) {
    long value = *(const long *)sptr;
    if (value >= 0 && value <= 150) {
        return 0;
    }
    ASN__CTFAIL(app_key, td, sptr, "...");
    return -1;
}

asn_TYPE_member_t asn_MBR_MySequence_1[] = {
    // ...
    { .encoding_constraints = {
          .per_constraints = &asn_PER_memb_age_constr_1,    /* UPER 编码用 */
          .general_constraints = memb_age_constraint_1        /* subtype 校验 */
      }, ... },
};
```

这些文件**不在 asn1c 源码树内**，位于编译 ASN.1 模块时的 `-D` 输出目录。

---

## 6. 完整调用链

以 `SEQUENCE { age INTEGER (0..150) }` 为例：

```
asn_check_constraints(&asn_DEF_MySequence, &obj, ...)
    │
    ▼
constraints.c
    │  调用 MySequence.encoding_constraints.general_constraints
    ▼
MySequence_constraint()          ← 编译器生成（或委托给 SEQUENCE_constraint）
    │
    ▼
SEQUENCE_constraint()            ← skeletons/constr_SEQUENCE.c
    │  遍历成员 age
    ▼
memb_age_constraint_1()          ← 编译器生成：检查 0..150
    │
    ├─ 合法 → return 0
    └─ 非法 → ASN__CTFAIL(...) → return -1
```

---

## 7. 与 `-fno-constraints` 的关系

| 代码 | 加 `-fno-constraints` 后 |
|------|--------------------------|
| `constraints.c` 入口 | 保留 |
| skeleton `SEQUENCE_constraint()` 等 | 保留 |
| 编译器生成的 `memb_*_constraint_*` / `*_constraint()` | **不生成** |
| descriptor 中 `.general_constraints` | 设为 `0` |
| `asn_PER_*_constr_*`（UPER 编码表） | **应保留**（当前实现有误，见分析文档） |

`-fno-constraints` 的设计意图（asn1c 0.9.6 文档）：

> Do not generate the ASN.1 subtype constraint checking code.

即只关第三层生成物，不影响 UPER 线格式。

---

## 8. 如何用 clangd 导航

| 想看什么 | 跳转目标 |
|----------|----------|
| 校验 API 入口 | `skeletons/constraints.c` → `asn_check_constraints()` |
| 回调类型定义 | `skeletons/constraints.h` → `asn_constr_check_f` |
| SEQUENCE 如何递归检查 | `skeletons/constr_SEQUENCE.c` → `SEQUENCE_constraint()` |
| 编译器如何生成校验逻辑 | `libasn1compiler/asn1c_constraint.c` → `asn1c_emit_constraint_checking_code()` |
| 编译器何时输出校验函数 | `libasn1compiler/asn1c_C.c` ~1341、~2902 |
| 某 ASN.1 类型的具体 `(0..150)` 检查 | 打开该模块 asn1c 生成的 `.c` 文件，搜 `*_constraint` |

**前提**：项目根目录有 `compile_commands.json`（如 `bear -- make` 生成），否则 clangd 可能无法正确解析头文件依赖。

---

## 9. 相关文件索引

### Runtime（skeletons）

- `constraints.h` / `constraints.c` — 入口
- `constr_TYPE.h` — `encoding_constraints` 结构
- `constr_SEQUENCE.c`、`constr_SET.c`、`constr_CHOICE.c`、`constr_SET_OF.c` — 构造类型
- `asn_application.h` — `asn_app_constraint_failed_f` 回调类型
- `converter-example.c` — 使用示例

### 编译器（libasn1compiler）

- `asn1c_constraint.c` / `asn1c_constraint.h` — 校验代码生成核心
- `asn1c_C.c` — 类型/成员 descriptor 与校验函数输出
- `asn1compiler.h` — `A1C_NO_CONSTRAINTS` 标志

### 文档

- [fno-constraints-with-uper-analysis.md](./fno-constraints-with-uper-analysis.md) — 与 UPER 兼容性分析
- [fno-constraints-with-uper-test-plan.md](./fno-constraints-with-uper-test-plan.md) — 修复后测试方案
