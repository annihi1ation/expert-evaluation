# 专家评分系统 · 本地运行

一个专家评分表单（`Expert Evaluation Form B (standalone).html`），在 localhost 上运行，
评分直接写入 Google Sheet（通过 Google Apps Script Web App）。

## 组成

| 文件 | 作用 |
|------|------|
| `Expert Evaluation Form B (standalone).html` | 前端（已内置样本数据、评分逻辑、Google Sheet 同步）。**无需修改。** |
| `serve.py` | 极小的本地静态服务器（Python 标准库，零依赖），把表单跑在 http 源上。 |
| `Code.gs` | Google Apps Script 后端（粘贴到与 Sheet 绑定的 Apps Script 项目）。 |
| `README.md` | 本文件。 |

数据流：`浏览器 (localhost) → Google Apps Script /exec → Google Sheet「Responses」`。
评分不经过本地，直接由浏览器发往 Google。

## 线上部署（GitHub Pages）

两个表单已发布到同一个 GitHub Pages 站点，除「基线报告」内容与 `formGroup` 外完全相同；
两者的 my_rag_v2 报告（被评系统）一致，只有对照的基线不同（见 `assignment_record.csv`）。

| 表单 | 文件 | `formGroup` | 对比（Report A/B 之一为基线） | 链接 |
|------|------|-------------|------------------------------|------|
| form_2 | `index.html` | `group2` | my_rag_v2 **vs vanilla_rag** | <https://annihi1ation.github.io/expert-evaluation/> |
| form_1 | `form1.html` | `group1` | my_rag_v2 **vs kg_rag** | <https://annihi1ation.github.io/expert-evaluation/form1.html> |

两个表单的评分按 `formGroup` 分桶写入同一个 Sheet（`group1` / `group2` 互不覆盖），后端 `Code.gs` 无需改动。
`form1.html` 由 `index.html` 派生：仅把每位 client 的基线一侧报告从 `*_a2/_b2`（vanilla_rag）替换为
`*_a1/_b1`（kg_rag），并将 `formGroup` 改为 `group1`；其余（client 档案、被评报告、UI）逐字节保持不变。

## 快速开始

```bash
cd 解压后的目录          # 例如: cd expert-evaluation-localhost
python3 serve.py
```

浏览器会自动打开 `http://localhost:8000/`。输入姓名和邮箱 → 选择一位 patient → 打分，
右上角状态会从 `Saving…` 变成 `Synced`，同时对应行会出现在 Google Sheet 里。

---

## 部署（localhost）

### 前置条件
- **Python 3**（3.6+，标准库即可，无需 `pip install`）。
  - macOS / Linux 一般自带：`python3 --version` 应能输出版本号。
  - Windows：到 <https://www.python.org/downloads/> 安装，安装时勾选 **Add Python to PATH**；
    命令用 `python` 而不是 `python3`（即 `python serve.py`）。
- 一个现代浏览器（Chrome / Edge / Safari / Firefox）。
- **联网**（评分要发往 Google，需要能访问 `script.google.com`）。

### 方式一：单机 localhost（最常见）
把整个文件夹拷到目标机器，然后：

```bash
cd expert-evaluation-localhost      # 进入解压后的目录
python3 serve.py                    # 默认 127.0.0.1:8000，只允许本机访问
```

看到 `本机访问: http://localhost:8000/` 即成功；浏览器会自动打开。
**换端口**（8000 被占用时）：`python3 serve.py 8080`。

> 不想用 `serve.py` 也行（等价的一行）：在该目录执行 `python3 -m http.server 8000`，
> 然后手动打开 `http://localhost:8000/Expert%20Evaluation%20Form%20B%20(standalone).html`。

### 方式二：一台机器托管，多位专家从各自电脑访问（局域网）
适合多人同时评分：只在一台机器上跑服务，其它人用浏览器连过来。

```bash
python3 serve.py 8000 0.0.0.0       # 绑定所有网卡
# 或： HOST=0.0.0.0 python3 serve.py
```

启动时会打印 `局域网访问: http://<本机IP>:8000/`，把这个地址发给各位专家即可。
注意事项：
- 需在托管机器的**防火墙**放行该端口（macOS：系统设置→网络→防火墙；Windows：入站规则放行 Python）。
- 各专家在页面里填**各自的邮箱**——数据按 `邮箱 + patient` 分行存储，互不覆盖。
- 这是明文 http、无鉴权，仅建议在可信内网使用；不要直接暴露到公网。

### 让服务常驻后台（可选）
默认 `python3 serve.py` 占用当前终端，关掉终端就停。要长期运行：

