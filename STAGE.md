# ChibaCC 实现计划

## 0. 目标

ChibaCC 的第一目标不是一次性做完“完整工业级 LL* parser generator”，而是先建立一个**可自举、可验证、可逐步增强**的最小闭环：

- 能解析 `.chibacc` 文件
- 能把 grammar 转成内部 IR
- 能生成最小可运行 parser
- 能支持顶层 `start programme`
- 能逐步演进到更强的 lookahead / recovery

一句话：

> 先做出能工作的 ChibaCC，再把它升级成更强的 ChibaCC。

---

## 1. 总体策略

实现分成三层：

1. **meta-parser 层**：解析 `.chibacc` 自身
2. **IR / codegen 层**：把 grammar 变成可生成 parser 的中间表示
3. **runtime 层**：提供 parser 运行时、checkpoint、rewind、best-effort recovery

第一阶段不直接追求“满血 LL*”。

第一阶段采用：

- recursive descent
- checkpoint / rewind
- selective lookahead
- predicate 参与分支选择
- Pratt 专门路径

第二阶段再逐步增强：

- 更系统的预测
- 更少回滚
- 更强的 FIRST / FOLLOW 风格恢复

---

## 2. 非目标（第一阶段明确不做）

以下内容不作为第一阶段交付目标：

- 完整 ANTLR 风格 adaptive LL(*) 预测引擎
- 增量解析
- IDE 级 CST 保真恢复
- 高度优化的 parser 生成质量

第一阶段更关注：

- 架构是否清楚
- 生成结果是否能跑
- 文档中的核心语义是否已经落地

---

## 3. 目录与基础依赖

先建立：

```text
chibacc/
	STAGE.md
	src/
		metalstd/        # 从上游拷贝
		regex/           # 从 chibalex 拷贝
		chibacc/
			ast.chiba
			token.chiba
			lexer.chiba
			parser.chiba
			ir.chiba
			lower.chiba
			codegen.chiba
			runtime.chiba
	main.chiba
```

依赖策略：

- `metalstd` 直接复用现有实现
- `regex` 直接复用 `chibalex/src/regex`
- `chibacc` 只新增 parser generator 自己的逻辑

这样可以避免在工程最开始重复造轮子。

---

## 4. 分阶段计划

## [DONE] M0：工程搭建

### 目标

建立 `chibacc/` 的最小工程骨架，并保证可以独立构建。

### 任务

- 创建 `src/`
- 拷贝 `metalstd`
- 拷贝 `regex`
- 建立 `main.chiba`
- 建立 `chibacc` 命名空间骨架
- 确认 `./main.o --project chibacc` 能跑通最小工程

### 交付物

- 可构建的 `chibacc` 工程
- 空实现但结构完整的模块文件

### 验收

- 项目可编译
- 模块引用无循环错误

---

## [DONE] M1：`.chibacc` 前端（词法 + AST + meta-parser）

### 目标

把 `.chibacc` 文件完整读起来，并直接产出 grammar AST。

### 任务

- 设计 `ChibaccToken`
- 实现 `.chibacc` lexer
- 定义 grammar AST
- 实现 header block 解析
- 实现 `start` 解析
- 实现普通 `rule` 解析
- 实现 `Alt / Seq / Quantifier / Group` 解析
- 实现 `?predicate(...)` 解析
- 实现 Pratt block 解析
- 实现 footer 保留或透传

至少覆盖：

- `#![CHIBACC]`
- header block
- `start RULE_NAME`
- `rule ... ::= ... ;`
- label 绑定
- `=> action`
- `?predicate(...)`
- `pratt { prefix / infix / postfix }`

### AST 方向

建议先有这些节点：

- `File`
- `HeaderBlock`
- `StartDecl`
- `RuleDef`
- `Alt`
- `SeqItem`
- `Predicate`
- `PrattRule`
- `PrefixArm`
- `InfixArm`
- `PostfixArm`

### 交付物

