# OpenClaw `web_fetch` Fake-IP Workaround (npm 全局安装版)

![Status](https://img.shields.io/badge/status-workaround-orange)
![Scope](https://img.shields.io/badge/scope-npm%20global%20install-blue)
![Target](https://img.shields.io/badge/target-web__fetch%20fake--ip-red)
![License](https://img.shields.io/badge/license-MIT-green)

> 针对 **npm 全局安装版 OpenClaw** 的临时补丁方案。  
> 适用于 Clash / Mihomo / Surge 等代理工具在 **TUN + fake-ip** 模式下，`web_fetch` 因 SSRF 保护误判 `198.18.0.0/15` 而失败的场景。

---

## 这是什么

这是一个在 OpenClaw 官方修复尚未落地前可用的 **临时 workaround**。

它不会修改你的 `openclaw.json` 配置，也不会要求你源码构建 OpenClaw，而是直接对 **npm 全局安装版** 包内的打包产物做最小修改：

```js
policy: { allowRfc2544BenchmarkRange: true }
```

插入到 `web_fetch` 实际调用 `fetchWithWebToolsNetworkGuard({...})` 的位置，从而允许 fake-ip 常见使用的 `198.18.0.0/15` 网段通过该层检查。

---

## 适用场景

你大概率适合使用这个补丁，如果同时满足下面几条：

- 你是用 **npm 全局安装** 的 OpenClaw
- 你使用 Clash / Mihomo / Surge 等代理工具
- 你启用了 **TUN + fake-ip**
- `web_fetch` 在某些网站上失败
- 错误表现接近：
  - `Blocked: resolves to private/internal/special-use IP address`
  - 或类似 SSRF blocked 错误

---

## 不适用场景

这个方案 **不解决** 下面这些问题：

- 证书错误
- 代理端口或规则配置错误
- 环境变量代理未生效
- 与 fake-ip 无关的 `web_fetch` 问题
- 你使用的是 **源码构建版** OpenClaw，而不是 npm 全局安装版

---

## 为什么专门写 npm 安装版教程

OpenClaw 常见有两种使用方式：

### 1. npm / pnpm 全局安装

例如：

```bash
npm install -g openclaw@latest
```

这种方式下，你实际运行的是 **已经打包好的 npm 包**。  
因此应该修改包内的 `dist/` 编译产物，而不是去找源码仓库里的 `src/...` 文件。

### 2. 从源码构建

例如：

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm ui:build
pnpm build
```

这种方式下才是改 `src/...` 再重新构建。

**本仓库 / 本教程只针对第 1 种：npm 全局安装版。**

---

## 如何判断自己是不是 npm 全局安装版

这是很多人最容易搞错的地方，所以单独列出来。

### 方法 1：回忆安装命令

如果你当初是这样安装的：

```bash
npm install -g openclaw@latest
```

那你就是 npm 全局安装版。

---

### 方法 2：查看 npm 全局模块目录

执行：

```bash
npm root -g
```

如果输出类似：

```bash
/home/yourname/.npm-global/lib/node_modules
```

并且下面存在：

```bash
$(npm root -g)/openclaw
```

那就是 npm 全局安装版。

你还可以继续确认：

```bash
ls "$(npm root -g)/openclaw"
```

如果能看到诸如：

- `package.json`
- `openclaw.mjs`
- `dist/`

那基本可以确定。

---

### 方法 3：查看 `openclaw` 命令实际来自哪里

执行：

```bash
command -v openclaw
readlink -f "$(command -v openclaw)"
```

如果最终路径落在 npm 全局目录里，例如：

```bash
/home/yourname/.npm-global/lib/node_modules/openclaw/openclaw.mjs
```

或者它的软链接目标在同一类目录下，那么也是 npm 全局安装版。

---

### 方法 4：你根本没有源码仓库

如果你本地没有下面这些典型源码仓库特征：

- `.git/`
- `src/`
- `pnpm-lock.yaml`
- `pnpm build`
- `git clone` 下来的完整项目结构

而只有 npm 全局目录下的包内容，那么你也不是源码构建版。

---

## 这个补丁做了什么

补丁不会改配置文件，不会引入新的 schema，也不会要求重新构建。

它只做一件事：

在 `web_fetch` 的实际打包后 JS 中，找到这段调用：

```js
fetchWithWebToolsNetworkGuard({
  url: params.url,
  maxRedirects: params.maxRedirects,
  timeoutSeconds: params.timeoutSeconds,
  init: { ... }
})
```

然后插入一行：

```js
policy: { allowRfc2544BenchmarkRange: true }, // openclaw-fakeip-patch
```

变成：

```js
fetchWithWebToolsNetworkGuard({
  url: params.url,
  maxRedirects: params.maxRedirects,
  timeoutSeconds: params.timeoutSeconds,
  policy: { allowRfc2544BenchmarkRange: true }, // openclaw-fakeip-patch
  init: { ... }
})
```

这样做的好处是：

- 不依赖配置 schema 是否支持新字段
- 不会因为配置项未被官方接受而导致启动失败
- 只放开 fake-ip 常见用到的 RFC2544 网段，不是放开全部私网

---

## 使用前提示

在执行补丁前，请先确认：

- 你确实是 **npm 全局安装版**
- 你的问题确实和 fake-ip / `198.18.x.x` 误判有关
- 你理解这是 **临时补丁**，不是官方正式修复
- OpenClaw 升级后，打包文件名可能变化，因此脚本会重新搜索目标文件，而不是写死具体文件名

---

## 完整补丁脚本

将下面内容保存为：

```bash
patch-openclaw-global-fakeip.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail

CMD="${1:-}"
MARKER="openclaw-fakeip-patch"

usage() {
  echo "用法:"
  echo "  bash patch-openclaw-global-fakeip.sh status"
  echo "  bash patch-openclaw-global-fakeip.sh inspect"
  echo "  bash patch-openclaw-global-fakeip.sh apply"
  echo "  bash patch-openclaw-global-fakeip.sh revert"
}

get_pkg_dir() {
  if [ -n "${OPENCLAW_PKG_DIR:-}" ]; then
    printf '%s\n' "$OPENCLAW_PKG_DIR"
    return
  fi

  local root=""
  root="$(npm root -g 2>/dev/null || true)"
  if [ -n "$root" ] && [ -d "$root/openclaw" ]; then
    printf '%s\n' "$root/openclaw"
    return
  fi

  local bin=""
  bin="$(command -v openclaw 2>/dev/null || true)"
  if [ -n "$bin" ]; then
    bin="$(readlink -f "$bin" 2>/dev/null || printf '%s' "$bin")"
    local maybe
    maybe="$(dirname "$bin")"
    if [ -f "$maybe/package.json" ]; then
      printf '%s\n' "$maybe"
      return
    fi
  fi

  return 1
}

run_node() {
  local mode="$1"
  local pkg
  pkg="$(get_pkg_dir)" || {
    echo "[ERROR] 找不到 openclaw 安装目录"
    echo "[HINT] 可以手动指定:"
    echo "       OPENCLAW_PKG_DIR=/你的/node_modules/openclaw bash $0 $mode"
    exit 1
  }

  echo "[INFO] package dir: $pkg"

  OPENCLAW_PKG_DIR="$pkg" MODE="$mode" node <<'NODE'
const fs = require('fs');
const path = require('path');

const pkg = process.env.OPENCLAW_PKG_DIR;
const mode = process.env.MODE;
const marker = 'openclaw-fakeip-patch';

function walk(dir, out = []) {
  for (const ent of fs.readdirSync(dir, { withFileTypes: true })) {
    const p = path.join(dir, ent.name);
    if (ent.isDirectory()) {
      if (ent.name === '.git') continue;
      walk(p, out);
    } else if (ent.isFile() && p.endsWith('.js') && !p.includes('.fakeip.')) {
      out.push(p);
    }
  }
  return out;
}

function findCandidates() {
  const files = walk(pkg);
  const out = [];
  for (const file of files) {
    let text = '';
    try {
      text = fs.readFileSync(file, 'utf8');
    } catch {
      continue;
    }

    if (
      text.includes('fetchWithWebToolsNetworkGuard({') &&
      text.includes('runWebFetch') &&
      (text.includes('name:"web_fetch"') || text.includes('name: "web_fetch"'))
    ) {
      out.push({ file, text });
    }
  }
  return out;
}

function findPatchPoint(text) {
  const lines = text.split('\n');

  const starts = [];
  for (let i = 0; i < lines.length; i++) {
    if (lines[i].includes('fetchWithWebToolsNetworkGuard({')) {
      starts.push(i);
    }
  }

  if (starts.length !== 1) {
    throw new Error(`目标调用块起点数量异常: ${starts.length}`);
  }

  const start = starts[0];
  let timeoutIdx = -1;
  let initIdx = -1;

  for (let i = start; i < Math.min(lines.length, start + 40); i++) {
    if (
      timeoutIdx === -1 &&
      (
        lines[i].includes('timeoutSeconds: params.timeoutSeconds') ||
        lines[i].includes('timeoutSeconds:params.timeoutSeconds')
      )
    ) {
      timeoutIdx = i;
      continue;
    }

    if (
      timeoutIdx !== -1 &&
      (
        /^\s*init:\s*\{\s*$/.test(lines[i]) ||
        lines[i].includes('init:{') ||
        lines[i].includes('init: {')
      )
    ) {
      initIdx = i;
      break;
    }
  }

  if (timeoutIdx === -1 || initIdx === -1 || initIdx <= timeoutIdx) {
    throw new Error('没找到预期的 timeoutSeconds -> init 结构');
  }

  return { lines, start, timeoutIdx, initIdx };
}

function showPatchWindow(text) {
  const { lines, start, timeoutIdx, initIdx } = findPatchPoint(text);

  console.log('--- exact patch window ---');
  for (let i = Math.max(0, start - 2); i <= Math.min(lines.length - 1, initIdx + 4); i++) {
    console.log(lines[i]);
  }

  console.log('');
  console.log('--- exact insert line ---');
  console.log(lines[timeoutIdx]);
  console.log('>>> WILL INSERT BELOW THIS LINE <<<');
  console.log(lines[initIdx]);
}

const candidates = findCandidates();

if (candidates.length === 0) {
  console.log('[STATUS] no-candidate');
  process.exit(0);
}

if (candidates.length > 1) {
  console.log('[STATUS] multiple-candidates');
  for (const c of candidates) {
    console.log(' - ' + c.file + (c.text.includes(marker) ? ' [patched]' : ' [clean]'));
  }
  process.exit(0);
}

const { file, text } = candidates[0];

if (mode === 'status') {
  console.log('[STATUS] ' + (text.includes(marker) ? 'patched' : 'clean'));
  console.log('[TARGET] ' + file);
  process.exit(0);
}

if (mode === 'inspect') {
  console.log('[STATUS] ' + (text.includes(marker) ? 'patched' : 'clean'));
  console.log('[TARGET] ' + file);
  console.log('');

  try {
    showPatchWindow(text);
  } catch (e) {
    console.log('[INSPECT-ERROR] ' + e.message);
    process.exit(1);
  }

  process.exit(0);
}

if (mode === 'apply') {
  if (text.includes(marker)) {
    console.log('[INFO] 已经打过补丁，跳过');
    console.log('[TARGET] ' + file);
    process.exit(0);
  }

  const { lines, initIdx } = findPatchPoint(text);
  const indent = (lines[initIdx].match(/^(\s*)/) || ['', ''])[1];
  const patchLine = `${indent}policy: { allowRfc2544BenchmarkRange: true }, // ${marker}`;

  const backup = file + '.fakeip.bak';
  if (!fs.existsSync(backup)) {
    fs.copyFileSync(file, backup);
  }

  lines.splice(initIdx, 0, patchLine);
  fs.writeFileSync(file, lines.join('\n'), 'utf8');

  const verify = fs.readFileSync(file, 'utf8');
  if (!verify.includes(marker)) {
    throw new Error('补丁校验失败');
  }

  console.log('[OK] 已应用补丁');
  console.log('[TARGET] ' + file);
  console.log('[BACKUP] ' + backup);
  process.exit(0);
}

if (mode === 'revert') {
  if (!text.includes(marker)) {
    console.log('[INFO] 当前未打补丁，无需回滚');
    console.log('[TARGET] ' + file);
    process.exit(0);
  }

  const lines = text.split('\n');
  const before = lines.length;
  const next = lines.filter(line => !line.includes(marker));
  const removed = before - next.length;

  if (removed !== 1) {
    throw new Error(`预期删除 1 行补丁，实际删除 ${removed} 行`);
  }

  const backup = file + '.fakeip.revert.bak';
  if (!fs.existsSync(backup)) {
    fs.copyFileSync(file, backup);
  }

  fs.writeFileSync(file, next.join('\n'), 'utf8');

  const verify = fs.readFileSync(file, 'utf8');
  if (verify.includes(marker)) {
    throw new Error('回滚后仍检测到补丁标记');
  }

  console.log('[OK] 已回滚补丁');
  console.log('[TARGET] ' + file);
  console.log('[BACKUP] ' + backup);
  process.exit(0);
}

throw new Error('未知模式');
NODE
}

case "$CMD" in
  status) run_node status ;;
  inspect) run_node inspect ;;
  apply) run_node apply ;;
  revert) run_node revert ;;
  *)
    usage
    exit 1
    ;;
