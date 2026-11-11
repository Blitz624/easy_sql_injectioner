# SQL 注入图形化渗透工具（通用版）

一个**通用**的图形化 SQL 注入漏洞检测与利用工具，不再绑定 DVWA，可适配多种网站与注入点类型，并支持**交互式分步选择**。

## 功能

1. **通用配置**：目标 URL、Cookie、参数名、提交按钮名、数据库类型（MySQL / SQL Server）
2. **漏洞检测**：自动检测注入通道（GET/POST/SESSION）与闭合方式
3. **交互式分步利用**（每一步都由你选择）：
   - 查库名 → **选库**
   - 查表名 → **选表**
   - 查列名 → **多选列**
   - 爆字段 → 多选列按行**一一对应**展示

## 使用流程

1. 填写目标 URL（**唯一必填项**）
2. Cookie、参数名、按钮名、数据库类型**均可留空**——留空时工具会自动探测，而非写死默认值
3. 点「开始检测」→ 自动探测注入通道、闭合方式、列数、回显位
4. 检测完成后弹出**选库**对话框 → 选择目标数据库
5. 弹出**选表**对话框 → 选择目标数据表
6. 弹出**多选字段**对话框 → 勾选要爆出的列（可多选，支持全选/全不选）
7. 程序自动爆出选定列的数据，按行一一对应展示在表格中

## 可选字段的自动探测机制（关键）

除 URL 外，所有输入框**不填也不会写死成一个固定值**，而是从内置候选字典里**逐个尝试**，找到第一个能成功注入的值即停止：

| 字段 | 留空时的行为 |
|------|-------------|
| 参数名 | 依次尝试 `id`、`name`、`user_id`、`uid`、`username`、`page`、`cat`、`keyword`、`q` 等 20 个常见参数名 |
| 提交按钮名 | 依次尝试「不携带按钮」→ `Submit`、`submit`、`button`、`提交`、`登录`、`查询` 等；**若网站无需提交按钮（如纯 GET），会自动适配为不携带** |
| 数据库类型 | 下拉框选「自动探测」，先按 MySQL 试，失败再试 SQL Server |
| 注入位置 | 下拉框选「自动探测」，依次尝试下面 8 种注入点（见下） |
| Cookie | 留空则匿名访问（不带 Cookie 头） |

## 注入点位置（载体）的通用性

真实靶场的 SQL 注入点**不只在 URL 参数里**，本工具抽象了 8 种「注入载体」并自动探测：

| # | 载体 | 说明 | 示例 |
|---|------|------|------|
| 1 | URL 查询参数 | `?id=1` 形式的 GET 注入 | `?id=1' AND 1=1 -- ` |
| 2 | POST 表单字段 | 表单里的普通字段（如 `id`、`name`） | 表单提交 `id=1` |
| 3 | 登录框注入 | `username`/`password` 字段，万能密码 + 联合查询 | `' or '1'='1` |
| 4 | JSON 请求体 | `{"id": 1}` 的接口注入 | `{"id":"1' or 1=1"}` |
| 5 | 请求头注入 | `User-Agent` / `Referer` / `X-Forwarded-For` 等 | `XFF: 1' and 1=1` |
| 6 | Cookie 注入 | Cookie 值本身作为注入点 | `Cookie: id=1' or 1=1` |
| 7 | URL 路径注入 | RESTful 风格 `/article/1` | `/article/1' or 1=1` |
| 8 | SESSION 通道 | 参数从 session 读取（DVWA high 级别） | 先写 session 页再读目标页 |

> 实测：DVWA 的 low/medium/high 三个级别，在**完全不填**任何可选字段的情况下，工具自动识别为：low→URL 查询参数、medium→POST 表单字段、high→SESSION 通道，均成功拖库。

## 自动适配能力

| 能力 | low 级别 | medium 级别 | high 级别 | 其他网站 |
|---|---|---|---|---|
| 注入通道 | GET / POST | POST | SESSION | 自动探测 |
| 注入点 | 字符串闭合 `'...'` | 数值型（无闭合） | 字符串闭合 + LIMIT 1 | 自动探测 |
| 防护 | 无 | `mysqli_real_escape_string()` | 参数移入 session | 按需绕过 |
| 数据库 | MySQL | MySQL | MySQL | MySQL / SQL Server |

## 使用方式

### 方式一：直接运行 exe（无需安装 Python）

双击 `dist/SQL注入图形化渗透工具.exe` 即可。

### 方式二：源码运行

```bash
pip install -r requirements.txt
python main.py
```

## 默认参数

| 项目 | 值 |
|---|---|
| 目标 URL | `http://192.168.182.131/dvwa/vulnerabilities/sqli/` |

> Cookie、参数名、按钮名、数据库类型均**可选**，留空会自动探测。Cookie 里的 `security` 值可自行改成 `low` / `medium` / `high`，工具会自动适配对应的注入通道与注入点类型。PHPSESSID 若过期需重新登录 DVWA 获取。

## 原理

采用 **UNION 联合注入 + 回显位**：

1. **探测注入通道**：依次尝试 POST → GET → SESSION，判断哪种通道下注入生效
2. **探测闭合方式**：遍历 `'`、`"`、`()`、`((''))`、无闭合（数值型）等，用 `AND 1=1` / `AND 1=2` 判断
3. **order by 测列数**：不断增大 `ORDER BY n` 直到报错，确定原查询列数
4. **union select 找回显位**：`UNION SELECT 1,2,3,...` 定位页面回显位置
5. **回显位提取数据**：
   - 当前库名：`database()`
   - 所有库名：`information_schema.schemata`
   - 表名：`information_schema.tables`
   - 列名：`information_schema.columns`
   - 字段数据：`group_concat` 一次性批量读取

### 各安全级别绕过要点

**low 级别**（`$_GET['id']`，无防护）：直接 GET 传参 + 单引号闭合即可。

**medium 级别**（`$_POST['id']` + `mysqli_real_escape_string()`）：
- 注入点是**数值型**（`WHERE user_id = $id`，无引号包裹），payload 无需引号，转义失效
- 字符串比较用 MySQL 的 `char(ascii,...)` 函数构造（如 `char(100,118,119,97)` = `'dvwa'`），完全避开单引号

**high 级别**（`$_SESSION['id']` + `LIMIT 1` + 隐藏报错）：
- 参数从 session 读取，需**先 POST 到 `session-input.php` 写入 session，再 GET 目标页**触发
- 报错被隐藏为 `Something went wrong.`，需识别该特征判断 `order by` 越界
- `LIMIT 1` 不影响 union 注入（用不存在的 id 让前半段无结果，union 结果直接回显）

相比盲注，UNION 注入把结果直接回显在页面上，速度快、结果稳定。

## 文件结构

```
├── main.py           # PyQt6 图形界面（深色主题）
├── sqli_engine.py    # 注入引擎核心逻辑
├── requirements.txt  # 依赖清单
└── dist/             # 打包好的 exe
```

## 免责声明

本工具仅用于 **授权测试与教学目的**（本课程作业）。请勿对未授权的目标进行测试。