- `.chibacc` lexer
- `.chibacc` AST 定义
- `parse_file(tokens) -> FileAST`

### 验收

- 能把一个最小 `.chibacc` 文件 tokenize
- token 流能打印 / 调试
- 能 parse 文档里的最小示例
- AST 打印结果结构正确

---

## [DONE] M2：规范化 IR + 最小 codegen（普通 rule）

### 目标

在 grammar AST 和 codegen 之间插入稳定 IR，并直接生成不含 Pratt 的基础 parser。

### 任务

- 展开分组 / 可选 / 重复为统一表示
- 统一 label 绑定位置
- 明确 action 挂载点
- 把 Pratt block 降为统一内部结构
- 收集 rule 引用关系
- 为恢复和预测准备 FIRST / FOLLOW 风格信息

### IR 需要表达的信息

- rule 列表
- alt 列表
- 每个 seq item 的种类
- token / rule 引用
- lookahead 边界信息
- predicate 依赖
- Pratt binding power
- 生成 parser runtime 骨架
- 生成 token cursor
- 生成 `mark()` / `rewind()`
- 生成 rule 函数
- 生成 `|` 的 checkpoint / rollback
- 生成 action 调用
- 生成 predicate 调用
- 生成顶层 `start` 包装

### 运行模型

- 单条 `rule` 负责 `token -> AST`
- 顶层 `start` 负责 `Vec[TokenItem] -> LabeledAST`
- 顶层 AST 由用户自己定义，例如 `Programme(items)`

### 交付物

- `lower(FileAST) -> IR`
- 可从 `.chibacc` 生成基础 parser 的生成器

### 验收

- 同一个 grammar AST 能稳定 lower 到可读 IR
- IR 足以支持后续 codegen
- 能处理不含 Pratt 的简单 grammar
- 能生成可编译 parser
- `start programme` 可以产出 `LabeledAST`

---

## [DONE] M3：Pratt 支持

### 目标

支持文档中的 `pratt { prefix / infix / postfix }`。

### 任务

- 生成 Pratt runtime
- 生成 prefix dispatch
- 生成 infix / postfix loop
- 支持固定 binding power
- 支持 `$lhs`
- 支持 Pratt arm 的 action

### 交付物

- Pratt rule codegen

### 验收

- 文档中的表达式示例可以跑通
- 生成结果能构造 `Expr_*`

当前已用以下链路验证：

- `chibacc/examples/pratt_rule.chibacc`
- `chibacc/examples/sample.chibacc`
- `working/sample_parser_runner`

---

## [DONE] M4：best-effort recovery

### 目标

实现与文档一致的、LL* 能做到的最佳努力恢复，并把 recovery 变成所有 rule 都可尝试的统一语义。

### 任务

- 区分“分支失败”和“真正 parse error”
- 建立同步边界推导
- 让普通 rule / Pratt rule / 被引用的子 rule 都先做本地 recovery 尝试，再决定是否向上提升
- 明确“恢复结果”在 rule 边界上的传播方式，避免只有顶层或少数特殊路径能 recover
- 支持：
	- item 级恢复
	- stmt 级恢复
	- 列表分隔符恢复
	- 闭合符恢复
	- Pratt 低优先级停止点恢复
- 顶层只负责在各层 rule recovery 都失败后包装 `LabeledAST.Err(...)`

### 原则

- 不强行构造统一错误 AST 节点
- 尽量保留 `Option[AST]`
- recovery 不是“只有顶层做一次”的逻辑，而是每个 rule 都可以尝试、组合、再提升
- 优先保证 rule 间恢复语义一致，再考虑更激进的局部修复策略
- 恢复失败时尽快提升到更高层边界

### 交付物

- rule-local recovery 语义
- recovery runtime
- recovery-aware codegen

### 注意

这一阶段先把 recovery 做扎实，保证错误输入时的行为稳定、可调试、可验证。重点不是给少数入口“补一个错误返回”，而是让整个 rule 调用链都有一致的 recovery 行为。

