# nekobox-harmony-install-skill

让任何装了 CodeArts / Code agent 的开发者，把 [NekoBox4Harmony](https://github.com/xiaoli8571/NekoBox4Harmony)（sing-box 内核 VPN 客户端）从源码构建并安装到纯血鸿蒙（HarmonyOS NEXT）真机。

## 用法

把 `SKILL.md` 放进你的 code agent skills 目录（或会话里直接 `@SKILL.md` 引用），对 agent 说「安装 NekoBox4Harmony 到我的鸿蒙手机」即可。agent 会按 SKILL.md 自适应检测你的环境并执行。

## 前提

- macOS（Linux/Windows 路径需自行调整）
- HarmonyOS command-line-tools（含 hvigor/ohpm/sdk/hdc）
- JDK 17+
- 鸿蒙开发者调试签名材料（`~/.ohos/config` 下 `.cer/.p7b/.p12`，已绑定你的 bundleName + 设备 UDID）
- 真机已开开发者模式 + USB 调试

## 为什么不能「零门槛」

纯血鸿蒙不允许安装未签名 HAP。每个人签名材料/设备/bundleName 不同，skill 只能自适应检测复用你已有的调试签名，无法替你生成（生成需 DevEco Studio GUI 或 AGC 网页操作）。

## 流程概览

clone → 改 bundleName 复用你的调试签名 → 改 SDK 版本匹配手机 → 移除个人拿不到的系统级权限 → 关闭测速绕过隧道默认值 → 移除 macOS 隔离属性 → hvigorw 构建（自动签名）→ hdc 安装 → 本机 HTTP 中转订阅。

详见 `SKILL.md`。