```bash
# macOS / Linux：后台运行并写日志
nohup python3 serve.py 8000 0.0.0.0 > serve.log 2>&1 &
echo $!            # 记下进程号，方便以后 kill

# 停止
kill <进程号>       # 或： pkill -f serve.py
```

### 停止 / 重启
- 前台运行时：在终端按 **Ctrl+C**。
- 后台运行时：`pkill -f serve.py`（或 `kill <进程号>`），再重新 `python3 serve.py` 启动。

### 常见问题
| 现象 | 处理 |
|------|------|
| `Address already in use` / 端口被占 | 换端口：`python3 serve.py 8080`；或先 `pkill -f serve.py`。 |
| `command not found: python3` | 装 Python 3；Windows 上改用 `python serve.py`。 |
| 找不到表单文件 | 必须在**与 `serve.py` 同一目录**运行（HTML 要和脚本在一起）。 |
| 右上角一直不显示 `Synced` | 检查是否联网、能否访问 `script.google.com`；见下方「验证」自查。 |
| 局域网别的电脑连不上 | 确认用了 `0.0.0.0` 启动，且托管机防火墙放行了该端口，双方在同一网段。 |

## Google Sheet 后端

前端里的 `this.SHEETS_ENDPOINT` 已经指向线上的 `/exec` 部署，`this.formGroup = 'group2'`。
`Code.gs` 就是该端点应有的逻辑：

- `doPost(e)`：解析 JSON，按 `group + email + patientId` **去重覆盖**写入 `Responses` 表。
- `doGet(e)`：`?type=load&email=&patientId=&group=` 返回 `{payload:{...}}`，裸 GET 返回健康检查。
- 表头：`ts, group, email, name, patientId, patientName, origLabel, progress, submitted, payload(JSON)`。

### 若需要（重新）部署 Code.gs —— 务必保持同一个 /exec URL

1. 打开目标 Google Sheet → **扩展程序 → Apps Script**。
2. 把 `Code.gs` 全量粘贴进去，保存。
3. **部署 → 管理部署 → 编辑（铅笔图标）现有部署 → 版本选“新版本” → 部署。**
   - 执行身份 = 我；有权访问 = 任何人。
   - **一定要“编辑现有部署”，不要“新建部署”** —— 新建会生成不同的 URL，届时得同步修改
     前端 `Expert Evaluation Form B (standalone).html` 里的 `this.SHEETS_ENDPOINT`。

> 想保留每次同步的完整历史（而非每人每 patient 一行）：把 `Code.gs` 里 `saveRecord_`
> 的覆盖分支去掉，始终 `appendRow`；`findRowIndex_` 已是“取最近一条”，load 仍然正确。

## 验证（端到端）

端点常量（下文用 `$EXEC` 代替）：
```
https://script.google.com/macros/s/AKfycbwFri9S3NCFu3U31uogEbgJJhcg1TIZf7EeviO63d4HVzGtXs96mWTwDXtNMzjnO2bw/exec
```

1. **探活（读）**
   ```bash
   curl -sSL "$EXEC?type=load&email=probe@example.com&patientId=p1&group=group2"
   # 期望: {"payload":null}
   ```
2. **测保存（写一条测试数据）**
   ```bash
   curl -sSL -X POST -H "Content-Type: text/plain" \
     -d '{"type":"save","formGroup":"group2","expert":{"name":"T","email":"probe@example.com"},"patientId":"p1","patientName":"11","progress":"1/8","submitted":false,"payload":{"idx":0,"answers":{"d0":4},"comments":{},"flags":{},"flagText":{},"submitted":false},"ts":"2026-07-01T00:00:00Z"}' \
     "$EXEC"
   ```
   然后到 Google Sheet 看 `Responses` 是否出现/更新了 `probe@example.com / p1` 这一行，
   且 `payload` 含 `d0:4`；再跑第 1 步的 load，应返回该 payload（不再是 null）。
3. **界面验证**：`python3 serve.py` → 输入姓名/邮箱 → 选 patient → 打分 → 状态变 `Synced` → 表格出现对应行。
4. **回填验证**：清掉浏览器 localStorage（或换个浏览器）后重进同一 patient，评分应从表格 rehydrate 回来。
5. **清理**：删掉表格里 `probe@example.com` 的测试行。

## 说明 / 已知限制

- 保存用 `fetch(..., {mode:'no-cors'})`，前端读不到响应，只按“发出即成功”处理。
  所以 save 是否真的成功，只能看 Google Sheet 是否落行（验证第 2、3 步）。
- 加载是跨域 GET，依赖 Apps Script 对 GET 返回的宽松 CORS；即使偶发失败，localStorage
  仍是本地主存储，不影响记录本身。
- 样本数据已内嵌在 HTML 中，`client_*` 目录下的 markdown 与 `assignment_record.csv` 仅作参考，运行时不需要。
