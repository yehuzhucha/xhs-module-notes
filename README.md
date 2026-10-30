# 小红书免 root 模块适配笔记

记录 2026-10 对小红书（`com.xingin.xhs`）做模块化适配的全部实测结论。
目标：去广告 + 保持登录 + 无水印下载，**手机无 root**（只能 NPatch / LSPatch 内置）。

## 一句话结论

**小红书 9.x 的补丁版登录不了，这是硬结论，别再试版本了。**
签名校验在 native 层（`libsecurebase.so`），NPatch 的 Java 层 hook 骗不过去。
社区所有能用的模块（静盒、红薯猪手、xhs-savefix）**都是给 LSPosed（root）用的**。

## 一、为什么补丁版登录不上

### 实测过程

| 版本 | 补丁方式 | 结果 |
|---|---|---|
| 9.50.0 | NPatch 默认（FPA 测试签名 + 低级绕过） | 登录失败 |
| 9.51.0 | NPatch L4 + 自定义签名 | 登录失败 |
| 9.51.0 | NPatch L5 Stealth + hidelibs | 登录失败 |
| 9.51.0 | **官方原版（未改签名）** | **登录成功** |
| 8.57.0 | NPatch L4 + 自定义签名 | 登录失败 |

提示统一是「**当前网络存在风险，请切换可靠网络后重试**」——看着像网络问题，实际是签名校验。

### 证据

```bash
# Java 层根本没有签名校验
grep -r "GET_SIGNATURES" jadx/sources | wc -l          # 0
grep -r "GET_SIGNING_CERTIFICATES" jadx/sources | wc -l # 0

# 校验在 native 层
unzip -l xhs.apk | grep libsecurebase                  # 211 KB
strings libsecurebase.so | grep -i signature           # 全空（字符串被抹）
```

NPatch 的签名绕过 hook 的是 `PackageManager`，返回假签名；
但 so 直接读 APK 签名块，拿到的是真实（被改过的）证书 → 服务端一比对就判定环境异常。

**要绕过只能逆向 `libsecurebase.so`（Ghidra + ARM64），工作量几小时，成功率不确定。**

## 二、版本与模块对应

```
小红书 9.51.0   ← 静盒 QuietBox 0.8.0（适配 9.51）
小红书 9.52.0   ← 静盒 QuietBox 0.9.0（只适配 9.52）
小红书 8.55~8.57 ← 红薯猪手 RedbookHelper 1.2.7（老版本，社区验证多）
小红书 9.33.4   ← xhs-savefix（解除"作者关闭保存"）
```

**QuietBox 用 libxposed Modern API 102** ⇒ 只能 NPatch 内嵌，LSPatch 不认（LSPatch 只支持老 API）。

## 三、QuietBox 结构（9.51.0）

```
app/src/main/java/com/quietbox/app/Feature.java     功能开关
app/src/main/java/com/quietbox/app/hooks/*.java     每个功能一个文件（20+ 个）
```

功能编号：

```
AD-01 开屏广告跳过        AD-06 详情页广告条
AD-02 信息流广告过滤      AD-07 评论带货插入
AD-03 评论广告过滤        AD-08 直播卡过滤
AD-04 广告外跳拦截        AD-09 带货卡过滤
AD-05 搜索广告过滤
EN-01 水印策略            EN-02 视频默认静音
```

### 水印链路（图片）

```
9.51.0  保存入口 ce8.i    合成水印 ce8.j
9.52.0  保存入口 jn6.i    合成水印 jn6.j
调用链  AsyncImageGalleryController.B3 → pc9.y → ce8.i.i(...) → ce8.j.a(..., disableWaterMark)
```

拦 `ce8.j.a` 的第 5 个参数 `disableWaterMark` 置 true。
**只覆盖图片，视频不走这条路。**

### 解除「作者关闭保存」

```java
// 已验证于 9.33.4
MediaSaveConfig.b()  →  false
```

## 四、小红书 APK 结构（9.51.0）

```
dex        22 个，约 230 MB
加固       无 libjiagu / libnesec / libDexHelper —— 可反编译
native     libsecurebase.so（签名校验）、libmsaoaidsec.so（MSA OAID）
混淆       R8 常规：androidx.* / com.xingin.* 类名可读，部分方法名保留
```

**服务器上的 `xhs-9.51.0-*.apk` 都是 NPatch 壳**，真包在 `assets/npatch/origin.apk`（175 MB）。
直接反编译外层只能拿到 loader stub（classes.dex 13 KB、classes2.dex 468 B 就是壳的标志）。

```bash
unzip -o -q xhs.apk "assets/npatch/origin.apk" -d .
jadx -j 4 --no-res --no-imports assets/npatch/origin.apk -d jadx
```

## 五、NPatch 补丁命令（踩过的坑）

```bash
# ❌ 默认补丁：日志出现这行就是错的
# Signing apk with built-in FPA keystore (V2+V3, minSdk 28)

# ✅ 正确：自定义 keystore + 高绕过等级
java -jar npatch.jar -l 4 -m module.apk \
  -k my.bks PASS alias PASS -o out/ base.apk
```

**`-k` 只认 BouncyCastle 的 BKS**：

- 给 JKS → `Wrong version of key store`
- 密码错 → `KeyStore integrity check failed`

JKS → BKS 转换：

```bash
keytool -importkeystore -srckeystore my.jks -srcstoretype JKS -srcstorepass PASS \
  -destkeystore my.bks -deststoretype BKS -deststorepass PASS \
  -providerclass org.bouncycastle.jce.provider.BouncyCastleProvider \
  -providerpath npatch.jar
```

`-l` 等级：0 无 / 1 基础 / 2 高 / 3 极端 / 4 Seccomp / 5 Stealth（**5 必须配 `--manager`，不能内嵌模块**）。

## 六、老版本 APK 从哪下

apkpure / apkmirror / apkfab 全部被 Cloudflare 403。能用的只有一个：

```bash
UA="Mozilla/5.0 (Linux; Android 13; Pixel 7) AppleWebKit/537.36 Chrome/122.0 Mobile Safari/537.36"
curl -sL -A "$UA" -e "https://aiting.com/xiao-hong-shu/com.xingin.xhs/download/8.57.0" \
  -o xhs-8570.apk "https://dl.37790.cn/apk/com.xingin.xhs_58049f1e.apk"
# 不带 Referer 会 403
```

## 七、如果一定要「登录 + 去广告」

只剩两条路：

```
① 虚拟机 + root + LSPosed + 官方小红书（原签名）→ 登录正常，模块可用
   代价：占空间、耗电、小红书有虚拟环境检测

② 手机 root（LSPosed）→ 最干净，但用户不愿意
```

**免 root 补丁这条路，在小红书 9.x 上已经封死。**

## 八、无水印下载（不需要登录）

不改 App 也能做：`XHS-Downloader2` 之类的工具，输入笔记链接直接拿无水印原图/视频。

## 九、文件位置（服务器）

```
原包      /home/ubuntu/xhs/xhs-origin.apk
反编译    /home/ubuntu/xhs/jadx/
QuietBox  /home/ubuntu/quietbox/（git clone yigenhuobah/quietbox）
8.57 包   /home/ubuntu/xhs/xhs-8570.apk
成品包    /var/www/dl/xhs-8.57.0-redbookhelper-npatch.apk
          /var/www/dl/xhs-9.51.0-arm64-*.apk
```
