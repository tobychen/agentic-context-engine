# ACE (Agentic Context Engineering) 实现原理分析

## 概述

ACE 是一个自适应学习框架，通过三个核心角色的循环协作，使 LLM 能够从经验中学习并持续改进策略。

## 核心架构

### 1. **Playbook（策略手册）**
- **作用**: 存储和管理学到的策略（Bullet）
- **结构**: 
  - 每个策略包含：内容、所属分类（section）、有用/有害/中立的计数
  - 支持 JSON 序列化，可以保存和加载
- **关键操作**:
  - 增加、更新、删除策略
  - 标记策略的有效性（helpful/harmful/neutral）
  - 应用 Delta 操作批量更新

#### Playbook 存储格式

Playbook 以 **JSON 格式**存储，文件扩展名通常为 `.json`。存储格式如下：

```json
{
  "bullets": {
    "section-00001": {
      "id": "section-00001",
      "section": "Section Name",
      "content": "策略的具体内容",
      "helpful": 5,
      "harmful": 1,
      "neutral": 0,
      "created_at": "2025-01-15T10:30:00.123456+00:00",
      "updated_at": "2025-01-15T11:45:00.789012+00:00"
    }
  },
  "sections": {
    "Section Name": ["section-00001", "section-00002"]
  },
  "next_id": 2
}
```

**字段说明**：
- **bullets**: 字典，key 是 bullet_id，value 是 Bullet 对象的完整信息
  - `id`: 策略的唯一标识符
  - `section`: 策略所属的分类
  - `content`: 策略的具体内容（文本）
  - `helpful`: 该策略被标记为有用的次数
  - `harmful`: 该策略被标记为有害的次数
  - `neutral`: 该策略被标记为中立的次数
  - `created_at`: 创建时间（ISO 8601 格式）
  - `updated_at`: 最后更新时间（ISO 8601 格式）
- **sections**: 字典，key 是分类名称，value 是该分类下的所有 bullet_id 列表
- **next_id**: 下一个可用的 ID 计数器（用于生成新的 bullet_id）

**存储方法**：
```python
# 保存到文件
playbook.save_to_file("my_playbook.json")

# 从文件加载
playbook = Playbook.load_from_file("my_playbook.json")

# 序列化为字符串
json_str = playbook.dumps()

# 从字符串反序列化
playbook = Playbook.loads(json_str)
```

**特点**：
- 使用 UTF-8 编码，支持中文等多语言内容
- JSON 格式缩进为 2 个空格，便于阅读
- 完整的元数据保留（创建时间、更新时间等）
- 可以手动编辑 JSON 文件进行策略调整

### 2. **三个核心角色**

#### Generator（生成器）
- **职责**: 使用 Playbook 中的策略来回答问题
- **工作流程**:
  1. 接收问题、上下文、当前 Playbook
  2. 根据策略进行推理
  3. 返回答案和使用的策略 ID
- **输出**: `GeneratorOutput` 包含推理过程、最终答案、使用的策略 ID

#### Reflector（反思器）
- **职责**: 分析 Generator 的表现，识别问题和成功经验
- **工作流程**:
  1. 接收问题、Generator 输出、环境反馈、正确答案
  2. 分析错误原因和正确方法
  3. 标记哪些策略有用、哪些有害
- **输出**: `ReflectorOutput` 包含错误识别、根因分析、关键洞察、策略标签

#### Curator（策展人）
- **职责**: 根据反思结果更新 Playbook
- **工作流程**:
  1. 接收 Reflector 的反思、当前 Playbook、进度信息
  2. 决定如何修改策略手册
  3. 生成 Delta 操作（添加、更新、标记、删除）
- **输出**: `CuratorOutput` 包含 Delta 操作批次

### 3. **Delta 操作系统（增量更新核心）**

#### DeltaOperation（原子操作）
ACE 定义了四种原子操作类型，每种操作都是自包含的：
- **ADD**: 添加新策略
  - 需要：section（分类）、content（策略内容）
  - 可选：bullet_id（指定ID）、metadata（初始计数）
- **UPDATE**: 修改现有策略
  - 需要：bullet_id（要修改的策略ID）
  - 可选：content（新内容）、metadata（更新计数）
- **TAG**: 更新策略的有用/有害计数
  - 需要：bullet_id、metadata（如 {"helpful": 1} 或 {"harmful": 2}）
- **REMOVE**: 删除策略
  - 需要：bullet_id

#### DeltaBatch（操作批次）
- 包含：reasoning（策展人的推理过程）和 operations（操作列表）
- 支持 JSON 序列化，可持久化保存
- 一个批次可以包含多个操作，按顺序执行

#### 增量更新流程

**第一步：Reflector 标记策略**
```python
# Reflector 分析后，立即标记策略的有效性
self._apply_bullet_tags(reflection)
# 这会直接更新策略的 helpful/harmful/neutral 计数
```

**第二步：Curator 生成 Delta 操作**
```python
# Curator 分析反思结果，生成 Delta 操作
curator_output = self.curator.curate(
    reflection=reflection,
    playbook=playbook,
    question_context=context,
    progress=progress
)
# 返回 CuratorOutput，包含 DeltaBatch
```

