# uvman-plugin

`uvman` 项目的插件仓库。仓库内每一个根目录下的 `*.toml` 文件就是一个 `uvman` 工具插件，用一份纯 TOML
声明某个命令行工具的下载源、版本获取方式、平台映射与二进制安装方案，供 `uvman` 运行时读取并执行安装（对应 uvman 0.2.x 的插件
Schema）。

## 仓库结构

```
uvman-plugin/
├── README.md                 # 本说明文档
├── LICENSE                   # MIT 许可证
├── node.toml                 # Node.js 插件（可作为真实插件安装）
└── templates/
    ├── template.toml         # 插件编写模板（含全部可解析字段与注释）
    └── 插件编写说明.md         # 插件 Schema 完整说明（权威文档）
```

约定：

- 每个真插件以独立 `<tool>.toml` 存放于仓库**根目录**，文件名与 `[tool] name` 保持一致（如 `node.toml`）。
- `uvman plugin list --remote` 会识别仓库**根目录**下的所有 `*.toml` 并展示给用户。因此 `templates/` 子目录下的
  `template.toml` **不会**被当作真实插件展示——它只是编写参考，模板里的 `name = "node"` 仅是示例占位，不可直接安装。
- 所有插件与文档均以 MIT 许可证发布。

> 编写新插件：复制 [`templates/template.toml`](./templates/template.toml) 并按 `[tool] name` 重命名放到根目录，
> 对照 [`templates/插件编写说明.md`](./templates/插件编写说明.md) 填写即可。

## 一个插件由 5 个区块组成

在 `[tool]` / `[registry]` / `[release]` / `[install]`（必填）之外，`[platform]` 可选。完整字段说明见
[`插件编写说明.md`](./templates/插件编写说明.md)，此处仅列概况：

| 区块           | 作用                                                                                                       |
|--------------|----------------------------------------------------------------------------------------------------------|
| `[tool]`     | 工具元信息：`name`（必填）、`description` / `version` / `author`（可选）                                                |
| `[registry]` | 下载源：`default`（必填）与 `mirrors`（可选）；候选顺序为 全局镜像 → 插件 mirrors → 插件 default                                    |
| `[release]`  | 版本列表：`source` 为 `"api"`（拉 JSON，配合 `url` / `version_path` / `version_pattern`）或 `"static"`（固定 `versions`） |
| `[platform]` | 系统 OS/ARCH 常量到下载标识的映射（`os_map` / `arch_map`）；缺失或未覆盖当前平台则安装报不支持，绝不静默回退                                    |
| `[install]`  | `[install.defaults]` 默认版本；`[[install.bin]]` 定义下载 / 校验 / 解压 / 部署流程                                        |

速览示例（Node.js）：

```toml
[tool]
name = "node"
description = "Node.js JavaScript runtime"

[registry]
default = "https://nodejs.org/dist"
mirrors = ["https://npmmirror.com/mirrors/node"]

[release]
source = "api"
url = "https://nodejs.org/dist/index.json"
version_pattern = '^v(.*)$'

[platform]
os_map = { windows = "win", linux = "linux", macos = "darwin" }
arch_map = { x86_64 = "x64", aarch64 = "arm64" }

[install.defaults]
version = "latest"

[[install.bin]]
os = ["windows", "linux", "macos"]
arch = ["x86_64", "aarch64"]

[install.bin.download]
path = "{registry}/v{version}/node-v{version}-{os}-{arch}.{ext}"
[install.bin.download.ext]
windows = "zip"
linux = "tar.gz"
macos = "tar.gz"
```

### 模板变量

下载路径、校验和路径等模板字符串支持以下占位符：

| 变量           | 含义                    | 示例值                         |
|--------------|-----------------------|-----------------------------|
| `{registry}` | 当前生效的下载源              | `https://nodejs.org/dist`   |
| `{version}`  | 已解析的具体版本号             | `20.11.0`                   |
| `{os}`       | 经 `os_map` 映射后的 OS 标识 | `win` / `darwin`            |
| `{arch}`     | 经 `arch_map` 映射后的架构标识 | `x64` / `arm64`             |
| `{ext}`      | 当前平台扩展名               | `zip` / `tar.gz`            |
| `{filename}` | 官方归档文件名（仅校验正则内自动匹配）   | `node-v20.11.0-win-x64.zip` |

## 贡献新插件

1. 复制 [`templates/template.toml`](./templates/template.toml) 为根目录 `<tool>.toml`，文件名与 `[tool] name` 一致。
2. 对照 [`插件编写说明.md`](./templates/插件编写说明.md) 填写 5 个区块，务必移除所有未实现字段
   （`homepage` / `license` / `aliases` / `mode` / `install.src` / `copy_extra` / `post_install` / `{install_root}`）。
3. 用 `uvman plugin install <tool> --path ./<tool>.toml` 本地验证可成功 `uvman install <tool>`。
4. 提交 Pull Request。

## 许可证

本仓库所有插件与文档均遵循 [MIT License](./LICENSE)。