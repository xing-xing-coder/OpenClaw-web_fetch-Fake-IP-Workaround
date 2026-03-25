# openclaw-web-fetch-fakeip-workaround
针对 npm 全局安装版 OpenClaw 的临时补丁方案。 适用于 Clash / Mihomo / Surge 等代理工具在 TUN + fake-ip 模式下，web_fetch 因 SSRF 保护误判 198.18.0.0/15 而失败的场景。
