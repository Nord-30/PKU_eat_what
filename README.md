# PKU 吃什么 (PKU What to Eat)

> 北京大学食堂推荐系统 — 基于实时营业状态和供应验证的"出门前决策"工具

## 核心理念

```
Truth > Completeness
Freshness > Quantity
Verification > Recommendation
```

宁可告诉用户"现在无法确认"，也不把可能买不到的东西伪装成"现在可以买到"。

## 项目结构

```
eatwhat/
├── backend/
│   ├── app/
│   │   ├── main.py              # FastAPI 入口
│   │   ├── config.py            # 配置管理
│   │   ├── database.py          # SQLAlchemy 引擎
│   │   ├── models.py            # 数据库模型
│   │   ├── schemas.py           # Pydantic 序列化
│   │   ├── meal_period.py       # 餐段判断逻辑（可配置）
│   │   ├── availability_engine.py  # 核心可用性验证引擎（6级验证）
│   │   ├── blacklist.py         # 黑名单系统（strict/keyword/semantic）
│   │   └── recommender.py       # 推荐算法（透明可解释评分）
│   ├── crawler/
│   │   ├── base.py              # 爬虫基类（Fetcher/Parser/Crawler）
│   │   ├── parser.py            # 数据标准化与验证
│   │   ├── scheduler.py         # APScheduler 定时任务
│   │   └── sources/
│   │       └── pku_dining.py    # 北大餐饮中心官网爬虫
│   ├── seed_data.py             # 初始化种子数据
│   ├── tests/
│   │   ├── test_meal_period.py # 餐段时间判断测试
│   │   ├── test_availability.py # 可用性引擎测试
│   │   ├── test_blacklist.py    # 黑名单匹配测试
│   │   └── test_recommender.py  # 推荐算法测试
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── App.tsx              # 主应用（路由）
│   │   ├── api.ts               # API 客户端
│   │   ├── types.ts             # TypeScript 类型
│   │   ├── pages/
│   │   │   ├── Home.tsx         # 首页（推荐 + 出门按钮）
│   │   │   ├── Blacklist.tsx    # 黑名单管理
│   │   │   ├── DishDetail.tsx   # 餐品详情（验证历史）
│   │   │   ├── Settings.tsx     # 设置页
│   │   │   └── Cafeterias.tsx   # 食堂列表
│   │   └── index.css            # 移动优先样式
│   ├── package.json
│   └── vite.config.ts
├── start.bat                    # Windows 一键启动
├── stop.bat                     # Windows 一键停止
├── docker-compose.yml
├── .env.example
└── README.md
```

## 数据源

本项目数据来源（优先级从高到低）：

| 来源 | URL | 可信度 |
|---|---|---|
| 北大餐饮中心官网 | https://cyzx.pku.edu.cn/ | 0.95 |
| 餐饮中心服务指南 | https://cyzx.pku.edu.cn/fwzn/ | 0.92 |
| 餐饮中心组织机构 | https://cyzx.pku.edu.cn/bmgk/zzjg/ | 0.90 |
| 北大暑期学校餐饮 | http://summer.pku.edu.cn/weixin/life_eat.php | 0.70 |
| 用户手动录入 | — | 0.60 |

## 快速开始

### 方法一：Windows 一键启动（推荐）

双击项目根目录的 **`start.bat`** 即可，无需任何手动操作：

```
双击 start.bat
```

脚本会自动完成：
1. **环境检查** — 检测 Python 和 Node.js 是否已安装，缺失时给出下载地址
2. **端口检查** — 服务已在运行则直接打开浏览器，不重复启动
3. **首次初始化** — 自动安装后端/前端依赖、创建并导入数据库
4. **启动服务** — 分别在新窗口启动后端 (8000) 和前端 (3000)
5. **自动打开浏览器** — 等待后端健康检查通过后打开 http://localhost:3000

| 文件 | 作用 |
|---|---|
| `start.bat` | 一键启动（首次运行自动装依赖，之后秒开） |
| `stop.bat` | 一键停止前后端服务 |

**前提**：已安装 Python 3.10 - 3.14（安装时勾选 *Add to PATH*）和 Node.js 18+。当前依赖的 pydantic-core 尚未提供 Python 3.15+ 的预编译 wheel。

手机等移动设备：同一局域网下访问启动完成后窗口里显示的本机 IP 即可。

### 方法二：手动运行

```bash
# 1. 克隆项目
cd eatwhat

# 2. 启动后端
cd backend
python -m venv venv
# Windows: venv\Scripts\activate
# Linux/Mac: source venv/bin/activate
pip install -r requirements.txt

# 3. 初始化数据库并启动
python seed_data.py
uvicorn app.main:app --reload --port 8000

# 4. 新终端启动前端
cd frontend
npm install
npm run dev
```

打开浏览器访问 http://localhost:3000

### 方法三：Docker Compose

```bash
docker-compose up --build
```

后端: http://localhost:8000 (API 文档: http://localhost:8000/docs)
前端: http://localhost:3000

## 六级验证机制

任何餐品在被推荐前，必须通过以下验证：

### Level 1: 时间判断
判断当前日期、星期几、时间、餐段（早餐/午餐/晚餐/夜宵等）

### Level 2: 食堂营业判断
当前这个食堂在当前时间是否营业

