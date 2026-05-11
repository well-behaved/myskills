---
name: apm-doc-gen
description: Generate APM frontend interface documentation from git branch differences. Handles single/double branch comparisons, path prefixes, and JSON examples.
---

# APM 前端接口文档生成器

根据 Git 分支差异，生成前端接口变更说明飞书文档。

---

## 模式检测

| 用户请求模式 | 模式 | 跳转 |
|-------------|------|------|
| "生成接口文档", "接口变更", "前端文档" | GENERATE_DOC | Phase 1-5 |
| 给出分支名 | INPUT_BRANCH | Phase 0 |

---

## PHASE 0: 接收输入 (BLOCKING)

<input_parsing>
用户可能以下列方式提供输入：

**方式 A：单分支**
- 用户给出一个分支名
- 对比：`<branch>` vs `origin/prod`

**方式 B：双分支**
- 用户给出两个分支名
- 对比：`<branchA>` vs `<branchB>`
- 第一个分支为修改分支，第二个为基准分支

**必须确认的信息：**
1. 分支A（必须）：
2. 分支B（可选，默认 origin/prod）：
3. 是否只看某模块/接口前缀，还是全量：

如果信息不完整，询问用户提供。
</input_parsing>

---

## ⚠️ 硬性约束：禁止切换分支（MANDATORY）

**严禁**使用任何会改变当前工作区状态的命令，包括但不限于：
- `git checkout <branch>`
- `git switch <branch>`
- `git checkout <branch> -- <file>`

需要查看其他分支的文件内容或 diff，必须使用以下只读替代命令：

| 目的 | 正确命令 |
|------|---------|
| 查看某分支某文件的完整内容 | `git show <branch>:path/to/File.java` |
| 查看某文件在两分支间的 diff | `git diff <baseB>...<branchA> -- path/to/File.java` |
| 列出某分支的文件树 | `git ls-tree -r <branch> --name-only` |
| 获取差异文件列表 | `git diff --name-only <baseB>...<branchA>` |

> 原因：当前工作区可能有未提交的改动或正在进行中的开发工作，切换分支会破坏现场。

---

## PHASE 1: 确定对比范围

<branch_comparison>
### 1.1 获取分支状态

```bash
# 执行以下命令并行获取信息
git branch -a
git fetch --all

# 确认分支存在
git rev-parse --verify <branchA>
git rev-parse --verify origin/prod  # 或 branchB
```

### 1.2 确定基准分支

```
IF 用户只给了一个分支:
  BASE_BRANCH = origin/prod
  COMPARE_BRANCH = <用户给的分支>

IF 用户给了两个分支:
  BASE_BRANCH = <第二个分支>
  COMPARE_BRANCH = <第一个分支>
```

### 1.3 获取差异

```bash
# 获取所有差异文件
git diff --name-only BASE_BRANCH...COMPARE_BRANCH

# 按文件类型分类
git diff --name-only BASE_BRANCH...COMPARE_BRANCH | grep -E "Controller|Param|DTO|Response"
```
</branch_comparison>

---

## PHASE 2: 识别接口变更

<identify_changes>
### 2.1 找出可能影响接口的改动

**需要关注的文件类型：**
| 文件类型 | 关注原因 |
|---------|---------|
| Controller | 新增/修改/删除接口 mapping |
| Param | 入参字段变更 |
| DTO/Response | 出参字段变更 |
| Service/Mapper | 仅在影响返回结构时关注 |

### 2.2 分析变更类型

对于每个差异文件，分析：

```
新增文件:
  -> 新接口

删除文件:
  -> 删除接口

修改文件:
  -> 检查 @RequestMapping/@GetMapping/@PostMapping 等注解变化
  -> 检查字段新增/删除/类型变化
  -> 检查字段命名变化
```

#### ⚠️ 强制步骤：逐个读取每个 Controller（MANDATORY）

> **必须对 diff 中每个 Controller 文件执行 `git show` 读取完整内容**，逐个列出其中所有新增/修改的接口方法。
> 禁止只读"主要的"Controller 就跳到文档生成阶段——每个 diff 中出现的 Controller 都可能包含新接口。

