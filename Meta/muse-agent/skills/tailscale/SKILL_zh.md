---
name: "tailscale"
title: "Tailscale"
description: "设置 Muse 内置的 Tailscale 连接器，加入 Tailscale 尾网或 Headscale 网络，查看状态，并通过 TCP 隧道代理访问私有设备。如需了解有关 Tailscale、VPN、MagicDNS、网络出口、出口节点或浏览器路由的问题及支持的限制，请参阅相关文档。"
metadata: { "不包含在提示中": 假 }
---
# Tailscale

在进行设置、网络访问或解答相关功能问题之前，请先阅读 `~/docs/devices/tailscale.md`。该文档涵盖了连接步骤、支持的命令、权限审批、DNS 以及网络限制等内容。

请使用其中所述的捆绑式 `/opt/hatch/bin/tailscale` 命令行工具。请勿安装上游客户端，也不要启动单独的 `tailscaled` 守护进程。