### Level 3: 窗口营业判断
提供该食品的窗口现在是否营业（即使食堂开门，窗口可能没开）

### Level 4: 餐品时段判断
这个食品是不是当前餐段供应（例如：牛肉面可能只在午餐供应）

### Level 5: 数据新鲜度判断
检查 `fetched_at`、`verified_at`、`valid_until`。同一天的菜单数据过有效窗口后**降级为 UNCERTAIN**（"X 分钟前验证，建议到店确认"）而不是直接过期消失——因为官方数据源只提供食堂营业时间，菜品级实时供应无法在线重验；跨天数据保持 EXPIRED 并触发每日滚动

### Level 6: 多源交叉验证
对核心推荐餐品，寻找多个独立来源进行验证，有冲突时标记 UNCERTAIN

## 可信度状态

| 状态 | 含义 | UI 显示 |
|---|---|---|
| CONFIRMED | 多条件满足，最近验证 | ✅ 当前供应已确认 |
| LIKELY | 较新数据，大概率供应 | ✓ 大概率供应 |
| UNCERTAIN | 数据冲突/过期/无法确认 | ⚠️ 无法确认 |
| UNAVAILABLE | 明确不可供应 | ❌ 不可用 |
| EXPIRED | 历史存在，数据已过期 | ⏰ 数据已过期 |

## 黑名单系统

### 匹配模式

| 模式 | 说明 | 示例 |
|---|---|---|
| strict | 精确匹配 | "鱼" 只屏蔽 "鱼" |
| keyword | 关键词匹配 | "鱼" 屏蔽 "红烧鱼"、"酸菜鱼"、"鱼香肉丝" |
| semantic | 语义匹配（智能） | "鱼" 屏蔽 "红烧鱼" 但不屏蔽 "鱼香肉丝" |

### 类型
- 食堂黑名单：屏蔽整个食堂
- 菜品黑名单：按名称/关键词屏蔽菜品
- 窗口黑名单：屏蔽特定窗口
- 标签黑名单：按标签屏蔽（如"spicy"、"fish"）

## 出门前决策流程

点击"🚶 我现在就要出门"按钮时，系统执行：

```
获取当前时间
↓
判断当前餐段
↓
刷新食堂状态
↓
刷新当前餐品
↓
检查营业时间（Level 2）
↓
检查窗口营业时间（Level 3）
↓
检查餐品供应时段（Level 4）
↓
检查数据新鲜度（Level 5）
↓
多源交叉验证（Level 6）
↓
应用用户黑名单
↓
过滤当前不可吃食品
↓
生成候选集合
↓
透明评分排序
↓
输出推荐
```

## 推荐评分模型

```
score = current_availability (0-30)
      + freshness (0-20)
      + confidence (0-15)
      + user_preference (0-15)
      + diversity (0-10)
      + price (0-5)
      - uncertainty_penalty
```

**关键原则**：可吃性过滤发生在排序之前。排序只在当前确认可吃的食品集合中进行。

## API 接口

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | /api/v1/recommend/ | 获取当前推荐 |
| POST | /api/v1/recommend/go-out-now | 出门前重新确认 |
| GET | /api/v1/recommend/dish/{id} | 检查单个餐品可用性 |
| GET | /api/v1/cafeterias/ | 食堂列表 |
| GET | /api/v1/cafeterias/{id} | 食堂详情 |
| GET | /api/v1/cafeterias/{id}/dishes | 食堂菜品 |
| GET | /api/v1/cafeterias/dishes/{id} | 菜品详情+验证历史 |
| GET | /api/v1/cafeterias/meal-periods | 餐段配置 |
| GET | /api/v1/blacklist/ | 黑名单列表 |
| POST | /api/v1/blacklist/ | 添加黑名单 |
| DELETE | /api/v1/blacklist/{id} | 删除黑名单 |
| PATCH | /api/v1/blacklist/{id}/toggle | 启用/禁用 |
| GET | /api/v1/settings/ | 设置 |
| PUT | /api/v1/settings/ | 更新设置 |

## 测试

```bash
cd backend
pip install pytest
pytest tests/ -v
```

测试覆盖：
- 餐段判断（07:30, 11:30, 12:00, 13:50, 17:00, 18:30, 21:00, 23:00）
- 营业时间（营业中、刚开门、刚关门、全天关闭）
- 黑名单（strict/keyword/semantic 三种模式，"鱼香肉丝"边缘案例）
- 数据过期（valid_until 超时降级、同日过期降为 UNCERTAIN、跨天保持 EXPIRED）
- 多源冲突（来源冲突时标记 UNCERTAIN）
- 爬虫失败（网络不可达时系统不崩溃）

## 技术栈

- **后端**: Python 3.12, FastAPI, Pydantic, SQLAlchemy
- **数据采集**: httpx, BeautifulSoup, APScheduler
- **数据库**: SQLite（可切换 PostgreSQL）
- **前端**: React 18, TypeScript, Vite
- **容器**: Docker, docker-compose

## 数据说明

本项目初始数据来自北京大学餐饮中心官网公开页面，包含：
- 12 个主校区食堂
- 40+ 个档口/窗口
- 100+ 道菜品
- 营业时间、位置、联系电话

数据每 30 分钟自动刷新，在早餐、午餐、晚餐前有定时任务。

## 许可

MIT License
