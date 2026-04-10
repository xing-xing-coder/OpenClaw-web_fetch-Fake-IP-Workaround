# OpenClaw `web_fetch` Fake-IP Workaround (npm 全局安装版)

![Status](https://img.shields.io/badge/status-workaround-orange)
![Scope](https://img.shields.io/badge/scope-npm%20global%20install-blue)
![Target](https://img.shields.io/badge/target-web__fetch%20fake--ip-red)
![License](https://img.shields.io/badge/license-MIT-green)

> 适用于 **npm 全局安装版 OpenClaw**。  
> 场景：Clash / Mihomo / Surge 开启 **TUN + fake-ip** 后，`web_fetch` 因 SSRF 保护误判 `198.18.0.0/15` 而失败。

---

## 最新情况

OpenClaw 已合并修复 PR [`#61830`](https://github.com/openclaw/openclaw/pull/61830)，并计划在 **`v2026.4.10`** 起正式解决这个问题。

如果你的版本 **低于 `v2026.4.10`**，仍可使用下面的方式处理。

### `v2026.4.10` 及以更新版本的解决方法

在 `openclaw.json` 中开启：

```json
{
  "tools": {
    "web": {
      "fetch": {
        "ssrfPolicy": {
          "allowRfc2544BenchmarkRange": true
        }
      }
    }
  }
}
```

常见报错表现：

- `Blocked: resolves to private/internal/special-use IP address`

---

## 适用条件

下面这些情况同时满足时，这个 workaround 才有意义：

- 你是通过 **npm 全局安装** 的 OpenClaw
- 你使用 Clash / Mihomo / Surge 等代理工具
- 你启用了 **TUN + fake-ip**
- `web_fetch` 在部分网站上失败
- 错误内容与 SSRF 拦截、`198.18.x.x`、`private/internal/special-use IP` 相关

---

## 不适用的情况

这个方案不解决以下问题：

- 证书错误
- 代理规则或端口配置错误
- 环境变量代理未生效
- 与 fake-ip 无关的 `web_fetch` 问题
- **源码构建版** OpenClaw

---

## 先确认是不是 npm 全局安装版

执行：

```bash
npm root -g
ls "$(npm root -g)/openclaw"
```

如果目录里能看到下面这些内容，通常就是 npm 全局安装版：

- `package.json`
- `openclaw.mjs`
- `dist/`

也可以再确认一次：

```bash
command -v openclaw
readlink -f "$(command -v openclaw)"
```

如果最终路径落在 `node_modules/openclaw` 下，也说明你当前用的是 npm 全局安装版。

---

## 这个补丁做了什么

这个 workaround **不会修改 `openclaw.json`**，也**不需要重新构建** OpenClaw。

它的思路很简单：
在 `web_fetch` 实际调用 `fetchWithWebToolsNetworkGuard({...})` 的位置，插入下面这行：

```js
policy: { allowRfc2544BenchmarkRange: true }
```

这样可以允许 fake-ip 常见使用的 `198.18.0.0/15` 网段通过该层检查。

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

## 应用补丁后还要做什么

补丁修改的是磁盘上的已安装 JS 文件。  
如果 OpenClaw 当前已经在运行，你还需要**重启正在运行的 OpenClaw / gateway / daemon**，让新代码重新加载。

必要时可以先执行：

```bash
openclaw status
```

确认当前运行状态。

---

## 如何验证是否生效

最直接的验证方式如下：

1. 找一个之前在 fake-ip 环境下必定失败的网址
2. 重启 OpenClaw
3. 再执行一次相同的 `web_fetch`
4. 观察是否不再出现 SSRF blocked / private/internal/special-use IP 类错误

如果之前会失败、补丁后恢复正常，基本就说明 workaround 已经生效。

---

## 如何回滚

如果你想恢复原状：

```bash
bash patch-openclaw-global-fakeip.sh revert
```

然后重启 OpenClaw gateway 即可。

---

## 升级后怎么办

OpenClaw 升级后，打包文件名或内部结构可能变化，因此这个脚本通常会重新搜索目标位置，而不是写死具体路径。

所以一般分两种情况：

- **升级后官方仍未修复**：通常重新执行一次脚本即可
- **升级后官方已修复**：则不再需要这个 workaround

如果你的版本已经是 **`v2026.4.10` 或更高**，优先使用官方配置项，不建议继续依赖这个临时补丁。

---

## 风险说明

请注意，这仍然是一个**临时 workaround**，不是官方正式发布的修复方案。

需要特别说明的是：

1. 它只针对 **RFC2544 fake-ip 网段 `198.18.0.0/15`**
2. 它不是放开所有私网或所有 SSRF 限制
3. 它只适用于 **npm 全局安装版**
4. 如果某个版本的打包结构变化较大，脚本可能无法命中目标位置

如果脚本检查失败，不要直接手改，建议先确认版本、安装方式，以及当前问题是否确实由 fake-ip 引起。

---

## FAQ

### Q1：这个补丁会不会导致配置文件报错？

不会。  
因为它不修改 `openclaw.json`，也不依赖配置 schema 是否已经支持新字段。

### Q2：为什么不直接改配置？

因为低版本在官方尚未完整支持对应配置前，直接加未知字段，可能存在校验失败或启动异常的风险。

### Q3：这个 workaround 一定有效吗？

不一定。  
它解决的是 fake-ip 被 SSRF 保护误判这一类问题。如果你的环境里还同时存在证书、代理规则、环境变量代理或其它网络问题，这个补丁不能保证一并解决。

---

## 相关链接

- OpenClaw 安装文档
  https://docs.openclaw.ai/zh-CN/install
  
- PR `#61830`
  https://github.com/openclaw/openclaw/pull/61830
  
- Issue `#25322`
  https://github.com/openclaw/openclaw/issues/25322
- Issue `#27597`
  https://github.com/openclaw/openclaw/issues/27597

---

## 致谢

感谢 OpenClaw 社区中对 `web_fetch`、SSRF guard 与 fake-ip 兼容性问题的讨论与修复提案。  
本文只是把一个对 **npm 安装版用户** 更友好的临时方案整理出来，方便在官方修复前临时使用。

---

## License

MIT
