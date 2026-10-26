# 豆包工作 Agent 跨账号迁移工具（Go）— 设计草案

- 状态：Draft（先落设计，再实现）
- 日期：2026-10-24
- 技术栈：Go 1.26.2（仅标准库；与本机已有 SDK 对齐）
- 目标读者：有 Java 背景、第一次写 Go 的开发者

## 1. 目标与验收

把本地豆包工作中的**一个完整项目**迁移到飞书账号里的豆包工作，迁移对象不是聊天记录，而是：

- Agent 定义：名称、描述、系统提示词、模型、技能、连接器、任务偏好
- 工作流：步骤、触发方式、节点关系
- 对话上下文：历史会话、关键决策、任务状态
- 用户记忆：短期情景记忆 + 长期语义记忆（偏好、性格、画像）
- 项目文件：资料与产物索引
- **运行环境：项目依赖的工具链、系统包与启动方式（如 Rust/Cargo），用 Docker 镜像 + workspace 还原**
- 权限：只记录引用，导入目标账号后重新绑定，**不跨账号复制权限**

验收清单：

1. `dbmig pack` 能把源项目打成加密迁移包 `.dbm`
2. `dbmig inspect` 不输入口令也能看清四层结构与条目数
3. 不带口令 `unpack`：只还原短期/程序/组织引用层，长期语义层保持锁定
4. 带口令 `unpack`：四层完整还原，校验和、签名全部通过
5. `dbmig export` 生成目标账号官方导入入口可识别的本地导出目录
6. 在飞书账号的豆包工作「设置 → 数据导入与同步」中完成导入，人工核对清单通过
7. `dbmig pack --with-env` 能识别 Rust 项目并生成可构建的 Dockerfile / Compose
8. 目标侧一条命令启动容器后能直接执行 Rust 脚本，输出与源项目一致

> 验收优先级（2026-10-25 确认）：先 M-A（Agent 数据/记忆），再 M-B（Rust 环境可运行）；最终两者合并为同一个迁移包。目标平台导入能力待实测，先做本地 Demo。

## 2. 硬约束

- **零封号风险**：只走豆包工作官方 UI / 官方导出 / 官方导入入口。
  - 不重放私有 Web 接口
  - 不逆向客户端协议
  - 不伪造登录态
- **只用 Go 标准库**：`crypto/pbkdf2`、`crypto/hkdf`、`crypto/aes`、`crypto/cipher`、`crypto/hmac`、`crypto/sha256`、`archive/zip`、`encoding/json`
- 跨平台优先 Windows，但代码不依赖 Windows API
- 口令不放在命令行参数里（会进 shell history），使用环境变量 `DBMIG_PASS`

## 3. 记忆分层：Mem0 四层映射

Mem0 的四个生命周期层：Conversation → Session → User → Organizational。
迁移包按下面四层落地：

| 层 | 名称 | 内容 | 生命周期 | 迁移策略 |
|---|---|---|---|---|
| L1 | 情景层 episodic | 历史会话、任务步骤、关键决策、产物索引 | 短期 | 明文打包，无需密钥 |
| L2 | 语义层 semantic | 用户偏好、性格画像、长期事实 | 长期 | **必须密钥**，AES-256-GCM 加密 |
| L3 | 程序层 procedural | Agent 配置、系统提示词、工作流、技能、连接器 | 长期 | 明文 + HMAC 签名，防篡改 |
| L4 | 组织层 org | 企业知识权限、成员、团队可见性 | 组织级 | **不复制**，只保留 ACL 引用，导入后重新绑定 |

作用域（与 Mem0 对齐）：`user_id` / `agent_id` / `run_id` / `app_id`。
每条记忆都带 scope，子密钥按 `layer + scope` 派生，做到“同一口令、不同项目/用户互不相通”。

判定规则（离线 demo 阶段）：

1. 显式 `layer` 字段优先
2. 其次 `type` 字段
3. 最后按关键词兜底：偏好/性格/习惯 → semantic；流程/技能/连接器 → procedural；权限/成员/可见 → org；其余 → episodic

## 4. 迁移包 `.dbm`