```
FOR EACH diff 中的 *Controller*.java 文件:
  1. git show <branch>:<filepath>  读取完整内容
  2. 与基准分支 diff，识别新增/修改的 @GetMapping/@PostMapping 等方法
  3. 将每个新增/修改的方法加入 NEW_INTERFACES 或 MODIFIED_INTERFACES
  4. 不得跳过任何 Controller 文件
```

### 2.3 ⚠️ 关键步骤：DTO/Param 变更必须反向追查所有引用方（MANDATORY）

> **本步骤是最容易漏接口的地方。** 当 diff 中存在 DTO/Param/Entity 文件变更但没有对应 Controller 变更时，
> 必须主动搜索所有引用了该 DTO/Param 的 Controller，它们的相关接口全部属于"修改接口"。

**执行规则：**

```
FOR EACH 变更的 DTO/Param/Response 文件:
  1. 提取类名，例如 OpsMaintenanceOrderDTO
  2. 在整个 src 目录下搜索所有引用该类名的 Controller 文件：
     grep -rl "<ClassName>" src --include="*Controller*.java"
  3. 对搜索到的每个 Controller：
     a. 读取文件，找出返回值类型或方法签名中包含该 DTO 的所有接口方法
     b. 将这些接口全部加入 MODIFIED_INTERFACES
  4. 对搜索到的每个 Service 接口（若 Controller 通过 Service 间接返回）：
     同样追查所有调用了该 Service 方法的 Controller
```

**Windows PowerShell 搜索命令（本项目使用）：**

```powershell
# 搜索所有引用变更DTO的Controller文件
Select-String -Path "src\**\*Controller*.java" -Pattern "OpsMaintenanceOrderDTO" -Recurse | Select-Object Filename

# 搜索所有注入了变更Service的Controller文件（间接影响）
Select-String -Path "src\**\*Controller*.java" -Pattern "OpsMaintenanceOrderService" -Recurse | Select-Object Filename
```

> ⚠️ **注意**：即使搜索结果中某个 Controller 只 import 了该 DTO 用于 Excel 导出（非直接返回），
> 只要导出数据中包含变更字段，该 Controller 的导出接口也属于"修改接口"，必须列入文档。

### 2.4 分类汇总

执行完 2.1-2.3 后，汇总所有受影响接口：

```
NEW_INTERFACES: []
MODIFIED_INTERFACES: []    ← 必须包含所有引用了变更DTO的Controller接口
DELETED_INTERFACES: []
```

**自检：** MODIFIED_INTERFACES 中的接口数量是否覆盖了所有引用 Controller 中返回该 DTO 的方法？
</identify_changes>

---

## PHASE 3: 路径前缀规则 (必须执行)

<path_prefix>
### 3.1 规则说明

```
IF Controller 类名不带 "App":
  前缀 = "/hs/operations"

IF Controller 类名带 "App":
  前缀 = "/operations"
```

### 3.2 拼接实际请求路径

```
前端实际请求路径 = 前缀 + Controller mapping + Method mapping
```

> 注意：不需要考虑网关配置与外部路由，按本项目约定拼出"前端实际请求路径"。
</path_prefix>

---

## PHASE 4: 生成文档内容

<generate_content>
### 4.1 接口变更分类处理

#### 新接口
- 必须有请求参数表（表格形式）
- 必须有 **完整 JSON 返回示例**（包含 code/data/msg 结构）
- JSON 中必须带中文注释

#### 修改接口
- 写清楚新增/删除/变化的入参字段
- 写清楚新增/删除/变化的出参字段
- 返回变更 >5 字段：改用 JSON 示例
- 新增对象/数组字段：补齐完整结构示例

#### 删除接口
- 标注"原接口：已移除"
- 保留原接口路径与简要说明

### 4.2 文档格式模板

#### 原接口/修改接口
```
#### {业务模块}-{功能}-{接口名称}(原接口/修改接口）

1. 接口：/hs/operations/xxx/yyy/{id}
2. 请求方式

| 请求参数 | 子参数 | 类型 | 参数名称 | 是否必填 |
|----------|--------|------|----------|----------|
| id | | long | 主键id | 是 |

1. 返回数据
fieldA _字段A说明_
fieldB _字段B说明_
```

