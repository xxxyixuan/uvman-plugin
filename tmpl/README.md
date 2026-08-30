# uvman 插件编写说明

> 对应代码：`src/core/plugin.rs`、`src/toolset/mod.rs`
> 适用范围：uvman 0.2.x（当前实现）。所有工具（Node、Python 等）统一用一份 TOML 插件文件描述，无需工具专属代码。

## 1. 概览

每个工具对应 `<tool>.toml` 一份插件文件，安装路径为 `uvman/plugins/<tool>.toml`。插件文件只描述「怎么下载、校验、解压、部署一个工具」，不包含任何业务逻辑。

一份完整插件由 5 个区块构成，其中 4 个必填、1 个可选：

| 区块           | 必填 | 作用                             |
|--------------|----|--------------------------------|
| `[tool]`     | ✅  | 工具元信息（名称、描述、插件版本、作者）           |
| `[registry]` | ✅  | 下载源（默认源 + 镜像）                  |
| `[release]`  | ✅  | 版本列表获取方式（API 拉取 / 静态列表）        |
| `[platform]` | ❌  | 系统常量 → 下载 token 映射（缺省则不支持任何平台） |
| `[install]`  | ✅  | 安装方案（默认版本 + 二进制安装项）            |

### 未实现字段（不要写）

以下字段出自早期设计稿，当前代码**不解析**，写了也会被 serde 静默忽略（不报错但完全不生效），请一律删除：

- `tool.homepage` / `tool.license` / `tool.aliases`
- `install.defaults.mode`（bin/src 双模式）
- `install.bin.deploy.copy_extra` / `post_install`
- 整个 `[[install.src]]`（含 `dependencies` / `build` / `download` / `extract` / `deploy`）
- 模板变量 `{install_root}`

这些属于 0.6.0 待定稿范围，届时会重新设计，勿在现版本使用。

## 2. `[tool]` 工具元信息

```toml
[tool]
name = "node"                      # [必填] str，工具唯一标识；安装/切换/查询均以此为准
description = "..."                # [可选] str
version = "1.0.0"                  # [可选] str，插件自身版本，plugin info 展示
author = ["yixuan", "alice"]       # [可选] str 数组
```

注意 `author` 是字符串数组，即使单作者也要写成 `author = ["yixuan"]`。

## 3. `[registry]` 下载源

```toml
[registry]
default = "https://nodejs.org/dist"    # [必填] str
mirrors = ["https://npmmirror.com/..."] # [可选] str 数组
```

**下载候选顺序（依次尝试，成功后停止）：**

1. 全局配置 `[registry]` 中同名 tool 的镜像（优先级最高，用户可覆盖）
2. 插件 `mirrors`
3. 插件 `default`（兜底）

模板变量 `{registry}` 依次替换为上述每个源，生成对应 URL。

## 4. `[release]` 版本列表

用 `source` 字段切换两种变体，二者互斥：

### 4.1 `source = "api"`：拉取远程 JSON

```toml
[release]
source = "api"
url = "https://nodejs.org/dist/index.json"   # [必填] JSON 接口
version_path = "data.tags"                    # [可选] 版本号在 JSON 中的路径（点分路径）
version_pattern = '^v(.*)$'                   # [可选] 从版本串提取真实版本的正则
```

- `version_path`：若接口返回嵌套 JSON，用点分路径定位到版本号数组，如 `"data.tags"`；不写则取 JSON 根。
- `version_pattern`：对每个候选版本串应用正则，取**第一个捕获组**作为版本号；常用去前缀（`^v(.*)$`）。

### 4.2 `source = "static"`：固定版本列表

```toml
[release]
source = "static"
versions = ["20.11.0", "20.10.0"]   # [必填] str 数组
```

适合没有规范版本接口的工具。参考实现：`python.toml`（static，官方源无版本接口）；对照 `node.toml`（api）。

