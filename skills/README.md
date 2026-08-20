# lilac Skill（AI 辅助打包配置）

单个 skill 覆盖 lilac 打包配置的**编写**与**运行理解**：

```
skills/
├── README.md                  # 本文件：定位、用法、维护说明
└── lilac/                     # 唯一 skill：配置生成/维护 + 运行原理 + 速查
    ├── SKILL.md               # 运行原理（触发/构建管线/沙箱）+ BuildReason 诊断 + 配置生成工作流 0→5 + 工程实战要点 + 常见运行时故障 + 自更新 SOP
    └── references/
        └── config.md          # 字段契约(§1) + API(§2) + 模块变量(§3) + update_on 全 source/别名/选型(§4) + 仓库级配置(§5) + 自纠错表(§6) + 故障速查(§7) + 扩展 SOP(§8)
```

## 用法

| 场景 | 做法 |
|---|---|
| 「根据 PKGBUILD 生成 / 修正 lilac.yaml」 | 加载 `lilac`，走 SKILL 步骤 0→5 |
| 「update_on 某个 source 怎么写 / tag 匹配方式选型」 | `lilac` → references §4（含 §4.1 tag 策略、§4.2 source 全量+别名目录、§4.4 选型表、§4.5 update_on_build、§4.6 多 source/split、§4.7 soname 规范） |
| 「依赖解释器/库，alias 怎么配」 | `lilac` → references §4.2 别名目录 + SKILL 步骤 2.3 别名探测（生成操作序列：解释器惯例必配、别名键≠包名对照、不配场景） |
| 「字段是否合法 / lilac.py 能用哪些函数」 | `lilac` → references §1-§3、§5 |
| 「lilac 怎么运行 / 为何不触发更新」 | `lilac` → SKILL 运行原理 + 常见运行时故障（详见 references §7 故障速查） |
| 「为何触发了构建 / 误触发排查」 | `lilac` → SKILL BuildReason 诊断（`nomypy.py` 7 类构建原因） |
| 「手动构建某个包 / 构建后产物没进仓库」 | `lilac <pkg>`（位置参数，**无 `-p`**）→ SKILL CLI 手动运维 + 构建产物收尾（§5、§7 #13） |
| 「遇到构建失败如何排查（soname/超时/AUR 校验等）」 | `lilac` → references §7 故障速查（按构建生命周期 13 类） |

> **决策不依赖仓库风格**：按上游真实形态（GitHub tag / Releases / 生态源 / 官方包 / 本仓库依赖 / 是否同步 AUR）驱动，对任何 lilac 仓库通用。
> **只写规则与字段契约，不写具体包版本快照**：实际 tag 形态由决策引擎 `git ls-remote` 实查后生成；易变量以 `lilac2/*` 源码与 `schema-docs/` 为权威源。

## 工程实战要点（面向 AI agent）

1. **新包接入**：取 PKGBUILD → 走 SKILL 0→5，产出最小配置 → 提醒放入仓库正确目录。
2. **配置修正**：先读现有 lilac.yaml，对照自纠错表（config §6）与当前上游真实 tag 形态改，不整体重写。
3. **拿不准 tag 形态**：决策引擎主动 `git ls-remote --tags` 实查，不靠猜。
4. **alias 生成**：扫 `depends`/`makedepends`，解释器/运行时（python 等模块包）**惯例必配**；库命中 §4.2 别名目录用「别名键」（openssl→libssl/libcrypto、boost-libs→boost）；目录外冷门 so 才手写 alpm（见 SKILL 步骤 2.3）。
5. **CLI 操作**：手动构建用 `lilac <pkg>`（无 `-p` 等 flag，参数即位置包名，可 `pkg:runner`）；构建后产物收尾不归 lilac（外部 `archrepo2`/`postrun`）。
6. **安全**：secret 不进 lilac.yaml（走 `config.toml`/keyfile）；不替用户执行 lilac 构建命令（除非明确要求）；钩子里只调 `lilac2/api.py` 公开 API，并统一用 `run_protected` 封装外部命令，不拼接 exec PKGBUILD 内容。

## 维护与更新（自更新）

技能内容随 lilac2 升级更新，遵循 references §8「扩展 SOP」：遇源码变更按下列映射定位到具体小节修改（**不**在文件内维护 `# LAST_VERIFIED` 之类标记，该标记已移除）：

| lilac2 源码变更 | 需更新 |
|---|---|
| `lilacyaml.py` / `typing.py`（LilacInfo 字段） | references §1 |
| `api.py`（API / 模块变量） | references §2 / §3 |
| `aliases.yaml`（别名目录） | references §4.2、§4.7 |
| `nvchecker.py` / `nomypy.py`（版本触发、BuildReason） | SKILL 运行原理 + BuildReason 诊断 + references §7 |
| `worker.py` / `cmd.py` / `pkgbuild.py`（构建管线、沙箱、构建前校验） | SKILL 运行原理 + references §7 |
| `building.py` / `db.py`（调度、失败重构建、构建历史 PostgreSQL） | SKILL 运行原理 + references §5、§7 |
| `l10n/*/mail.ftl`（失败类型） | references §7（以**类型**为准，不逐字抄） |
| `update_on` 新增 source / tag 策略 | references §4 + SKILL 步骤 1/2 |
| 仓库级配置项变化（`dburl`/`postrun`/`disable_local_worker` 等） | references §5 |
| `scripts/`（dbsetup.sql、build-cleaner 等运维设施） | SKILL 构建产物收尾 + references §5 |

扩展新 source：按 references §8 扩展 SOP 追加写法并更新选型表（§4.4）；验证方式：对照 `lilac2/aliases.yaml` 与 nvchecker 文档，必要时抽取目标仓库真实包跑决策引擎与仓库内真实配置比对。测试：仓库 `tests/`（test_api / test_dependency_resolution / test_lilaclib / test_rpkgs）跑 `pytest`；`schema-docs/` 提供 lilac-yaml schema 权威定义，字段改动时优先查它。