**第三步：Playbook 应用 Delta**
```python
# 批量应用所有 Delta 操作
self.playbook.apply_delta(curator_output.delta)
# 内部会逐个执行每个操作
```

#### 应用 Delta 的具体实现

```python
def apply_delta(self, delta: DeltaBatch) -> None:
    for operation in delta.operations:
        self._apply_operation(operation)

def _apply_operation(self, operation: DeltaOperation) -> None:
    if operation.type == "ADD":
        self.add_bullet(...)
    elif operation.type == "UPDATE":
        self.update_bullet(...)
    elif operation.type == "TAG":
        self.tag_bullet(...)
    elif operation.type == "REMOVE":
        self.remove_bullet(...)
```

#### 增量更新的优势

1. **原子性**: 每个操作是独立的，失败不影响其他操作
2. **可追溯**: Delta 操作包含推理过程，可以理解为什么做这些改变
3. **可撤销**: 如果需要，可以记录操作历史并回滚
4. **批量处理**: 一次可以执行多个相关操作
5. **LLM 驱动**: Curator 使用 LLM 智能决定如何更新，而不是硬编码规则

#### Curator 如何生成 Delta

Curator 通过提示工程让 LLM 分析：
- 当前 Playbook 的状态（策略数量、统计信息）
- Reflector 的反思结果（错误分析、成功经验）
- 训练进度和上下文

然后 LLM 输出 JSON 格式的 Delta 操作，包括：
- 推理过程（为什么需要这些更新）
- 操作列表（具体要做什么）

这种设计使得 Playbook 的更新是**增量式**的，而不是每次都重新生成整个 Playbook。

#### 如何找到要修改的策略

ACE 通过以下机制定位需要修改的策略：

**第一步：Generator 记录使用的策略**
```python
# Generator 在生成答案时，会返回使用的策略 ID
generator_output = generator.generate(...)
# generator_output.bullet_ids = ["math-00001", "math-00002"]
```

**第二步：Reflector 提取策略上下文**
```python
# Reflector 通过 bullet_ids 提取实际使用的策略内容
playbook_excerpt = _make_playbook_excerpt(playbook, generator_output.bullet_ids)
# 返回格式：
# [math-00001] 对于两位数乘法，使用分解方法
# [math-00002] 验证计算的中间步骤
```

**第三步：Reflector 分析并标记策略**
```python
# Reflector 的 LLM 分析后返回标记信息
reflection.bullet_tags = [
    BulletTag(id="math-00001", tag="helpful"),
    BulletTag(id="math-00002", tag="harmful")
]
```

**第四步：Curator 分析整个 Playbook**
```python
# Curator 接收完整的 Playbook 和反思结果
curator_output = curator.curate(
    reflection=reflection,
    playbook=playbook,  # 完整的 Playbook 内容
    question_context=context,
    progress=progress
)
```

**策略定位的关键机制**：

1. **通过 bullet_id 唯一标识**：
   - 每个策略有唯一的 ID（如 `math-00001`）
   - 通过 `playbook.get_bullet(bullet_id)` 查找

2. **Generator 返回使用的策略 ID**：
   - Generator 在推理过程中选择策略
   - 输出中包含 `bullet_ids` 列表

3. **Reflector 基于使用情况分析**：
   - 只分析 Generator 实际使用的策略
   - 通过 `playbook_excerpt` 提供上下文

4. **Curator 全局分析**：
   - 接收完整的 Playbook（通过 `playbook.as_prompt()`）
   - LLM 可以：
     - 根据 Reflector 的 `bullet_tags` 更新特定策略
     - 根据 `key_insight` 添加新策略
     - 分析整体模式，发现重复或冲突的策略
     - 通过内容相似度判断是否需要更新

**示例：策略定位流程**

```
1. Generator 生成答案
   → 使用策略: ["math-00001", "math-00002"]
   → 返回: bullet_ids = ["math-00001", "math-00002"]

2. Reflector 分析
   → 提取: playbook_excerpt = 
        "[math-00001] 对于两位数乘法，使用分解方法
         [math-00002] 验证计算的中间步骤"
   → LLM 分析: math-00001 有用，math-00002 有害
   → 返回: bullet_tags = [
        BulletTag(id="math-00001", tag="helpful"),
        BulletTag(id="math-00002", tag="harmful")
    ]

3. Curator 决定更新
   → 接收完整 Playbook（所有策略）
   → LLM 分析：
      - math-00001 多次有用 → TAG 增加 helpful
      - math-00002 多次有害 → 考虑 UPDATE 或 REMOVE
      - 发现缺少"负数乘法"策略 → ADD 新策略
   → 生成 Delta 操作
```

**策略查找方法**：

在代码中，Playbook 提供了查找方法：
```python
# 通过 ID 查找单个策略
bullet = playbook.get_bullet("math-00001")

# 获取所有策略
all_bullets = playbook.bullets()

# 通过 section 查找（需要遍历）
math_bullets = [b for b in playbook.bullets() if b.section == "math"]
```