`.dbm` 是标准 zip，布局固定：

```
manifest.json                     # 清单：项目元信息、分层、校验和、KDF 参数、策略
episodic/data.json                # L1：sessions + episodic memories（明文）
semantic/memories.jsonl.enc       # L2：语义记忆（AES-256-GCM 密文）
procedural/assets.json            # L3：agent + workflows + procedural memories（明文）
org/acl.json                      # L4：权限引用（copied=false）
compose.yaml                      # 目标侧一键启动环境
env/Dockerfile                    # 运行环境镜像定义
env/entrypoint.sh                 # 容器入口，默认执行项目的启动命令
env/runtime.json                  # 探测结果：语言/版本/系统包/启动命令
env/image.tar                     # 可选：docker save 的离线镜像（--with-image）
files/...                         # 项目文件原样保留，同时作为 Docker 构建上下文
```

`files/` 既是项目文件目录，也是 `compose.yaml` 的构建上下文和工作区挂载点，避免源码重复打包。

`manifest.json` 关键字段：

```json
{
  "format_version": "dbmig/1",
  "bundle_id": "uuid",
  "created_at": "RFC3339",
  "source": {"platform": "doubao-work", "account": "personal"},
  "target": {"platform": "doubao-work-feishu", "account": "feishu"},
  "project": {"id": "...", "name": "...", "owner": "..."},
  "kdf": {"algorithm": "pbkdf2-sha256", "salt": "base64", "iterations": 600000},
  "check_value": "base64(nonce+ciphertext)",
  "layers": [
    {"name": "episodic", "path": "episodic/data.json", "encrypted": false, "entries": 3, "sha256": "..."},
    {"name": "semantic", "path": "semantic/memories.jsonl.enc", "encrypted": true, "entries": 5, "sha256": "..."},
    {"name": "procedural", "path": "procedural/assets.json", "encrypted": false, "entries": 2, "sha256": "...", "hmac_sha256": "..."},
    {"name": "org", "path": "org/acl.json", "encrypted": false, "entries": 3, "sha256": "..."}
  ],
  "files": [{"path": "files/brief.md", "size": 123, "sha256": "..."}],
  "memory_policy": {
    "mapping": "mem0-four-layer",
    "short_term": ["episodic"],
    "long_term": ["semantic"],
    "requires_key": ["semantic"],
    "org_copied": false
  }
}
```

## 5. 密钥体系：分层信封 + scope 派生

```
口令 passphrase
  └─ PBKDF2-SHA256(salt 16B, 600000 次) → KEK 32B
       └─ HKDF-SHA256(info="dbmig/v1|master") → master key 32B
            ├─ HKDF(info="dbmig/v1|layer=semantic|scope=user:<uid>") → L2 子密钥
            ├─ HKDF(info="dbmig/v1|layer=procedural|scope=agent:<aid>") → L3 签名密钥
            ├─ HKDF(info="dbmig/v1|layer=check|scope=") → 口令校验密钥
            └─ 其他层按需派生
```

- 对称加密：AES-256-GCM，12B 随机 nonce，AAD 绑定 `layer|scope|bundle_id`
- 口令校验：manifest 存 `check_value`，解密固定串成功即口令正确
- L3 完整性：HMAC-SHA256（用 procedural 子密钥）；无口令时仍可还原 L3，但签名标记为“未验证”
- L2 无口令时：不写明文，输出 `locked` 状态；只迁短期层
- 升级路径：以后可换 scrypt/Argon2id、或 X25519 收件人加密；manifest 的 `kdf.algorithm` 已留版本位

> 说明：Go 标准库只有 PBKDF2（Go 1.24+），没有 scrypt；换成 scrypt 需要引入 `golang.org/x/crypto`。首版坚持零外部依赖，用 PBKDF2 600k 次。

## 6. CLI 设计

```text
dbmig pack    -src <项目目录> -out <包.dbm>           # 需要 DBMIG_PASS
dbmig inspect -in  <包.dbm>                          # 不需要口令
dbmig unpack  -in  <包.dbm> -out <还原目录>           # 口令可选
dbmig verify  -in  <包.dbm>                          # 口令可选，校验 checksum/HMAC
dbmig export  -in  <包.dbm> -out <导出目录> -target chatgpt|traework
```

