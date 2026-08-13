# chaineye

> 零依赖、纯 Node 单文件「依赖链眼」。一条命令把仓库变成 import 影响图，秒答**「改了 X，谁要跟着改」**。

[![node](https://img.shields.io/badge/node-%3E%3D18-brightgreen)](https://nodejs.org)
[![零依赖](https://img.shields.io/badge/dependencies-0-brightgreen)](./package.json)
[![license](https://img.shields.io/badge/license-MIT-blue)](./LICENSE)

每次进陌生仓库，最怕两件事：一是 grep 半天才理清「谁依赖谁」，二是改了一个底层文件，漏掉上游调用方导致连锁崩。**chaineye 只解决这一个高频问题——影响范围**，零配置、确定性、可进 CI，不跟你谈「全功能依赖分析平台」。

```bash
# 仓库概览 + CI 门禁
npx chaineye .

# 改了 parser.js，谁受影响？（沿依赖链向上传递）
npx chaineye impact src/core/parser.js

# 谁直接 import 了它（一层上游）
npx chaineye why src/core/parser.js

# 循环依赖 / 孤立文件 / 枢纽文件
npx chaineye cycles
npx chaineye orphans
npx chaineye hubs
```

---

## 为什么是 chaineye（而不是别的）

| 工具 | 依赖 | 配置成本 | 聚焦 | 痛点 |
|------|------|---------|------|------|
| [madge](https://github.com/pahen/madge) | 需 `npm i` | 中（支持多种配置） | 循环依赖 / 可视化图 | 功能散、输出重、CI 门禁要自己拼 |
| [dependency-cruiser](https://github.com/sverweij/dependency-cruiser) | 需 `npm i` | 高（规则文件复杂） | 全功能依赖策略 | 强大但「杀鸡用牛刀」，上手曲线陡 |
| **chaineye** | **0** | **零**（开箱即用） | **影响范围这一件事** | 切中「改 X 前先看清波及面」的刚需 |

**chaineye = 零 npm 依赖 + 单文件 + 聚焦影响范围 + 原生 CI 门禁** 的交集空白：

- **零依赖**：只用 Node 内置 `fs` / `path`，`git clone` 即可跑，无 `node_modules`。
- **聚焦**：不搞全功能平台，只把「改 X 影响谁」做到极致，确定性结果可复现。
- **CI 门禁**：内置 `--max-cycles` / `--max-orphans` / `--fail-on-cycle` / `--json`，直接当流水线质量闸门。

---

## 安装

```bash
# 全局安装
npm install -g chaineye

# 或免安装直接用
npx chaineye .

# 或克隆后本地跑
git clone https://github.com/huanweide/chaineye && cd chaineye && node index.js .
```

要求 Node.js >= 18（纯内置模块，无特殊运行要求）。

---

## 使用

```bash
chaineye [root]                      仓库概览 + CI 门禁
chaineye impact <file> [root]        改了 file，谁会受影响（反向依赖闭包）
chaineye why   <file> [root]         谁直接 import 了 file（一层上游）
chaineye cycles [root]               列出所有处在循环依赖中的文件
chaineye orphans [root]              列出孤立文件（无入边无出边）
chaineye hubs   [root]               枢纽文件 Top（被依赖最多）
```

全局选项：

```bash
--json               机器可读 JSON 输出（适合 CI 消费）
--root <dir>         指定仓库根目录（默认 "."）
--max-cycles <N>     循环依赖节点数上限（超过则不通过）
--max-orphans <N>    孤立文件数上限（超过则不通过）
--fail-on-cycle      只要存在循环依赖就不通过
```

示例：

```bash
# CI 里强制零循环依赖
chaineye . --max-cycles 0 --fail-on-cycle

# 拿 JSON 给流水线用
chaineye . --json
```

---

## 四问摘要（选题论证）

1. **真实痛点是什么？** 开发者 / AI 代理进入陌生仓库时，要靠 grep 手动梳理依赖关系；改底层文件前无法快速知道波及哪些上游调用方，容易漏改导致连锁故障。「影响范围」是高频且刚需的分析。
2. **为什么是它（差异化）？** madge / dependency-cruiser 能画图但配置重、输出重、CI 门槛要自己拼。chaineye 只做「影响范围」一件事，零依赖单文件、开箱即用、原生 CI 门禁，填补「巨头嫌小、现有 OSS 嫌重」的缝隙（与 formlite / wigolo 同属开源替代杠杆范式）。
3. **是否可长期维护？** 纯静态解析、无外部依赖、无网络调用，逻辑面小且稳定；新增语言只需在 `parseImports` 加一个分支，扩展成本低。
4. **能否上 star（社区价值）？** 切中「重构前的波及面评估」「CI 防循环依赖」「AI 代理理解陌生仓库」三类真实场景，单文件零依赖降低采用摩擦，易传播。

---

## 工作原理

1. **扫描**：递归遍历仓库，忽略 `node_modules` / `.git` / `dist` 等目录，收集 JS/TS/Python/Go 源文件。
2. **解析**：用正则提取每个文件里的相对导入（`import` / `export ... from` / `require` / 动态 `import()` / Python `from .x import` / Go `import "./x"`），解析成仓库内真实文件路径（自动补扩展名、目录索引）。
3. **建图**：构建依赖 DAG，`A → B` 表示「A 依赖 B」。同步维护反向边。
4. **分析**：
   - **影响范围** = 从目标文件出发，沿反向边做传递闭包（所有会因它变更而受影响的文件）。
   - **循环依赖** = 在依赖图上做 DFS 三色标记，收集所有处在环中的节点。
   - **孤立文件** = 入度与出度均为 0 的文件。
   - **枢纽文件** = 入度最高的文件（改它影响面最大）。

结果确定性：同一仓库同一次扫描，输出完全一致，可进 CI 做回归闸门。

---

## 支持语言

| 语言 | 支持的导入写法 |
|------|----------------|
| JavaScript / TypeScript | `import ... from` / `import` / `export ... from` / `require()` / 动态 `import()` |
| Python | `import x` / `from x import y` / `from .x import y` / `from ..x.y import z`（相对导入跨扩展名解析） |
| Go | `import "x"` / `import ( "x" "y" )` 中的相对路径 |

仅分析仓库内相对导入；第三方裸模块（如 `react`）不计入依赖边。

---

## CI 门禁示例

GitHub Actions：

```yaml
- name: 依赖影响门禁
  run: npx chaineye . --max-cycles 0 --fail-on-cycle
```

退出码：门禁通过为 `0`，不通过为 `1`，可直接阻断合并。

---

## 许可

[MIT](./LICENSE) © Overlord Forge