esac
```

---

## 给脚本加执行权限

```bash
chmod +x patch-openclaw-global-fakeip.sh
```

---

## 推荐使用顺序

### 1. 查看状态

```bash
bash patch-openclaw-global-fakeip.sh status
```

### 2. 先检查目标位置，不直接修改

```bash
bash patch-openclaw-global-fakeip.sh inspect
```

如果输出里出现类似下面的结构，就说明脚本已经准确定位到 `web_fetch` 的实际打包代码片段：

```text
--- exact patch window ---
        let finalUrl = params.url;
        try {
                const result = await fetchWithWebToolsNetworkGuard({
                        url: params.url,
                        maxRedirects: params.maxRedirects,
                        timeoutSeconds: params.timeoutSeconds,
                        init: { headers: {
```

以及：

```text
--- exact insert line ---
                        timeoutSeconds: params.timeoutSeconds,
>>> WILL INSERT BELOW THIS LINE <<<
                        init: { headers: {
```

这说明补丁会被插入到正确位置。

### 3. 应用补丁

```bash
bash patch-openclaw-global-fakeip.sh apply
```

### 4. 再检查一次

```bash
bash patch-openclaw-global-fakeip.sh inspect
```

此时应看到：

```text
[STATUS] patched
```

并且目标位置会多出：

```js
policy: { allowRfc2544BenchmarkRange: true }, // openclaw-fakeip-patch
```

---

## 应用补丁后要做什么

补丁只修改磁盘上的已安装 JS 文件。  
如果 OpenClaw 已经在运行，你还需要 **重启当前运行中的 OpenClaw / gateway / daemon**，让新代码重新加载。

你也可以先执行：

```bash
openclaw status
```

确认当前运行状态。

---

## 如何验证补丁是否生效

最简单的方法：

1. 找一个之前在 fake-ip 环境下必定失败的网址
2. 重启 OpenClaw
3. 再执行一次同样的 `web_fetch`
4. 观察是否不再报 SSRF blocked / private/internal/special-use IP 类错误

如果之前失败、补丁后恢复正常，说明补丁已经生效。

---

## 如何回滚

如果你想恢复原状：

```bash
bash patch-openclaw-global-fakeip.sh revert
```

然后重启 OpenClaw 即可。

脚本也会自动备份目标文件：

- 应用补丁时：`*.fakeip.bak`
- 回滚时：`*.fakeip.revert.bak`

---

## OpenClaw 升级后怎么办

OpenClaw 升级后，打包文件名可能会变化，例如：

```text
pi-embedded-xxxxxx.js
```

所以这个脚本不会写死具体文件路径，而是会重新扫描 npm 包目录，寻找同时满足以下条件的目标：

- 包含 `fetchWithWebToolsNetworkGuard({`
- 包含 `runWebFetch`
- 包含 `name:"web_fetch"` 或 `name: "web_fetch"`

也就是说：

- **升级后如果官方还没修复**，通常重新执行一次脚本即可
- **升级后如果官方已经修复**，则不再需要这个补丁

---

## 风险说明

请注意：

1. 这是 **临时补丁**，不是官方正式发布的修复
2. 它只针对 **RFC2544 fake-ip 网段 `198.18.0.0/15`**，不是放开所有私网
3. 它适用于 **npm 全局安装版**，源码构建版请不要直接套用
4. 如果某个版本的打包结构变化太大，脚本可能会提示：
   - `no-candidate`
   - `multiple-candidates`
   - `没找到预期的 timeoutSeconds -> init 结构`
   这时不要强行修改，建议先 `inspect`，再决定是否调整脚本

---

## FAQ

### Q1：这个补丁会不会导致配置文件报错？

不会。  
因为它不改 `openclaw.json`，也不依赖 schema 支持新的配置字段。

### Q2：为什么不直接改配置？

因为在官方尚未加入对应 schema 支持前，直接往配置里加未知字段，未来某些版本有可能导致校验失败或启动报错。

### Q3：为什么不用源码补丁？

因为本教程的目标用户是 **npm 全局安装版**。  
这类用户通常没有源码仓库，也没有本地构建链。

### Q4：这个补丁是不是 100% 保证有效？

不是。  
它命中的是真正的问题根因之一：`web_fetch` 没有把 fake-ip 需要的策略传进网络守卫。  
如果你的问题还夹杂了证书、代理规则、环境变量代理或其它网络问题，这个补丁不能保证一并解决。

---

## 相关链接

- OpenClaw 安装文档  
  https://docs.openclaw.ai/zh-CN/install

- PR #51407  
  https://github.com/openclaw/openclaw/pull/51407

- Issue #25322  
  https://github.com/openclaw/openclaw/issues/25322

- Issue #27597  
  https://github.com/openclaw/openclaw/issues/27597

---

## 致谢

感谢 OpenClaw 社区中对 `web_fetch`、SSRF guard 与 fake-ip 兼容性问题的讨论与修复提案。  
本文只是把一个对 **npm 安装版用户** 更友好的临时方案整理出来，方便在官方修复前临时使用。

---

## License

MIT