### 验收

- 有语法错误的输入不会立刻全停
- 嵌套 rule、列表 rule、Pratt rule 都能先尝试本地 recovery，再决定是否向上失败
- 顶层返回 `LabeledAST.Err(...)`
- 能保留失败 token 片段

当前已用以下链路验证：

- `chibacc/examples/sample.chibacc`
- `chibacc/examples/sample_missing_rparen.source`
- `chibacc/examples/stmt_recovery.chibacc`
- `chibacc/examples/stmt_recovery_missing_expr.source`

---

## 5. 测试计划

测试分四类：

### 5.1 lexer 测试

- `.chibacc` token 覆盖
- 注释 / 空白 / 标点 / 关键字覆盖

### 5.2 parser 测试

- 最小 grammar
- 普通 `rule`
- label / quantifier / group / predicate
- Pratt grammar

### 5.3 codegen 测试

- 生成文件可编译
- 生成 parser 可运行
- 生成 parser 能构造用户 AST

### 5.4 recovery 测试

- 所有 rule 都具备 recovery 尝试能力
- 嵌套 rule 失败后的上抛与继续解析
- 缺失分隔符
- 缺失闭合符
- 错误 token 插入
- 错误 alt 分支回滚
- 顶层 `LabeledAST.Err(...)` 输出

---

## M5：预测与优化

### 目标

在 recovery 已经稳定、测试用例已经固定之后，再推进预测增强和生成质量优化。

### 任务

- 基于 IR 计算更强的 lookahead 信息
- 优化公共前缀 alt 的选择
- 让 predicate 更早参与预测
- 把高频分支从纯回滚改成预测优先

### 原则

- 不为了“更像论文里的 LL*”而打乱已经稳定的 recovery 语义
- 优先做可测量、可回归的优化
- 保持生成结果可读、可调试

### 交付物

- prediction-aware codegen
- 更少回滚的 parser 生成策略

### 验收

- 常见公共前缀 grammar 的回滚次数下降
- predicate 能更早参与分支选择
- 不破坏 recovery 的既有测试结果

---

## 6. 自举与验证

由于整个仓库的工作流要求稳定自举，ChibaCC 也应遵守同样思路：

1. 使用现有稳定 `main.o` 构建 `chibacc`
2. 用构建出的结果再次生成 / 构建，检查输出稳定性
3. 为关键 grammar 准备固定测试用例

建议至少保留：

- 一个最小 grammar 示例
- 一个含 predicate 的 grammar 示例
- 一个含 Pratt 的 grammar 示例
- 一个故意损坏、用于 recovery 的 grammar 示例

---

## 7. 第一版成功标准

ChibaCC 第一版可认为“完成”的标准是：

- 能解析 `.chibacc`
- 能生成 parser
- 支持普通 `rule`
- 支持 predicate
- 支持 Pratt
- 支持顶层 `start programme`
- 发生语法错误时能返回 `LabeledAST.Err(...)`
- 能完成至少一次稳定的双重自举验证

---

## 8. 当前建议的启动顺序

真正开工时按这个顺序推进：

1. 建立 `src/` 与依赖拷贝
2. 写 `token.chiba`
3. 写 `lexer.chiba`
4. 写 `ast.chiba`
5. 写 `parser.chiba`
6. 写 `ir.chiba` / `lower.chiba`
7. 写 `runtime.chiba`
8. 写 `codegen.chiba`
9. 先跑普通 `rule`
10. 再接 Pratt
11. 先做 recovery
12. 补 recovery 测试
13. 最后做预测与优化

---

## 9. 结论

ChibaCC 的实现应采用：

- **先闭环，后增强**
- **先可自举，后做完整预测**
- **先工程可用，后逼近更强 LL***

第一阶段不拒绝 LL*，只是拒绝在系统尚未站稳之前，把最复杂的预测器提前写死。