> 若返回变更字段 >5，必须改为 JSON 示例（带中文注释）

#### 新接口
```
#### {业务模块}-{功能}-{接口名称}(新接口）

1. 接口：/operations/xxx/yyy
2. post

| 请求参数 | 子参数 | 类型 | 参数名称 | 是否必填 |
|----------|--------|------|----------|----------|
| account | | string | 用户账号 | 是 |

1. 返回数据（新接口必须是 JSON 示例，且带中文注释）
{
  "code": 200, // 状态码
  "data": {},
  "msg": "" // 提示信息
}
```

### 4.3 JSON 示例要求

**新接口：必须用 JSON 示例**
```json
{
  "code": 200, // 状态码
  "data": {
    "fieldA": "value", // 字段A说明
    "fieldB": 123 // 字段B说明
  },
  "msg": "" // 提示信息
}
```

**返回变更 >5 字段：用 JSON 示例**
- 每个关键字段带中文注释
- 对象/数组字段补齐完整结构

**新增对象/数组字段示例：**
```json
"substituteMaterialDto": [
  {
    "materialId": 1, // 物料ID
    "materialName": "物料名称", // 物料名称
    "quantity": 10 // 数量
  }
]
```
</generate_content>

---

## PHASE 5: 输出与验证

<output_verification>
### 5.1 强制覆盖面自检（输出前必须逐项确认）

```
变更类型覆盖：
  - [ ] diff 中有 Controller 变更 → 已处理
  - [ ] diff 中有 DTO/Param/Response 变更 → 已执行 PHASE 2.3 反向追查
  - [ ] 追查到的所有引用 Controller 均已列入文档

接口完整性（针对每个受影响 Controller）：
  - [ ] 该 Controller 下所有返回该 DTO 的方法（findList/findPage/getById/findDisplay等）均已列出
  - [ ] App 端 Controller（类名含 App）是否已单独检查
  - [ ] 模具/计量等平行业务模块是否有同名 Controller 复用同一 DTO

路径前缀：
  - [ ] 含 App 的 Controller → 前缀 /operations
  - [ ] 不含 App 的 Controller → 前缀 /hs/operations
```

### 5.2 输出确认

返回给用户：
1. 文档概要（新增/修改/删除各多少接口）
2. 关键变更点总结
3. 询问是否需要补充或调整
</output_verification>

---

## 快速参考

### 路径前缀速查

| Controller 类名 | 前缀 |
|-----------------|------|
| 不含 App | /hs/operations |
| 含 App | /operations |

### 变更判断速查

| 变更类型 | 处理方式 |
|---------|---------|
| 新增 Controller | 新接口 |
| 删除 Controller | 删除接口 |
| 修改 mapping | 修改接口 |
| 新增字段 | 写入变更清单 |
| 删除字段 | 写入变更清单 |
| 字段类型变化 | 写入变更清单 |
| 字段 >5 个变更 | 用 JSON 示例 |

---

## 反模式

- 不要只关注 mapping 变化，忽略 DTO/Param 字段变化
- 新接口不放 JSON 示例
- 修改接口不放字段变化详情
- 路径前缀拼错
- **【最常见漏洞】DTO/Param 变更后，只记录 diff 中的文件，不追查所有引用该 DTO 的 Controller** → 必须执行 PHASE 2.3
- **【平行模块遗漏】只看 maintenance 模块，忘记 mold/meterage 等平行模块的同名 Controller 复用同一 DTO** → PHASE 2.3 grep 会覆盖全模块
- **【App端遗漏】App Controller 不在 maintenance 目录下，grep DTO 类名搜不到（因为 App Controller 通过 Service 间接返回）** → 必须同时 grep Service 注入名
- **【切换分支】使用 `git checkout`/`git switch` 切换分支读取文件** → 必须用 `git show <branch>:path/to/file` 替代
- **【Controller 跳读】diff 中有多个 Controller 变更，只读了"主要的"一个就开始写文档，遗漏其他 Controller 中的新增接口** → 必须逐个读取每个 diff 中的 Controller 文件，执行 PHASE 2.2 强制步骤
