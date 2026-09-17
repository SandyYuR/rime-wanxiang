# 万象拼音 custom_phrase 与 abbrev_phrase 体系重构说明

## 背景

随着手机前端越来越丰富，以及新用户不断进入 Rime 生态，9 键、14 键、17 键、18 键等共键布局的需求越来越明显。

传统 Rime 主要面向 26 键输入设计，而共键布局存在一些特殊需求：

- 输入编码不一定等于真实拼音；
- 用户更加依赖候选词联想；
- 希望候选能够恢复真实输入编码到 preedit；
- 希望用户词、简码、公共词库能够统一管理。

但是 Rime 原生体系本身并没有一个特别合适的方式解决这些问题。

或者说：

> Rime 有足够的组件，但缺少一个适合共键布局的组合方式。

---

# 第一阶段：replacer 动态数据库方案

最开始万象采用的是类似现在 replacer 的实现方式。

核心思路：

将简码动态写入数据库，并在加载阶段完成转换和拼接。

同时：

- 将原始输入编码保存到每一个候选值中；
- 候选输出时再通过 Lua 恢复；
- 从而实现九键、共键布局下的编码回显。

整体流程类似：

```
输入编码

↓

动态数据库

↓

加载阶段转换

↓

候选恢复原始编码

↓

preedit显示
```

这个方案确实解决了功能问题。

但是随着数据量增加，问题也逐渐明显。

---

# 存在的问题

## 1. 数据规模带来的性能压力

replacer 本质上需要维护额外的数据：

- 简码转换；
- 数据拼接；
- 编码保存；
- 候选恢复。

如果数据量过大：

- 加载压力增加；
- 查询压力增加；
- 数据维护复杂度增加。

因此不能无限扩大。

---

## 2. 与原有 custom_phrase.txt 体系割裂

传统 Rime 用户词：

```
custom_phrase.txt
```

而 replacer：

```
动态数据库
```

两套体系同时存在。

导致：

- 用户迁移成本增加；
- 用户词和简码使用不同机制；
- 不同实现之间优先级不好统一。

最重要的是：

> 既然用户词本质也是固定数据，为什么不能直接进入 dictionary 体系？

于是开始考虑：

能不能直接复用 Rime dictionary。

---

# 第二阶段：dictionary + script_translator

经过分析发现：

Rime dictionary 本身已经具备：

```
编码
权重
排序
prism转换
多方案复用
```

真正缺少的是：

> 如何让 dictionary 数据拥有 script_translator 的 preedit 能力。

---

# 为什么不用 table_translator？

table_translator 的优势：

- 查询快；
- 结构简单；
- 天然适合固定词库。

但是问题：

它不能完整显示候选注释。

例如：

输入：

```
jys
```

词库：

```
静夜思	jys
```

table 可以输出：

```
静夜思
```

但是无法利用完整 comment 恢复：

```
jys
```

作为 preedit。

---

# 为什么选择 script_translator？

script_translator 可以拿到：

```
candidate.comment

candidate.preedit

candidate.type
```

因此可以：

1. 查询 dictionary；
2. 获取完整注释；
3. 将编码转换为 preedit；
4. 清空 comment；
5. 输出候选。

理论上：

```
dictionary

↓

script_translator

↓

Lua处理

↓

table效果候选
```

---

# 新问题：script_translator 会自动组句

但是 script_translator 最大的问题出现了。

例如：

词库：

```
jys -> 静夜思
```

输入：

```
jys
```

它可能认为：

```
jys jys
```

也是合理输入。

于是产生：

```
静夜思静夜思
```

甚至：

```
静夜思
李白
床前明月光
...
```

参与组句。

这和 table_translator 的行为不一致。

---

# Lua过滤实现 table 化行为

最开始想到：

删除 sentence 类型。

但是发现不够。

因为删除句子后：

还会出现：

```
经
静夜思  #有意思他跑到了第二个
静
精
竟
```

这些派生候选。

所以不能只看 type。

必须判断候选是否完整覆盖当前输入。

核心判断：

```lua
candidate.start == 0

candidate._end == #input
```

只有：

- 从输入开始；
- 到输入结束；

才认为是完整候选。

同时过滤：

- sentence；
- 后续分词；
- 派生候选。