退出码：0 成功；1 运行错误；2 用法错误；3 口令错误。

## 7. 数据流

```
源项目（样例 JSON / 后续官方导出）
  → 规范化 SourceProject
  → Mem0 四层分类
  → 分层信封加密
  → .dbm 迁移包
  → inspect / verify
  → export 为目标平台本地导出格式
  → 豆包工作官方「数据导入与同步」UI 导入
  → 目标账号人工验收
```

## 8. 目标侧兼容策略

豆包工作 2026-10-14 上线的导入入口只认 **ChatGPT / Claude Code / Trae Work 的本地导出**，不支持豆包工作账号之间直接导入。

首版 `export -target chatgpt` 生成：

```
chatgpt-export/
├─ conversations.json          # 历史会话（ChatGPT 标准结构）
├─ user.json                   # custom_instructions（来自 L2 语义记忆）
└─ migration_manifest.json     # 补充：Agent/工作流/技能/ACL 引用
```

注意：`migration_manifest.json` 是补充文件，官方导入是否读取需在 UI 实测；如果被忽略，下一迭代改适配 Trae Work 导出结构。
导入完成后必须人工核对：会话标题/上下文、任务偏好、技能、工作记忆、项目文件。

## 9. 实施阶段

| 阶段 | 内容 | 产出 |
|---|---|---|
| D0 | 设计 + git 初始化 + Go 环境 + Go 基础教学 | 本文件、初始提交、可编译 hello |
| D1 | 样例项目 + pack/inspect/verify/unpack | 本地加密闭环 |
| D2 | export 兼容包 + 官方 UI 导入 | 第一条端到端链路 |
| D3 | 接真实源数据 + 目标账号验收 | 满足验收标准 |
| D4 | 加固：脱敏、断点续传、增量同步、审计日志 | 可长期使用 |

## 10. 风险与不做的事

- 官方导入格式未公开文档化：兼容性只能以 UI 实测为准；失败就换 Trae Work 结构
- 本地客户端数据格式未知：D3 前不碰；需要时只做离线副本解析，并先征得同意
- 权限不可跨账号复制：L4 只保留引用，避免越权
- 长期记忆含敏感画像：默认加密、默认不带口令不落盘；不在日志中打印内容
- 不做：私有接口重放、验证码绕过、风控对抗、批量账号操作

## 11. 未来目录规划（D1 再创建）

```text
cmd/dbmig/               CLI 入口
internal/model/          数据模型
internal/memlayer/       四层分类
internal/keys/           KDF / 信封加密
internal/bundle/         pack / unpack / verify
internal/adapter/chatgpt 目标格式导出
testdata/sample_project/ 样例项目
```

## 12. Go 学习路线（对照 Java）

| 课 | 知识点 | Java 对照 |
|---|---|---|
| 1 | package / import / func main / fmt | class + public static void main |
| 2 | var / := / 基本类型 / 零值 | 局部变量与默认值 |
| 3 | 函数、多返回值、error | 方法、异常 |
| 4 | struct / method / 指针接收者 | class / this |
| 5 | slice / map / range | List / HashMap / for-each |
| 6 | interface / 隐式实现 | interface / implements |
| 7 | JSON 序列化 + 文件 IO | Jackson + java.nio |
| 8 | crypto + zip | JCA + ZipOutputStream |
| 9 | go test | JUnit |
| 10 | goroutine / channel（选学） | 线程池 / BlockingQueue |

节奏：一次只讲一个知识点 → 自己写脚本 → 我负责 `go build` / `go run` 检查并指出问题。

## 13. 环境迁移子系统（2026-10-25 增补）

### 13.1 迁移单元

迁移单元 = Agent 数据 + 记忆 + 工作流 + **可运行环境**。环境不是附件，而是验收的一部分。

### 13.2 环境探测

