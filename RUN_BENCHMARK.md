# VillagerBench 运行指南

本文档说明如何启动三种经典 Benchmark 任务（Construction / Farming / Puzzle），以及每次实验结束后各类 log 文件的保存位置。

---

## 一、前置准备

### 1. 确认 API Key 配置

`API_KEY_LIST` 文件（项目根目录）中已配置 OpenAI key：

```json
{
   "OPENAI": ["sk-xxxxxxxxxxxx"],
   "Qwen": ["sk-xxxxxxxxxxxx"]
}
```

当前 `start_with_config.py` 已配置为使用 `OPENAI` key + `gpt-5.1` 模型，无需额外修改。

### 2. 确认 Minecraft 服务器已启动

```bash
# 示例（在 MineCraft 目录下）
python env/minecraft_server.py -H 127.0.0.1 -P 25565 -LP 5000 -U Alice -W world -D false
```

---

## 二、三种 Benchmark 任务

### 任务类型对比

| 任务 | `task_type` | 人数 | 核心目标 |
|---|---|---|---|
| **Construction（建造）** | `construction` | 多人（推荐 3） | 根据蓝图协作放置方块，完成建筑 |
| **Farming（农业/烹饪）** | `farming` | 多人（推荐 3） | 采集材料、合成/烹饪，完成食谱目标（如 cake / rabbit_stew） |
| **Puzzle（密室逃脱）** | `puzzle` | 多人（推荐 2-4） | 在多个密室内协作解谜，最终到达出口坐标 |

---

## 三、启动步骤

所有任务均分两步：**① 生成 launch config** → **② 启动实验**。

---

### Step 1：生成 Launch Config

使用 `config.py` 生成对应任务类型的配置文件。

```bash
# Construction（建造）
python config.py --task construction --api_model gpt-5.1 --host 127.0.0.1 --port 25565 --agent_num 3

# Farming（农业）
python config.py --task farming --api_model gpt-5.1 --host 127.0.0.1 --port 25565 --agent_num 3

# Puzzle（密室逃脱）
python config.py --task puzzle --api_model gpt-5.1 --host 127.0.0.1 --port 25565 --agent_num 3
```

执行后，项目根目录会生成对应的 JSON 配置文件：

```
gpt_5_1_launch_config_construction.json
gpt_5_1_launch_config_farming.json
gpt_5_1_launch_config_puzzle.json
```

> **注意**：`api_model` 中的 `-`、`.`、空格、`/` 会自动转换为 `_`，所以 `gpt-5.1` 对应文件名前缀为 `gpt_5_1`。

---

### Step 2：启动实验

```bash
# Construction（建造）
python start_with_config.py --launch_config gpt_5_1_launch_config_construction.json

# Farming（农业）
python start_with_config.py --launch_config gpt_5_1_launch_config_farming.json

# Puzzle（密室逃脱）
python start_with_config.py --launch_config gpt_5_1_launch_config_puzzle.json
```

同时需要在另一个终端启动对应的 judger（裁判进程）。**三种任务各有专属 judger，请勿混用**：

```bash
# Construction（建造）→ build_judger.py
python env/build_judger.py --idx 0 --host 127.0.0.1 --port 25565 \
    --agent_num 3 --agent_names Alice,Bob,Cindy --task_name <task_name>

# Farming（农业/烹饪）→ farm_craft_judger.py
python env/farm_craft_judger.py --idx 0 --host 127.0.0.1 --port 25565 \
    --agent_num 3 --agent_names Alice,Bob,Cindy --task_name <task_name>

# Puzzle（密室逃脱）→ escape_room_judger.py
python env/escape_room_judger.py --idx 0 --host 127.0.0.1 --port 25565 \
    --agent_num 3 --agent_names Alice,Bob,Cindy --task_name <task_name>

# Meta benchmark（当前默认）→ meta_judger.py
python env/meta_judger.py --idx 0 --host 127.0.0.1 --port 25565 \
    --agent_num 1 --agent_names Alice --task_name <task_name>
```

---

### 可选参数说明（start_with_config.py）

| 参数 | 默认值 | 说明 |
|---|---|---|
| `--launch_config` | `qwen_max_launch_config_meta.json` | 指定使用的配置文件 |
| `--controller` | `full` | 控制器版本，`full`（完整）或 `tiny`（轻量） |
| `--enable_memory_agent` | 关闭 | 开启在线 memory 写入（实验功能） |
| `--enable_memory_retrieval` | 关闭 | 开启 memory 检索注入（实验功能） |

**跑标准 benchmark 时，不加任何 memory 相关参数即可，memory 组件默认完全关闭。**

---

## 四、Log 文件保存位置

每个任务完成后，所有 log 都保存在：

```
result/<task_name>/
```

`<task_name>` 由 launch config 中的 `task_name` 字段决定，格式为：
`<任务描述>_<月日时>` （如 `craft_cobbled_deepslate_slab_inventory_2_id7_020513`）

---

### 各 Log 文件说明

| 文件名 | 生成来源 | 内容说明 |
|---|---|---|
| `score.json` | `env/meta_judger.py` | **最终评分**，记录任务是否成功及得分明细 |
| `config.json` | `env/meta_judger.py` | 本次任务的完整配置参数（evaluation_arg 等） |
| `action_log.json` | `start_with_config.py` → 移自 `data/` | Agent 所有动作的完整序列记录 |
| `tokens.json` | `start_with_config.py` → 移自 `data/` | LLM 调用的 token 消耗统计 |
| `TM_history.json` | `pipeline/task_manager.py` | TaskManager 的规划历史（system prompt + 规划结果） |
| `DM_history.json` | `pipeline/data_manager.py` | DataManager 的查询历史（环境状态查询记录） |
| `DM_query.json` | `pipeline/data_manager.py` | DataManager 的原始查询记录 |
| `<AgentName>_history.json` | `pipeline/agent.py` | 每个 Agent（如 Alice）的执行历史（动作+反馈） |
| `<AgentName>_reflect.json` | `pipeline/agent.py` | 每个 Agent 的反思记录（reflect prompt + 结果） |

---

### 示例目录结构

```
result/
└── craft_cobbled_deepslate_slab_inventory_2_id7_020513/
    ├── score.json          ← 得分结果（首先看这个）
    ├── config.json         ← 任务配置
    ├── action_log.json     ← 全部动作序列
    ├── tokens.json         ← token 消耗
    ├── TM_history.json     ← 任务规划历史
    ├── DM_history.json     ← 状态管理历史
    ├── DM_query.json       ← 状态查询记录
    ├── Alice_history.json  ← Alice 执行历史
    └── Alice_reflect.json  ← Alice 反思记录
```

---

## 五、快速参考

```bash
# 完整流程（以 Farming 为例）

# 1. 生成配置
python config.py --task farming --api_model gpt-5.1 --host 127.0.0.1 --port 25565 --agent_num 3

# 2. 终端A：启动实验
python start_with_config.py --launch_config gpt_5_1_launch_config_farming.json

# 3. 终端B：启动裁判（farming 对应 farm_craft_judger）
python env/farm_craft_judger.py --idx 0 --host 127.0.0.1 --port 25565 \
    --agent_num 3 --agent_names Alice,Bob,Cindy --task_name <task_name>

# 4. 查看结果
cat result/<task_name>/score.json
```
