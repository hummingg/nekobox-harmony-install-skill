# NekoBox4Harmony 安装技能

> 把 [xiaoli8571/NekoBox4Harmony](https://github.com/xiaoli8571/NekoBox4Harmony)（sing-box 内核 VPN 客户端）从源码构建并安装到纯血鸿蒙（HarmonyOS NEXT）真机。
> 适用：已具备鸿蒙调试签名环境的开发者。任何装了 CodeArts/Code agent 的机器加载本 skill 即可执行。

## 触发条件

用户请求包含：安装/装 NekoBox、NekoBox4Harmony、nekobox 鸿蒙、sing-box 鸿蒙客户端等到手机/真机/纯血鸿蒙。

## 前置环境（必须，缺一不可，逐项检测并报告缺失项）

0. **华为开发者账号**：访问 https://developer.huawei.com 注册个人开发者账号 → 完成实名认证（身份证）。后续 AGC 申请证书/Profile 都依赖此账号。

1. **HarmonyOS command-line-tools**：含 `hvigor`、`ohpm`、`tool/node`、`sdk/default/openharmony`（含 ets/native/toolchains/hdc）。常见路径 `~/Downloads/command-line-tools`。用 `mdfind -name hvigorw.js` 定位。
2. **JDK 17+**：系统 java 1.8 会让 hap-sign-tool 报 `parseAlgParameters failed`。检测 `java -version`，<17 则 `brew install openjdk@17` 并用其绝对路径。
3. **调试签名材料**：`~/.ohos/config/default_<bundleName>_<...>.{cer,p7b,p12}` + build-profile.json5 里的**加密格式**密码（`0000001B...`）。本流程走 hvigorw 构建，hvigor 能解加密密码，**无需明文**。
   - **无签名材料**：需要调试证书 + profile（绑定 bundleName + 设备 UDID）。获取途径任选其一：
     - (推荐) 装 DevEco Studio，新建任意空项目 → File→Project Structure→Signing Configs→Automatically generate signature，登录华为账号自动生成；
     - 或手动在 [AGC](https://developer.huawei.com/consumer/cn/agconnect/) 申请（步骤见下「手动 AGC 申请调试签名」）；
     - 或从已有鸿蒙项目复用其 `~/.ohos/config` 材料（本 skill 即复用既有材料，无需 DevEco）。

#### 手动 AGC 申请调试签名（无 DevEco 时）

1. **生成本地密钥对 + CSR**：
   ```
   java -jar <hap-sign-tool.jar> generate-keypair -keyAlias debugKey -keyPwd <明文> -keyAlg ECC -keySize NIST-P-256 -keystoreFile ~/.ohos/config/debug.p12 -keystorePwd <明文>
   java -jar <hap-sign-tool.jar> generate-csr -keyAlias debugKey -keyPwd <明文> -subject "C=CN,O=Personal,CN=Debug" -signAlg SHA256withECDSA -keystoreFile ~/.ohos/config/debug.p12 -keystorePwd <明文> -outFile ~/.ohos/config/debug.csr
   ```
   牢记明文密码（hvigorw 后续用加密格式，但此处生成需明文）。
2. **AGC 创建项目+应用**：登录 AGC → 我的项目→新建项目→常规→应用→添加应用（平台选 HarmonyOS NEXT），记下 bundleName（三段式，如 com.you.name）。
3. **上传 CSR 申请调试证书**：AGC → 用户与访问 → 凭证 → 数字证书 → 新建调试证书 → 上传 debug.csr → 下载 .cer。
4. **创建调试 Profile**：AGC → 凭证 → Profile → 新建调试 Profile → 选应用 + 选刚建的证书 + 填设备 UDID（`hdc shell bm get --udid`）→ 下载 .p7b。
5. **加密密码写入 build-profile.json5**：用 DevEco 的加密工具或 hvigor 内置转换把明文 storePassword/keyPassword 转成 `0000001B...` 格式填入 signingConfigs（hvigorw 构建只认加密格式）。若无加密工具，可直接在 build-profile.json5 里填明文让 hvigorw 首次构建时自动加密回写（部分版本支持）。
   - **忘 keystore 密码**：密码不可逆，打不开旧 .p12。须重新生成一套（同上任意途径），旧的应用需卸载重装。
4. **调试 profile 绑定的 bundleName**：从 `.p7b` 二进制里正则提取三段式 bundleName（如 `com.xxx.yyy`）。构建时把 NekoBox 的 `AppScope/app.json5` bundleName 改成它。
5. **真机已连接**：`<hdc> list targets` 非空；手机已开开发者模式 + USB 调试。
6. **手机/SDK API 版本**：`<hdc> shell "param get const.ohos.apiversion"` 得手机 API；`sdk/default/sdk-pkg.json` 得 SDK API。两者须一致；项目 `build-profile.json5` 的 `compatibleSdkVersion`/`targetSdkVersion` 改成 SDK 支持的格式（API≥26 用 `"26.0.0"`，API 10–25 用 `"5.0.0(12)"` 样式）。

## 执行流程

### 1. clone 源码（绕过全局代理）
```
git -c http.proxy= -c https.proxy= clone --depth 1 https://github.com/xiaoli8571/NekoBox4Harmony.git
```
（用户全局 git 代理未开时直连绕过；若仍不通试 ghproxy 镜像）

### 2. 改 bundleName
`AppScope/app.json5` 的 `bundleName` → 步骤 4 提取的调试 bundleName。

### 3. 改 build-profile.json5
- `signingConfigs[0].material`：`certpath`/`profile`/`storeFile` 改成本机 `~/.ohos/config` 下对应材料的绝对路径；`keyPassword`/`storePassword` 用**加密格式**（build-profile.json5 原样从用户既有项目拷贝，hvigorw 能解）。`keyAlias`/`signAlg` 保持。
- `compatibleSdkVersion`/`targetSdkVersion` → 步骤 6 确定的值。
- 保留 `externalNativeOptions`（CMake native 构建）、`buildModeSet`。

### 4. 改 hvigor 依赖路径
`hvigor/hvigor-config.json5` 的 `@ohos/hvigor` / `@ohos/hvigor-ohos-plugin` 的 `file:` 路径 → 本机 `<command-line-tools>/hvigor/hvigor` 与 `.../hvigor-ohos-plugin`（仓库默认是 Windows 路径）。

### 5. 补 node_modules 软链
```
mkdir -p node_modules/@ohos
ln -sfn <cmdline>/hvigor/hvigor-ohos-plugin node_modules/@ohos/hvigor-ohos-plugin
ln -sfn <cmdline>/hvigor/hvigor node_modules/@ohos/hvigor
ln -sfn ../hvigor <cmdline>/hvigor/hvigor-ohos-plugin/node_modules/@ohos/hvigor
```

### 6. 移除个人调试 profile 拿不到的系统级权限
`entry/src/main/module.json5` 的 `requestPermissions` 里删 `ohos.permission.READ_PASTEBOARD`（user_grant + 需 ACL，个人调试 profile 无授权 → 安装报 9568289）。保留 INTERNET/GET_NETWORK_INFO/KEEP_BACKGROUND_RUNNING/GET_BUNDLE_INFO/DETECT_GESTURE。代价：「从剪贴板导入」会弹系统逐次确认框，功能仍可用。

### 7. 关闭「测速时绕过隧道」默认值
`entry/src/main/ets/model/Profile.ets` 的 `testBypassTunnel: boolean = true` → `false`。原因：开启会 `SO_BINDTODEVICE` 绑定物理网卡，鸿蒙三方应用无此权限，测速全红报「绑定物理网卡失败」。

### 8. 移除 macOS 隔离属性（仅 macOS）
```
xattr -rd com.apple.quarantine <command-line-tools>/
```
不解会导致 SDK 自带 ninja/cmake 被 SIGKILL（exit 137），CMake 配置阶段 `Subprocess killed`。

### 9. 构建
```
export PATH="<jdk17>/bin:$PATH"
<command-line-tools>/bin/hvigorw --mode module -p product=default assembleHap --no-daemon
```
产物：`entry/build/default/outputs/default/entry-default-signed.hap`（hvigorw 用加密密码自动签名）。

### 10. 安装
```
<hdc> install entry/build/default/outputs/default/entry-default-signed.hap
```
若报 9568289 grant request permissions failed → 回步骤 6 检查是否还有受限权限未删。
若旧设置残留导致测速异常 → `<hdc> uninstall <bundleName> && <hdc> install <hap>` 清数据重装。

### 11. 订阅导入（手机直连境外订阅会超时）
- 本机 `curl` 抓订阅内容（clash yaml 或 sing-box json）。
- clash yaml 需转 sing-box JSON：把 `proxies` 映射成 `outbounds`（ss→shadowsocks、vless+reality-opts→vless.tls.reality、加 selector + direct + tun inbound + route）。
- 本机起 `python3 -m http.server 8000` 托管 JSON，手机 NekoBox 加订阅 URL `http://<本机局域网IP>:8000/singbox-sub.json`。
- 导入完成后 `pkill -f "http.server 8000"`。

## 排障清单

| 现象 | 根因 | 修复 |
|---|---|---|
| hvigor `Cannot find module '@ohos/hvigor'` | hvigor-config 里 Windows 路径 | 步骤 4 |
| `Cannot read properties of undefined (reading 'getName')` | compatibleSdkVersion 格式与 SDK 不匹配 | 步骤 3 + 步骤 6 版本格式 |
| `api version parameter is illegal` | API≥26 写了 `"7.0.0(26)"` 旧格式 | 改 `"26.0.0"` |
| CMake `Subprocess killed` / ninja exit 137 | macOS quarantine | 步骤 8 |
| `parseAlgParameters failed` | java 1.8 | 步骤 2 用 JDK17+ |
| 安装 9568289 | 受限权限无 ACL | 步骤 6 |
| 测速「绑定物理网卡失败」 | testBypassTunnel 默认开 | 步骤 7 |
| 订阅获取超时 | 订阅域名被墙 | 步骤 11 本机中转 |

## 限制（须如实告知用户）

- **需鸿蒙开发者调试签名环境**：纯血鸿蒙不允许装未签名 HAP。个人开发者须先在 DevEco Studio 或 AGC 生成调试证书 + profile（绑定 bundleName + 设备 UDID）。无此环境者不能自助安装。
- **VPN 扩展授权**：VpnExtensionAbility 在调试 profile 下通常可用；若个别设备/系统版本需 ACL 授权，个人开发者可能拿不到，届时 VPN 起不来——这是系统层限制，非本 skill 能解。
- **改了 bundleName/权限/默认设置**：与上游发布版有差异，仅自用；勿把改后的包提交 AGC 审核（会撞 bundleName 与原作者）。