| 源项目特征 | runtime | 基础镜像 |
|---|---|---|
| `rust-toolchain.toml` / `Cargo.toml` | rust-cargo | `rust:<pin>-slim` |
| 只有 `.rs` 文件，无 `Cargo.toml` | rust-single | `rust:<pin>-slim`（`rustc` 编译单文件） |
| `package.json` | node | `node:<pin>-slim` |
| `pyproject.toml` / `requirements.txt` | python | `python:<pin>-slim` |
| 以上都没有 | generic | `debian:stable-slim` |

规则：

- 版本优先来自 `rust-toolchain.toml`、`package.json`、`.python-version`；没有就用稳定版，并写进 `runtime.json`
- 有 lockfile 必须带上：`Cargo.lock`、`package-lock.json`、`poetry.lock`
- 系统包写入 `runtime.json` 的 `apt_packages`，由 Dockerfile 安装
- 探测结果落到 `env/runtime.json`，后续生成 Dockerfile/Compose 只读这份文件

### 13.3 Rust 环境模板

`env/Dockerfile`：

```dockerfile
FROM rust:1.83-slim
WORKDIR /workspace
COPY files/Cargo.toml files/Cargo.lock ./
RUN cargo fetch
COPY files/ ./
RUN cargo build --release
CMD ["cargo", "run", "--release"]
```

`compose.yaml`：

```yaml
services:
  agent:
    build:
      context: .
      dockerfile: env/Dockerfile
    working_dir: /workspace
    volumes:
      - ./files:/workspace
    command: ["cargo", "run", "--release"]
```

说明：

- 镜像负责工具链，`files/` 负责代码；两者都在包里，目标机器不需要预装 Rust
- 代码挂载进容器，迁移后仍可直接改脚本，不用每次重建镜像
- `--with-image` 时额外执行 `docker save`，生成 `env/image.tar`，支持离线导入

只有单文件、没有 `Cargo.toml` 时，`runtime.json` 标记 `"mode": "single_file"`，Dockerfile 改为：

```dockerfile
FROM rust:1.83-slim
WORKDIR /workspace
COPY files/ ./
RUN rustc main.rs -o /usr/local/bin/app
CMD ["app"]
```

对应 Compose 的 `command` 为 `["app"]`。`main.rs` 的实际文件名以探测结果为准，不写死。

### 13.4 CLI 增补

```text
dbmig env detect -src <项目目录>                 # 只探测，输出 runtime.json
dbmig env build  -src <项目目录> -tag <tag>      # 构建镜像 + 本地 smoke test
dbmig pack       -src <项目目录> -out <包.dbm> [--with-env] [--with-image]
dbmig env run    -in <包.dbm> -out <目录>        # unpack + compose build + 运行验收脚本
```

退出码补充：`4` 环境构建失败；`5` smoke test 失败。

### 13.5 数据流

```text
源项目
 ├─ Agent 数据 → 四层分类 → 加密 → .dbm
 └─ 环境探测 → runtime.json → Dockerfile/Compose → build → smoke test
                                             └─ 可选 docker save → env/image.tar
                    ↓
                 .dbm 迁移包
                    ↓
目标侧 unpack → docker compose build/up → 运行 Rust 脚本 → 输出比对
```

### 13.6 Rust 验收用例

当前源项目还没有 Rust 文件，验收时现场配置。工具不预先假设路径，现场执行 `env detect`。

两种样例：

A. Cargo 项目

```text
rust-toolchain.toml
Cargo.toml
Cargo.lock
src/main.rs        # 输出 hello from rust
```

B. 单文件

```text
main.rs            # 输出 hello from rust
```

验收步骤（两种模式相同）：

1. 现场在源项目目录执行 `dbmig env detect -src <目录>`，确认 `runtime.json` 的 `runtime` 为 `rust-cargo` 或 `rust-single`
2. `dbmig pack -src <目录> -out rust.dbm --with-env --with-image`
3. 目标机器只装 Docker，不装 Rust
4. `dbmig env run -in rust.dbm -out restored`
5. 输出 `hello from rust`，退出码 0

判定：源项目和目标容器输出逐字节一致，即验收通过；Cargo 和单文件各跑一次。

### 13.7 密钥与 secret 边界