**注意**：`source` 必须是 `"api"` 或 `"static"`，其他值会导致整份插件解析失败（不是静默忽略）。

## 5. `[platform]` 平台映射（可选）

```toml
[platform]
os_map = { windows = "win", linux = "linux", macos = "darwin" }
arch_map = { x86_64 = "x64", aarch64 = "arm64" }
```

- `os_map`：系统 OS 常量 → 下载 URL 里的 OS 标识。
- `arch_map`：系统 ARCH 常量 → 下载 URL 里的架构标识。

运行时把系统常量经映射得到 `{os}` / `{arch}` token。若整个 `[platform]` 缺失，或当前系统的 OS/ARCH 不在映射内，安装会报
`PlatformNotSupported`，**不会**静默回退。

系统常量词汇表：`env::consts::OS` → `windows / linux / macos`；`env::consts::ARCH` → `x86_64 / aarch64`。

## 6. `[install]` 安装方案

### 6.1 默认版本

```toml
[install.defaults]
version = "latest"   # [必填] 用户执行 uvman install node（不带版本）时的默认版本
```

可用 `latest`；也可写具体版本号。

`latest` 取版本集合中**最大的 semver**（semver 优先级忽略 `+` 后的 build metadata，即 `3.12.14+20260825` 与
`3.12.14+20260101` 视为相等，避免在 static 列表里放入仅 build metadata 不同的两个版本）。`3.12`、`3.12.14` 等
部分版本请求按版本串**前缀匹配**后取最大；请求完整版本号时必须能通过 semver 解析（含 build metadata 合法）才按原样使用。

### 6.2 二进制安装项 `[[install.bin]]`

可含多个 `[[install.bin]]`（数组表格），每个描述一套「平台下载/校验/解压/部署」组合：

```toml
[[install.bin]]
os = ["windows", "linux", "macos"]   # [必填] 适用系统
arch = ["x86_64", "aarch64"]         # [必填] 适用架构
```

- `os` / `arch` 必须**精确等于**系统常量字面值（`windows / linux / macos`、`x86_64 / aarch64`），当前代码**不支持** `"all"`
  通配、也不做前缀匹配。要覆盖多个平台就把它们逐个列全。

**下载** `[install.bin.download]`：

```toml
[install.bin.download]
path = "{registry}/v{version}/node-v{version}-{os}-{arch}.{ext}"  # [必填] URL 模板
[install.bin.download.ext]
windows = "zip"      # OS → 扩展名
linux = "tar.gz"
macos = "tar.gz"
```

**哈希校验** `[install.bin.download.hash]`：

```toml
[install.bin.download.hash]
enabled = true                          # [必填] 是否校验
algorithm = "sha256"                    # [可选] 缺省 sha256
path = "{registry}/v{version}/SHASUMS256.txt"  # [可选] 校验和文件 URL 模板
pattern = '^(?P<hash>[0-9a-f]{64})\s+.*node-v{version}-{os}-{arch}.{ext}$'  # [可选]
```

- 不写 `pattern` 时：按官方文件 `{filename}`（自动取自 URL 末段）行内匹配 `<hash>  <文件名>` 格式的行
  （文件名会做正则转义，含 `+` 等特殊字符也安全）；匹配不到时兜底取文件中第一段 64/128 位 hex。
  适合 sha256sum 风格的聚合校验和文件（如 python-build-standalone 的 `SHA256SUMS`）。
- 写 `pattern` 时：用命名捕获组 `(?P<hash>...)`（或第一个捕获组）提取哈希。
  ⚠ 变量插值是**纯文本替换、不做正则转义**：版本串含 `+` 等正则特殊字符时不要把 `{version}` 插进
  pattern（`+` 会变成量词导致匹配错乱），此时应改用上面的文件名自动匹配。
- 校验和文件 URL 同样支持所有模板变量（含 `{filename}`，可拼 sidecar 文件如 `{filename}.sha256`）。

