# VillagerAgent 完整复现指南（含每次启服）

本指南将 `MineCraft` 目录里的服务器启停方式与 `RUN_BENCHMARK.md` 的实验流程合并，形成一套可重复执行的标准步骤。

## 0. 目录与约定

- 项目根目录：`/hpc2hdd/home/jchen873/Skill Memory/VillagerAgent-Minecraft-multiagent-framework`
- Minecraft 服务端目录：`~/Skill Memory/VillagerAgent-Minecraft-multiagent-framework/MineCraft/MineCraft/MineCraft`
- 本文默认：
  - 主机 `host=127.0.0.1`
  - Minecraft 端口 `port=25565`
  - 代理数量 `agent_num=3`（可按需调整）

## 1. 一次性准备（首次）

在项目根目录执行：

```bash
cd "/hpc2hdd/home/jchen873/Skill Memory/VillagerAgent-Minecraft-multiagent-framework"
source venv1/bin/activate
```

确保以下内容已满足：

- `API_KEY_LIST` 已配置（至少 `AGENT_KEY` 可用）
- Python 依赖已安装（`venv1` 已验证可用）
- JS 依赖已安装（可执行 `python js_setup.py`）
- Java 可用（`java -version`）

## 2. 每次实验前：启动 Minecraft 服务器（重点）

### 2.1 进入服务器目录

```bash
cd "/hpc2hdd/home/jchen873/Skill Memory/VillagerAgent-Minecraft-multiagent-framework/MineCraft/MineCraft/MineCraft"
```

说明：

- 路径最前面的 `/` 代表从系统根目录开始查找，不能省略真实前缀（`/hpc2hdd/home/jchen873/...`）。
- 目录名包含空格（如 `Skill Memory`）时，务必使用双引号包裹整个路径。

### 2.2 启动“新世界”服务器（推荐）

```bash
./start_server_fresh.sh
```

说明：

- 脚本会检查并尝试停止已有 `test.jar` 进程。
- 默认会删除旧 `world`，保证每次从干净世界开始。
- 若你想保留旧世界备份，可改用：

```bash
./start_server_fresh.sh --keep-backup
```

### 2.3 停服命令（实验结束后）

```bash
./stop_server.sh
```

该脚本会优雅停止 `test.jar`，超时后再强制结束。

## 3. 运行 Benchmark（结合 RUN_BENCHMARK）

回到项目根目录并激活环境：

```bash
cd "/hpc2hdd/home/jchen873/Skill Memory/VillagerAgent-Minecraft-multiagent-framework"
source venv1/bin/activate
```

### 3.1 生成 launch config

示例（以 `gpt-5.1` 为模型名）：

```bash
python config.py --task construction --api_model gpt-5.1 --host 127.0.0.1 --port 25565 --agent_num 3
python config.py --task farming --api_model gpt-5.1 --host 127.0.0.1 --port 25565 --agent_num 3
python config.py --task puzzle --api_model gpt-5.1 --host 127.0.0.1 --port 25565 --agent_num 3
```

会生成类似文件（模型名会被转为下划线）：

- `gpt_5_1_launch_config_construction.json`
- `gpt_5_1_launch_config_farming.json`
- `gpt_5_1_launch_config_puzzle.json`

### 3.2 启动实验主程序

当前仓库中的 `start_with_config.py` 默认读取：

- `base_agent_multi_test_config.json`

因此你有两种方式：

1) 把你要跑的配置文件重命名为 `base_agent_multi_test_config.json`；或  
2) 修改 `start_with_config.py` 中读取配置文件的那一行。

启动：

```bash
python start_with_config.py
```

### 3.3 启动对应 Judger（另开终端）

`task_type` 与 judger 对应关系：

- `construction` -> `env/build_judger.py`
- `farming` -> `env/farm_craft_judger.py`
- `puzzle` -> `env/escape_room_judger.py`
- `meta` -> `env/meta_judger.py`

示例（construction）：

```bash
python env/build_judger.py --idx 0 --host 127.0.0.1 --port 25565 \
  --agent_num 3 --agent_names Alice,Bob,Cindy --task_name <task_name>
```

示例（farming）：

```bash
python env/farm_craft_judger.py --idx 0 --host 127.0.0.1 --port 25565 \
  --agent_num 3 --agent_names Alice,Bob,Cindy --task_name <task_name>
```

示例（puzzle）：

```bash
python env/escape_room_judger.py --idx 0 --host 127.0.0.1 --port 25565 \
  --agent_num 3 --agent_names Alice,Bob,Cindy --task_name <task_name>
```

## 4. 每次完整复现实验的推荐顺序

1. 在服务器目录执行 `./start_server_fresh.sh`（确保干净世界）  
2. 回项目根目录激活 `venv1`  
3. `python config.py ...` 生成本次配置  
4. 准备 `base_agent_multi_test_config.json`（或改脚本读取文件）  
5. 终端 A：`python start_with_config.py`  
6. 终端 B：启动对应 `judger`  
7. 任务完成后查看 `result/<task_name>/score.json`  
8. 停服：`./stop_server.sh`

## 5. 结果与日志位置

每个任务结果在：

```text
result/<task_name>/
```

关键文件：

- `score.json`：最终评分
- `config.json`：评测配置
- `action_log.json`：动作序列
- `tokens.json`：token 消耗
- `TM_history.json` / `DM_history.json` / `DM_query.json`：中间过程

## 6. 常见问题排查

- 端口冲突：确认没有旧的 `test.jar` 进程，必要时先 `./stop_server.sh`
- 世界状态污染：务必使用 `./start_server_fresh.sh` 重新启动
- judger 无输出：检查 `task_type` 与 judger 是否匹配
- 任务不落盘：确认 `start_with_config.py` 正常运行到结束并写入 `result/<task_name>/`

## 7. 最小可跑示例（Farming）

### 终端 1（启服）

```bash
cd "/hpc2hdd/home/jchen873/Skill Memory/VillagerAgent-Minecraft-multiagent-framework/MineCraft/MineCraft/MineCraft"
./start_server_fresh.sh
```

### 终端 2（主实验）

```bash
cd "/hpc2hdd/home/jchen873/Skill Memory/VillagerAgent-Minecraft-multiagent-framework"
source venv1/bin/activate
python config.py --task farming --api_model gpt-5.1 --host 127.0.0.1 --port 25565 --agent_num 3
# 准备 base_agent_multi_test_config.json
python start_with_config.py
```

### 终端 3（Judger）

```bash
cd "/hpc2hdd/home/jchen873/Skill Memory/VillagerAgent-Minecraft-multiagent-framework"
source venv1/bin/activate
python env/farm_craft_judger.py --idx 0 --host 127.0.0.1 --port 25565 \
  --agent_num 3 --agent_names Alice,Bob,Cindy --task_name <task_name>
```