- **secret 不进镜像层**：Docker 层会保留历史，`ENV` / `COPY` 的密钥无法真正删除
- 运行期 secret 走 `--env-file`，或加密后放进 `env/secrets.enc`，由 `dbmig` 解密后注入
- `.dockerignore` 排除 `.env`、`target/`、`*.pem`、`*.key`
- 长期记忆仍按第 5 节信封加密；环境 secret 复用同一 KEK，但使用独立子密钥

### 13.8 里程碑拆分

| 里程碑 | 内容 | 验收 |
|---|---|---|
| M-A（先做） | Agent 数据/四层记忆 pack/inspect/unpack | 带口令完整还原 |
| M-B | Rust 环境探测、Dockerfile 生成、镜像构建、目标侧运行 | Cargo 与单文件均可运行 |
| M-C | 合并成一个 `.dbm` + 官方导入 | 目标账号人工验收 |

先做 M-A 的原因：当前没有现成的 Rust 源项目，环境部分等验收时现场配置；先把不依赖 Docker 的数据/记忆闭环跑通。目标平台的镜像导入能力未知，M-B 先在本地 Docker 验证，平台实测放到 M-C。

### 13.9 目录规划增补

```text
internal/runtime/        语言与版本探测
internal/docker/         Dockerfile/Compose 生成 + docker CLI 调用
internal/envbundle/      环境打包、smoke test、镜像导入导出
templates/               Dockerfile / compose 模板
testdata/sample_rust/    Rust 验收样例
```

### 13.10 学习路线调整

调整后的顺序：1-8 不变，第 9 课仍是 `go test`；新增第 10 课，原第 10 课顺延。

| 课 | 内容 | 用途 |
|---|---|---|
| 9 | `go test` | 给探测和打包逻辑写测试 |
| 10 | `os/exec` + `flag` | 调用 `docker build/save/load/run`，解析命令行 |
| 11 | `goroutine/channel`（选学） | 并行打包 / 进度输出 |

Docker 基础单独做一次实战，不计入 Go 课：只学 `Dockerfile`、`compose.yaml`、`docker build/run/save/load` 五个点。

### 13.11 已确认与待实测

已确认（2026-10-25）：

- 第一版先做 M-A（Agent 数据/记忆），M-B 环境迁移后置
- 源项目当前没有 Rust 文件，验收时现场配置；工具只负责现场探测，不预置样例
- Rust 验收同时覆盖 Cargo 项目和单文件脚本
- 目标平台导入能力未知，先做本地 Docker Demo

待实测：

1. 飞书账号里的豆包工作是否支持导入自定义 Docker 镜像或环境模板？
2. 目标机器/沙箱是否允许运行 Docker？
3. 迁移包是否有大小限制，是否需要默认不打包镜像、只传 Dockerfile？

## 14. 隐私标签与完整性校验（2026-10-26 增补）

### 14.1 参考边界

蚂蚁阿福的公开材料只披露控制面，没有披露“记忆写入时打标签”的内部实现。可确认的是：

- 记忆是结构化存储，用户可在【记忆】中查看、管理，授权后用于个性化回复
- 健康生理信息属于敏感个人信息，需要单独同意
- “无痕拍/无痕模式”下照片打码，对话结束自动清除，不进历史记录
- 信通院“卓越型”认证强调全链路防护与查询/更正/删除权利

因此本工具不假设源项目一定带隐私标签，必须同时支持：**显式标签、策略默认、检测器推断、完全无标签**。

### 14.2 PrivacyEnvelope

每条记录（对话、记忆、工作流、文件）外层统一包一个隐私信封：

```go
type Level int

const (
	Unknown Level = iota
	Public
	Internal
	Sensitive
	Restricted
)

type Verdict struct {
	Level      Level
	Confidence float64
	Source     string // explicit / policy / detector
	Version    string
	Evidence   string
}

type PrivacyEnvelope struct {
	Level      Level     // 最终生效等级
	Declared   Level     // 源记录自带的等级，仅作参考
	Verdicts   []Verdict // 所有来源的裁决记录
	Ephemeral  bool      // 无痕记录，默认不导出
	Quarantine bool      // 未知等级，进入隔离区
}
```

裁决规则：

