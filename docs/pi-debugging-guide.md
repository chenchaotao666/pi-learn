# Pi 调试指南

本文档说明如何在源码级调试 Pi（monorepo 中的 `packages/coding-agent` CLI 及其依赖包）。

## 前提

- Node.js >= 22.19.0（源码模式依赖较新的 `--import` 与原生 TypeScript strip-types 行为）。验证：`node --version`。
- 依赖已安装：`npm install --ignore-scripts`。

## 为什么需要 source-resolver

直接运行 `packages/coding-agent/src/cli.ts` 时，代码里 import 的是包名（如 `@earendil-works/pi-ai`）。Node 默认会把这些解析到各包的 `dist/` 产物，导致：

- 断点打在 `.ts` 源码上命中不了（实际执行的是 `dist` 里的编译产物）；
- 改了源码不重新 build 就不生效。

`packages/coding-agent/src/experimental/source-resolver.ts` 通过 `node:module` 的 `registerHooks`，按根 `tsconfig.json` 的 `compilerOptions.paths` 把 `@earendil-works/*` 包名解析到对应包的 **TypeScript 源码**。所有源码级运行/调试都必须带上它。

`./pi-test.sh` 内部就是 `node --import <source-resolver> packages/coding-agent/src/cli.ts`。

## 方式一：VS Code（推荐）

仓库根目录已提供 `.vscode/launch.json`，包含三个配置：

### Debug pi (interactive)

启动交互式 TUI。要点：

- `runtimeArgs` 里通过 `--import` 加载 source-resolver；
- `console` 必须是 `integratedTerminal`（TUI 需要真实终端渲染，debug console 无法显示）。

在任意 `.ts` 文件打断点，按 F5 选择该配置即可。

### Debug pi (print mode)

非交互单次执行，适合调试核心逻辑（Agent 循环、Provider 调用、工具执行等），不用操作 TUI：

```
"args": ["--print", "hello"]
```

按需修改 `args` 里的 prompt 或其他 CLI 参数。

### Debug vitest (current file)

在编辑器中打开某个测试文件后按 F5，只运行该文件的 vitest 测试（`node_modules/vitest/dist/cli.js --run ${relativeFile}`）。适合调试单个测试或用测试驱动调试某个模块。

### 屏蔽 API key

`--no-env` 只是 `pi-test.sh` 脚本自己的参数（负责 unset 环境变量），CLI 本身不认识。调试时如需屏蔽某个 key，在 launch 配置中加：

```json
"env": { "ANTHROPIC_API_KEY": null }
```

`null` 表示从进程环境中移除该变量。可 unset 的完整变量列表见 `pi-test.sh`。

## 方式二：Chrome DevTools / Node Inspector

不用 VS Code 时，手动加 `--inspect-brk`：

```bash
RESOLVER_URL="$(node -p 'require("node:url").pathToFileURL(process.argv[1]).href' packages/coding-agent/src/experimental/source-resolver.ts)"
node --inspect-brk --import "$RESOLVER_URL" packages/coding-agent/src/cli.ts
```

然后 Chrome 打开 `chrome://inspect` 附加进程。Node 原生 strip-types 运行，`.ts` 断点直接生效，无需 sourcemap。

## 方式三：调试测试

```bash
# vitest（大多数包）
node node_modules/vitest/dist/cli.js --run test/specific.test.ts

# packages/tui 使用 node:test
node --test test/specific.test.ts
```

加 `--inspect-brk` 或用 launch.json 的 vitest 配置即可断点调试。

## 不建议的方式

不要调试构建产物 `packages/coding-agent/dist/bundle/cli.js`：它是 bundle 后的代码，无法对应源码行号。需要验证构建产物行为时，用 `./pi-test.sh` 确认源码行为后再 `npm run build` 对比。

## 常见问题

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| 断点不命中 / 命中的是 `dist` | 没带 source-resolver | 确认 `--import` 参数存在 |
| TUI 显示异常或立刻退出 | 用了 debug console | `console` 改为 `integratedTerminal` |
| `--import` 报错或 TS 语法报错 | Node < 22.19 | 升级 Node |
| 请求发了真实 API | 环境里有 provider key | launch 配置 `"env"` 里置 `null`，或调试 `test/suite/`（faux provider，无真实调用） |