**LLM 如何知道要修改哪个策略**：

1. **直接引用**：Reflector 已标记的策略 ID（bullet_tags）
2. **内容匹配**：LLM 分析反思中的 `key_insight`，匹配 Playbook 中相似的内容
3. **模式识别**：LLM 识别重复、冲突或需要改进的策略
4. **缺失识别**：发现 Playbook 中缺少的策略，需要 ADD

这种设计使得策略的定位和修改是**智能的、上下文相关的**，而不是简单的硬编码规则。

#### 增量更新示例

假设当前 Playbook 有一个策略：
```
[math-00001] 对于两位数乘法，使用分解方法
```

**场景 1: 发现错误后更新**
```json
{
  "reasoning": "发现策略缺少具体步骤，需要补充",
  "operations": [
    {
      "type": "UPDATE",
      "bullet_id": "math-00001",
      "content": "对于两位数乘法（如 23×45）：先分解为 (20+3)×(40+5)，计算四个乘积后相加",
      "metadata": {"helpful": 1}
    }
  ]
}
```

**场景 2: 添加新策略**
```json
{
  "reasoning": "需要处理负数乘法的情况",
  "operations": [
    {
      "type": "ADD",
      "section": "arithmetic",
      "content": "负数乘法规则：负数×负数=正数，负数×正数=负数",
      "metadata": {"helpful": 1}
    }
  ]
}
```

**场景 3: 标记策略有效性**
```json
{
  "reasoning": "策略使用成功，增加 helpful 计数",
  "operations": [
    {
      "type": "TAG",
      "bullet_id": "math-00001",
      "metadata": {"helpful": 2}
    }
  ]
}
```

**场景 4: 批量操作**
```json
{
  "reasoning": "发现重复策略，合并并删除冗余",
  "operations": [
    {
      "type": "UPDATE",
      "bullet_id": "math-00001",
      "content": "更新后的合并策略..."
    },
    {
      "type": "REMOVE",
      "bullet_id": "math-00005"
    }
  ]
}
```

### 4. **适配器（Adapter）**

#### OfflineAdapter（离线适配器）
- **用途**: 在固定训练集上多轮训练
- **流程**: 
  1. 多个 epoch 迭代训练样本
  2. 每个样本经过 Generator → 环境评估 → Reflector → Curator
  3. 不断更新 Playbook
- **适用场景**: 初始训练，建立基础策略库

#### OnlineAdapter（在线适配器）
- **用途**: 实时学习，持续改进
- **流程**: 
  1. 顺序处理样本流
  2. 每处理一个样本就更新一次 Playbook
  3. 可以处理无限数据流
- **适用场景**: 生产部署，持续学习

## 核心机制

### 学习循环
```
输入问题 → Generator（使用策略）→ 环境评估 
   ↑                                      ↓
Curator ← Reflector（分析反馈）← 获得反馈
   ↓
更新 Playbook
```

### 提示工程
- 使用结构化的提示模板（`prompts.py` 和 `prompts_v2.py`）
- 要求 LLM 输出严格的 JSON 格式
- 包含重试机制处理 JSON 解析失败

### 反馈机制
- **环境评估**: 通过 `TaskEnvironment` 抽象接口
- **策略评分**: 通过 helpful/harmful/neutral 计数
- **反思窗口**: 保留最近 N 次反思作为上下文

### 可观测性
- 集成 Opik 进行追踪和监控
- 记录每个步骤的性能指标
- 支持导出分析数据

## LLM 集成

### 抽象层
- `LLMClient`: 基础接口
- `LiteLLMClient`: 支持 100+ 模型（OpenAI, Claude, Gemini 等）
- `LangChainClient`: LangChain 集成
- `TransformersLLMClient`: 本地模型支持

### 灵活性
- 支持不同模型用于不同角色
- 支持自定义提示模板
- 支持回退和重试策略

## 数据流

1. **输入**: `Sample` (问题 + 上下文 + 真实答案)
2. **生成**: Generator 产生答案
3. **评估**: Environment 提供反馈
4. **反思**: Reflector 分析表现
5. **更新**: Curator 修改策略
6. **迭代**: 下一个样本使用更新后的策略

## 关键特性

1. **自适应**: 从经验中持续学习
2. **可解释**: 策略以人类可读的形式存储
3. **可持久化**: Playbook 可保存和加载
4. **模块化**: 三个角色可独立配置
5. **灵活**: 支持离线训练和在线学习

## 实现亮点

- **JSON 通信**: 所有角色通过结构化 JSON 交互
- **错误处理**: 多层重试机制确保鲁棒性
- **可扩展**: 易于添加新的环境和评估方法
- **类型安全**: 使用 dataclass 和类型注解
- **可追踪**: 内置可观测性支持

## 总结

ACE 通过将"学习"分解为三个明确的角色（生成、反思、策展），实现了一个可解释、可控的自适应系统。核心创新在于将策略显式化为 Playbook，使 LLM 的"记忆"和"学习"过程变得可见和可管理。