**解压** `[install.bin.extract]`：

```toml
[install.bin.extract]
strip = 1   # [必填] 剥离顶层目录层数；压缩包无顶层目录时填 0
```

**部署** `[install.bin.deploy]`：

```toml
[install.bin.deploy]
bin_dir = { windows = "", linux = "bin", macos = "bin" }
```

`bin_dir` 是「可执行文件相对于解压根目录的路径」（Windows zip 常把二进制放根目录 → 空串；Linux/macOS tar 常放 `bin/`）。

`bin_dir` 支持两种写法（按 OS 区分或全平台统一）：

| 写法      | 示例                                          | 适用场景      |
|---------|---------------------------------------------|-----------|
| 全平台统一   | `bin_dir = "bin"`                           | 各平台目录布局一致 |
| 按 OS 区分 | `bin_dir = { linux = "bin", windows = "" }` | 各平台布局不同   |

**按 OS 区分写法下，当前平台缺失对应键会报错**（不是静默回退），以在安装前暴露布局配置错误。

## 7. 模板变量

可在 `download.path`、`hash.path`、`hash.pattern` 等模板字符串中使用（`{key}` 占位替换）：

| 变量           | 含义                               | 示例值                         |
|--------------|----------------------------------|-----------------------------|
| `{registry}` | 当前下载源（按候选顺序逐个替换）                 | `https://nodejs.org/dist`   |
| `{version}`  | 已解析的具体版本号（非别名）                   | `20.11.0`                   |
| `{os}`       | 经 `[platform].os_map` 映射后的 OS 标识 | `win` / `darwin`            |
| `{arch}`     | 经 `[platform].arch_map` 映射后的架构标识 | `x64` / `arm64`             |
| `{ext}`      | 当前平台的扩展名                         | `zip` / `tar.gz`            |
| `{filename}` | 官方归档文件名（自动取自 URL 末段）             | `node-v20.11.0-win-x64.zip` |

> `{filename}` 由 download URL 末段推导，仅可用于 `hash.path` / `hash.pattern`，**不能**用于 `download.path`
> （URL 尚未生成，占位符会被原样保留）。未知占位符一律原样保留，不会报错。
>
> `{install_root}` 未实现，不要使用。

## 8. 平台决策的误区提醒

- **`ext` 不是模板变量错误**：`download.ext` 是一个 `OS → 扩展名` 的映射表，`{ext}` 变量由它按当前平台解析后注入模板。
- **`bin_dir` 与 `ext` 都是按 OS 表达**：同一工具不同平台布局不同（Node 的 Windows zip 二进制在根目录、linux/macos 在
  `bin/`），因此这两个字段都需要区分 OS。
- **缺失报错优于静默回退**：平台映射、Per-OS `bin_dir` 缺键时都直接报错，帮助在安装前发现问题。

## 9. 快速自检清单

编写完插件后，逐一确认：

- [ ] 5 个区块齐全，`[tool] / [registry] / [release] / [install]` 必填，`[platform]` 可选
- [ ] `release.source` 是 `"api"` 或 `"static"`，没有拼写错误
- [ ] 没有残留未实现字段（aliases / mode / src / copy_extra / post_install / homepage / license / `{install_root}`）
- [ ] `download.ext`、`bin_dir` 覆盖了你声明的所有 `os`
- [ ] `download.path` 里用到的 `{os} / {arch} / {ext}` 在 `os_map / arch_map / ext` 中都有对应键
- [ ] `hash.enabled = false` 时 `pattern`/`path` 可省略；`true` 时确认 `algorithm` 与校验和文件格式匹配
- [ ] `hash.pattern` 里没有插值含正则特殊字符的 `{version}`（如 `+`）；拿不准就不写 pattern 走文件名自动匹配
- [ ] 可用 `uvman plugin install mytool --path ./mytool.toml` 本地安装验证，能成功 `uvman install mytool`