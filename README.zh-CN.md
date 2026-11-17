# 历史人物画像系统

[English](README.md) | **中文**

![CI](https://github.com/ChenyangZhao7/historical-persona-lab/actions/workflows/ci.yml/badge.svg)
![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)

以历史人物的行为画像为驱动，进行历史推演。

数据库中的每个人物都被编码成 10 维行为向量，向量从维基百科（也可以使用你提供的补充材料）中抽取。随后，这些向量被用来把一个人与历史人物匹配、寻找跨文明的对应者，并生成有时代依据的「他会怎么做」叙事。

---

## 十个维度

| # | 维度 | 0.0 | 1.0 |
|---|------|-----|-----|
| 1 | `reaction_to_oppression` 面对压迫 | 默默承受 | 积极反抗 |
| 2 | `group_dependency` 群体依赖 | 独狼 | 依赖集体 |
| 3 | `principle_vs_interest` 原则与利益 | 原则高于一切 | 利益高于一切 |
| 4 | `interpersonal_trust` 人际信任 | 高度开放 | 深度多疑 |
| 5 | `change_orientation` 变革倾向 | 维持现状 | 激进变革 |
| 6 | `emotional_stability` 情绪稳定 | 冷静克制 | 情绪冲动 |
| 7 | `core_motivation` 核心动机 | 理想主义 | 生存与权力 |
| 8 | `historical_mission` 历史使命 | 活在当下 | 使命驱动 |
| 9 | `response_to_injustice` 面对不公 | 接受并适应 | 愤怒与反抗 |
| 10 | `expression_style` 表达风格 | 沉默内敛 | 直言强势 |

每个维度还带一个**置信度**分数。置信度低的维度对匹配的贡献更小；当聊天人格或历史重建需要一个完整向量时，会用行为相似的人物来补全。

## 相似度如何报告

一次匹配会报告为**一个数字加一份拆解**，两者都是绝对值——这个人物与你所描述的样子有多像：

| 字段 | 含义 |
|------|------|
| `closeness` | 十个维度上 `1 - \|差值\|` 的加权平均。这是界面唯一展示的相似度数字。 |
| `dimensions` | 逐维度的同一组信息：双方取值、差距、一致程度，以及各自在该维度上的置信度。 |
| `known_dimensions` | 该人物画像覆盖了十个维度中的几个。 |
| `compared_dimensions` | 双方**都有证据**的维度数量——分数实际依赖的就是这些维度。 |

只有双方画像都有证据时，一个维度才会计入。一段从未提到冒险倾向的描述，不能被当成用户恰好坐在该维度的正中间；在这条规则之前，一段简短的描述会让系统去排名「哪个人物最平均」，而不是「哪个人物最像我」。界面会打印 `based on 5 of 10 dimensions`，让单薄的描述可见，而不是隐藏起来。

候选人物还必须覆盖调用方描述的至少 80% 维度（`COVERAGE_SHARE`）。没有这道门槛，一个只在用户六个特征中的两个上有记录的人物，可能凭一个无法与五维分数比较的二维分数排到第一。画像抽取以 `temperature=0` 运行，因此同样的文本总是得到同样的画像——也就得到同样的匹配。

有两样东西是刻意**不展示**的：

- **原始余弦相似度**——早期版本里的「95% 行为相似度」。在未中心化的 0..1 向量上，几乎每一对都接近 0.95：语料中位数人物与任意画像的分数约为 0.95，因此它无法区分强匹配和弱匹配。
- **排名或百分位**——「447 位人物中的前 3%」是语料的属性，不是这一对的属性。再导入一千位人物，同样两个画像会打印出不同的数字。

匹配按 `closeness` 排序，因此列表顺序始终与旁边的数字一致。覆盖少于 `MIN_KNOWN_DIMENSIONS`（6）个维度的人物会被排除，而不是在一个几乎空白的画像上排名；目前有六位低于这条线。他们仍留在数据库和 Database 页面中。

## 功能

| 页面 | 做什么 |
|------|--------|
| **Match（匹配）** | 描述你自己 → 得到你的 10 维画像 → 找到你的历史镜像 |
| **Reverse（反向）** | 任意人物的跨文明行为对应者 |
| **Predict（预测）** | 相似的古代与现代人物以第一人称回答你的问题 |
| **Reconstruct（重建）** | 描述一个行动 → 找到行为上的亲缘人物 → 编造一份符合时代的叙述 |
| **Chat（对话）** | 与人物角色对话 |
| **Database（数据库）** | 添加任意维基百科人物，可选补充来源文本 |
| **Graph（图谱）** | 围绕一个人物的行为邻域 |
| **Locations（地点）** | 人物生平中的关键地点 |
| **Relics（器物）** | 用隐含行为把一个器物描述匹配到人物 |
| **Map（地图）** | 全语料的客户端 PCA 投影 |
| **Benchmark（基准）** | 语料中行为模式的分布 |

匹配支持 5 个预设（Equal、Mission-first、Leadership、Idealist、Emotion）以及逐维度滑块、IndexedDB 向量缓存、可拖动雷达图实时重匹配和深色主题。

## 环境要求

- Python **3.12**——3.10–3.12 可用；更新的解释器可能没有固定版本 numpy/scikit-learn 的 wheel
- Node.js **18+**
- 所选 LLM 提供商的 API key——key 和账单都归你

### 选择 LLM 提供商

提供商层只有一个模块（`backend/services/llm.py`）。在 `.env` 中设置 `LLM_PROVIDER`：

| `LLM_PROVIDER` | Key | 模型变量 | 默认模型 |
|---|---|---|---|
| `anthropic`（默认） | `ANTHROPIC_API_KEY` | `ANTHROPIC_MODEL` | `claude-sonnet-4-6` |
| `deepseek` | `DEEPSEEK_API_KEY` | `DEEPSEEK_MODEL` | `deepseek-chat` |

DeepSeek 通过 OpenAI 兼容的 REST 端点用 `requests` 调用，因此切换提供商不需要额外依赖。`DEEPSEEK_BASE_URL` 默认是 `https://api.deepseek.com`。

## 快速开始

```bash
git clone <your-repo-url> historical-persona-system
cd historical-persona-system

# 后端
python -m venv venv
#  macOS/Linux:  source venv/bin/activate
#  Windows:      venv\Scripts\activate
pip install -r backend/requirements.txt

# 前端
npm --prefix frontend install

# key
cp .env.example .env      # 然后把你的 key 填进 .env
# 或者：export ANTHROPIC_API_KEY=sk-ant-...
```

随后打开 <http://localhost:5173>。

`npm start` 会在 `:8000` 启动 API，在 `:5173` 启动 Vite 开发服务器。开发服务器会代理所有后端路由，因此不需要配置 CORS 或 base URL。`scripts/run-backend.mjs` 会在 Windows 和 macOS/Linux 上找到 `venv/` 里的解释器。

如果要以生产式的单进程运行：

```bash
npm --prefix frontend run build     # 生成 frontend/dist
uvicorn main:app --app-dir backend --port 8000
```

FastAPI 随后会从同一来源提供构建后的前端。

## 数据集

`backend/data/historical.db` 自带 **447 位人物**及其向量。

- **443** 位带有出生年份（`era`）和 `period`，取自 Wikidata 的 `P569`，按下面的历史学分段归类。
- **4** 位无法确定年代（Wikidata 上没有出生日期），保持 `era = NULL`、`period = "unknown"`。他们可被匹配和浏览，但会从任何需要世纪信息的场景中排除，而不是被猜进某个世纪。

按出生年份应用的时期分段：

| `period` | 中文 | 出生年份 | 边界 |
|---|---|---|---|
| `ancient` | 古代 | < 476 | 西罗马帝国灭亡 |
| `medieval` | 中世纪 | 476 – 1499 | |
| `early_modern` | 近代 | 1500 – 1916 | 大航海时代 → 十月革命 |
| `modern` | 现代 | 1917 – 1944 | 十月革命 → 二战结束 |
| `contemporary` | 当代 | 1945 – | 冷战至今 |

`scripts/backfill_era.py` 会从 Wikidata 重建这两列。除非传入 `--apply`，否则只是试运行；它会把 API 响应缓存到 `data/era_cache.json`，并在写入前备份数据库：

```bash
cd backend
python scripts/backfill_era.py            # 只报告
python scripts/backfill_era.py --apply    # 写入
```

Predict 会把 `ancient` 桶里行为最近邻的人物与 `modern` + `contemporary`（出生年份 ≥ 1917）的人物做对比。Reconstruct 只从 `ancient` 里抽取候选。

扩大语料：

```bash
cd backend
python -m uvicorn main:app --port 8000 &   # 导入器需要一个运行中的 API
python bulk_import.py                      # 每个人物约 4 秒，可断点续跑
```

或者使用 **Database** 页面，为维基百科条目太单薄的人物补充来源文本。

### 清理数据集

`backend/scripts/migrate_dataset.py` 会移除退化画像（全零向量、扁平向量、空置信度），合并重名，并清除旧的 `era = 0` 占位值。除非传入 `--apply`，否则只是试运行，并且会先把数据库复制到 `backend/data/backup/`。

```bash
cd backend
python scripts/migrate_dataset.py           # 显示将要改动什么
python scripts/migrate_dataset.py --apply   # 执行
```

## API

| 方法 | 端点 | 描述 |
|------|------|------|
| POST | `/analyze/` | 抓取维基百科（可选补充材料）并抽取画像 |
| GET | `/figures/` | 列出带向量的人物；`?evidence=false` 返回精简数据 |
| POST | `/figures/` | 从原始文本创建人物 |
| POST | `/match/` | 把自我描述与人物匹配（接受 10 个权重） |
| GET | `/reverse/{name}` | 跨文明对应者 |
| POST | `/predict/` | 古代 + 现代人物回答你的问题 |
| POST | `/reconstruct/` | 为一个行动编造历史叙述 |
| POST | `/chat/` | 人物角色对话 |
| POST | `/graph/` | 行为邻域 |
| POST | `/locations/` | 生平地理 |
| POST | `/relics/match/` | 器物归属 |

交互式文档：<http://localhost:8000/docs>。

## 项目结构

```
historical-persona-system/
├── backend/
│   ├── main.py                 # FastAPI 应用、生命周期、静态前端
│   ├── database.py             # 表结构、索引、连接辅助
│   ├── env.py                  # 极简 .env 加载器（无依赖）
│   ├── imputer.py              # 缺失维度重建
│   ├── routers/                # 每个端点组一个模块
│   ├── services/               # 抽取、匹配、预测、重建、聊天……
│   ├── scripts/                # 数据集迁移
│   ├── tests/                  # pytest 测试套件
│   └── data/historical.db      # 447 位人物及其向量
├── frontend/
│   └── src/
│       ├── pages/              # 每个功能一个组件
│       ├── components/         # DraggableRadar、Skeleton、Similarity
│       └── services/           # IndexedDB 向量缓存、客户端相似度
├── scripts/run-backend.mjs     # 跨平台开发启动器
└── render.yaml                 # 单服务部署
```

## 测试

```bash
pip install -r backend/requirements-dev.txt   # 给 venv 加上 pytest
npm test                                       # 等同于：cd backend && python -m pytest tests -v
```

测试套件覆盖相似度数学、缺失值补全、LLM 输出校验，以及用 mock 依赖的 HTTP 层。它从不调用 LLM 提供商。

## 成本与密钥

每次抽取、匹配、预测、重建、聊天回合和器物检索都会发起一次或多次 Anthropic API 调用，费用由 key 的所有者承担。一次预测是 4 次调用，一次重建是 3 次。API 没有身份验证——不要把有余额的 key 留在公开部署上。

## 部署

`render.yaml` 会构建前端并运行一个 Python 服务。把 `DATABASE_PATH` 指向一个挂载磁盘：首次启动时应用会把随包数据库复制过去，然后写入该磁盘。

## 许可证

MIT——见 [LICENSE](LICENSE)。