1. 显式标签作为 `Declared`，不无条件信任
2. 依次运行策略默认值和检测器，取**最高等级**，失败关闭
3. 显式低等级与检测高等级冲突时，按高等级处理并记录冲突
4. 无标签且无可靠推断 → `Unknown` → `Quarantine`，默认不迁移
5. `Ephemeral` 走单独路径，默认不打包；需要导出时必须显式确认
6. **目标侧重新评估**：源标签不可信，目标策略更严时以目标为准
7. `Sensitive` / `Restricted` 的长期记忆必须同时具备密钥和显式同意

### 14.3 可插拔 Gate

```go
type Tagger interface {
	Name() string
	Tag(ctx context.Context, r Record) (Verdict, error)
}

type Gate struct {
	Taggers           []Tagger
	Default           Level
	QuarantineUnknown bool
}
```

- 具体用哪些 Tagger 由配置决定，不在代码里写死
- 每个裁决记录 `tagger 名称 + 版本 + 置信度 + 证据`，保证结果可复现
- 需要降级时生成待审批项，不静默降级
- 隔离区输出 `quarantine/` 和清单，用户确认后再进入迁移流程

### 14.4 完整性通道

“逐字节一致”只对没有被解码/重编码的数据成立。清单中每个条目必须声明校验通道：

| 通道 | 适用对象 | 校验方式 | 能否逐字节 |
|---|---|---|---|
| `raw` | 附件、Rust 源码、原始 JSON blob、Dockerfile/compose/lockfile | `SHA-256(raw bytes)`，源/目标对比 | **能** |
| `canonical-json` | 需要转成目标 schema 的结构化记录、对话、记忆条目 | RFC 8785 JSON 规范化后 `SHA-256` | 不能，承诺规范化等价 |
| `semantic` | 目标会改写、摘要、重排的内容 | 字段断言 + 抽样比对 | 不能，只承诺语义 |

补充规则：

- 加密数据比**明文哈希**，不比密文；记录 `plaintext_sha256`、`key_id`、`cipher_suite`、`aad_hash`
- Docker 环境不承诺镜像字节一致：`Dockerfile/compose/lockfile` 走 `raw`，镜像比 `config digest` 与 `diffID`，最终用 `rustc --version` 和脚本输出哈希验收
- 每个条目记录 `hash_mode`、`source_sha256`、`target_sha256`、`status`
- 所有条目哈希排序后计算 Merkle root，再用迁移密钥做 HMAC 签名，防篡改
- `verify` 必须独立读取目标侧并重算，不能只信任导出侧写入的哈希

### 14.5 manifest 增补字段

第 4 节的 `manifest.json` 增加：

```json
{
  "privacy_policy": {
    "levels": ["public", "internal", "sensitive", "restricted"],
    "unknown_action": "quarantine",
    "ephemeral_export": false
  },
  "items": [
    {
      "id": "mem-001",
      "hash_mode": "canonical-json",
      "source_sha256": "...",
      "target_sha256": "...",
      "status": "matched",
      "privacy": {
        "level": "sensitive",
        "declared": "internal",
        "ephemeral": false,
        "sources": ["explicit", "detector:pii"]
      }
    }
  ],
  "merkle_root": "...",
  "manifest_hmac": "..."
}
```

### 14.6 验收报告

`verify` 输出必须分通道统计，不能只给一个总数：

```text
raw:        128 items, 128 matched, 100%
canonical:  942 items, 942 matched, 100%
semantic:    37 items,  35 passed, 2 need review
quarantine:  11 items, not migrated (unknown label)
```

验收结论明确写成：**raw 通道逐字节一致；canonical 通道规范化等价；semantic 通道语义一致；隔离区未迁移。** 不把“语义一致”包装成“逐字节一致”。

### 14.7 目录增补

```text
internal/privacy/    Tagger 实现、Gate、策略配置
internal/integrity/  哈希、规范化 JSON、Merkle、清单签名
internal/verify/     独立读取目标侧并重算
```

M-A 阶段先落 `raw` + `canonical-json` 两个通道；`semantic` 和目标侧复评在 M-C 接入官方导入后补齐。