最终：

script_translator 输出行为接近：

```
table_translator
```

---

# 最终方案：一个 Lua，两个翻译器

最终确定：

不再拆成两个 Lua。

而是在一个 filter 中：

同时控制两个 translator。

结构：

```
                Lua Filter

                    |

        -------------------------

        |                       |

 custom_phrase            abbrev_phrase

 用户词置顶                简码游走

```

---

# 两层设计

## 第一层：custom_phrase

作用：

> 对标以前的自定义用户词。

特点：

- 固定词；
- 始终参与；
- 候选整体置顶；
- 不依赖权重竞争。

配置：

```yaml
custom_phrase:

  dictionary: custom_phrase
  prism: wanxiang_phrase_t9

  enable_user_dict: false
  enable_completion: false

  always_show_comments: true
  spelling_hints: 50
```

---

## 第二层：abbrev_phrase

作用：

> 简码候选，根据需求灵活插入。

特点：

- 可以开关；
- 可以控制插入位置；
- 可以控制数量。

配置：

```yaml
abbrev_phrase:

  dictionary: wanxiang_abbrev
  prism: wanxiang_abbrev_t9

  enable_user_dict: false
  enable_completion: false

  always_show_comments: true
  spelling_hints: 50

  insert_position: 6
  max_candidates: 1
```

例如：

```yaml
insert_position: 6
```

表示：

简码插入候选第 6 位。

---

# 用户迁移方式

对于用户来说：

原：

```
custom_phrase.txt
```

变为：

```
custom_phrase.dict.yaml
```

本质没有变化。

只是从：

```
userdb
```

转为：

```
dictionary
```

例如：

以前：

```
静夜思	jys
```

现在：

```yaml
静夜思\n\s\3李白\n床前明月光\n疑似地上霜\n举头望明月\n低头思故乡	jys	5
```

依然保存：

- 编码；
- 权重；
- 内容。

---

# 简码词库设计

简码同样采用 dictionary。

文件：

```
wanxiang_abbrev.dict.yaml
```

例如：

```yaml
---
name: wanxiang_abbrev

version: "LTS"

sort: by_weight

use_preset_vocabulary: false


import_tables:

  - dicts/abbrev

...
```

预设公共简码：

```
aid	aid
ann	ann
```

用户自己的简码：

直接写在：

```
...
```

下面即可。

---

# T9 与其他共键布局

为了降低用户配置复杂度：

T9 使用独立词库：

```
custom_phrase_t9

wanxiang_abbrev_t9
```

14、17、18 等布局：

基于 26 键编码 patch。

原因：

不同共键布局最终映射关系不同。

这是 Rime 架构下无法完全避免的问题。

---

# 第一阶段改造收益

## 1. 简码从 replacer 中拆出

以前：

```
replacer

├── 简码
├── 用户词
├── 数据转换
├── 候选恢复
```

现在：

```
dictionary

├── custom_phrase
└── abbrev_phrase


Lua filter

└── 候选控制
```

职责更加清晰。

---

## 2. 用户词正式进入 Rime dictionary

不再依赖：

```
custom_phrase.txt
```

而使用：

```
*.dict.yaml
```

获得：

- prism；
- 多方案；
- T9转换；
- 统一部署。

---

## 3. replacer 回归真正定位

如果未来 OpenCC 新版本性能提升：

部分预设转换数据可以重新交给 OpenCC。

而 replacer 可以回归：

> 一个高度自由的 Lua 自定义框架。

它可以继续用于：

- 个性化候选加工；
- 特殊规则；
- 高级玩法。

而不是承担简码数据库职责。

---

# 总结

最终结构：

```
两个 dictionary

        +

一个 Lua filter

        +

两个 translator 实例
```

实现：

- 用户词固定置顶；
- 简码任意位置插入；
- 九键 preedit 恢复；
- script_translator 避免自动组句；
- 替代 custom_phrase.txt；
- 保留 Rime 原生 dictionary 体系。

这一次改造的核心不是增加功能，而是：

> 将原本由 replacer 承担的数据职责重新交回 dictionary，把 Lua 限定在候选控制和表现层。

从而让万象拼音在共键输入时代拥有更清晰、更可扩展的结构。