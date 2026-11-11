# avium-build-odin

只负责构建 **AviumUI 16.2（Android 16 / lineage-23.2 基线）** 的 **odin**：
Xiaomi MIX 4（型号 2106118C，SM8350）。

仓库内不含任何 ROM 源码，只有 CI 与设备清单；源码在 runner 上由 `repo sync` 拉到
`$HOME/android/avium-odin`。

```
.
├── .github/workflows/build.yml   # 单机型 workflow，机型写死 odin
├── local_manifests/odin.xml      # 设备树/内核/vendor 清单（均为第三方）
└── README.md
```

> ⚠️ LineageOS 官方**没有** odin 设备树。本仓库使用的 device / kernel / vendor 全部来自
> 第三方维护者 **Inkypen79**（`lineage-23.2` 分支），出问题需要自行修设备树。

## 1. 在 GitHub 上先建空仓库

New repository → 名称 `avium-build-odin` → 可见性自选 → **不要**勾选
Add README / .gitignore / license。

## 2. 放入文件并推送

方式 A（先 clone 空仓库）：

```bash
git clone https://github.com/<你的账号>/avium-build-odin.git
cd avium-build-odin
cp -r <本目录>/.github ./
cp -r <本目录>/local_manifests ./
cp <本目录>/README.md ./
git add .github local_manifests README.md
git commit -m "init: AviumUI 16.2 odin self-hosted build"
git push origin main
```

方式 B（本地直接 init）：

```bash
cd <本目录>                      # 即 outputs/avium-build-odin
git init -b main
git add .
git commit -m "init: AviumUI 16.2 odin self-hosted build"
git remote add origin https://github.com/<你的账号>/avium-build-odin.git
git push -u origin main
```

`<本目录>` 指 `outputs/avium-build-odin`。push 需要**你本人**的凭据
（HTTPS 用带 `repo` 权限的 PAT，或配置 SSH key）。

## 3. runner 前提

| 项 | 要求 |
|---|---|
| 系统 | Ubuntu 22.04 / 24.04 x86_64 |
| 磁盘 | ≥350 GB 空闲（建议 500 GB；源码树 300–400 GB） |
| 内存 | ≥32 GB |
| CPU | ≥16 核 |
| 网络 | github.com、android.googlesource.com、**gitlab.com** |
| 权限 | 可 sudo（装依赖） |
| JDK | 不用手装，AOSP 16 自带 `prebuilts/jdk` |

注册：本仓库 Settings → Actions → Runners → New self-hosted runner → Linux/x64，然后

```bash
mkdir -p ~/actions-runner && cd ~/actions-runner
curl -o actions-runner.tar.gz -L https://github.com/actions/runner/releases/download/v<版本>/actions-runner-linux-x64-<版本>.tar.gz
tar xzf actions-runner.tar.gz
./config.sh --url https://github.com/<你的账号>/avium-build-odin --token <页面给出的一次性 token>
sudo ./svc.sh install && sudo ./svc.sh start
```

默认 `runs-on: self-hosted`。若想本仓库只用指定机器，把 `build.yml` **第 38 行**
改成 `runs-on: [self-hosted, odin]`，并在该机器上给 runner 加 `odin` 标签。

## 4. 触发与产物

Actions → Build AviumUI (odin) → Run workflow（可选 `clean_build`）。

- 产物：`AviumUI-odin-<run_number>`，保留 7 天
- 内容：`*.zip`、`boot.img`、`dtbo.img`、`vbmeta.img`、**`vendor_boot.img`**（GKI 机型）、`build-info.txt`
- 同一仓库的两次运行由 `concurrency: avium-odin` 串行化
- 编译日志：job 里 `build.log`

## 5. 本机型说明

- 清单三项均为第三方：`Inkypen79/device_xiaomi_odin`、`Inkypen79/kernel_xiaomi_odin`、
  `Inkypen79/vendor_xiaomi_odin`，分支 `lineage-23.2`；另有 `LineageOS/android_hardware_xiaomi`。
- 设备树 `BoardConfig.mk` 规定 `TARGET_KERNEL_CLANG_VERSION := r574158`，
  `vendorsetup.sh` 会在 lunch 阶段从 `gitlab.com/inkypen` 拉取到
  `prebuilts/clang/host/linux-x86/clang-r574158`。**runner 必须能访问 gitlab.com**；
  离线环境需自备该目录并改 `vendorsetup.sh`。
- 该设备树没有 `lineage.dependencies`，kernel/vendor 完全依赖 `odin.xml` 指定。
- 变体为 `userdebug` + test-keys；**GMS 默认开启**，见下节。

## 6. GMS（默认开启，`WITH_GMS=true`）

- **来源**：`MindTheGapps/vendor_gapps`，分支 **`baklava`**（Android 16），落到 `vendor/gapps`。
- **体积**：git 仓约 1.25 GB，checkout 约 1.7 GB（APK 约 440 MB）；产物 zip 约 +250–400 MB，
  system 占用约 +0.8–1.2 GB。
- **打开方式**：workflow 顶层 `env: WITH_GMS: "true"`；设备树 `lineage_odin.mk` 里本来就是
  `ifeq ($(WITH_GMS), true)` 条件 inherit `vendor/gapps/arm64/arm64-vendor.mk`，
  **设备树无需改动**。`breakfast` 后用 `get_build_var WITH_GMS` 校验，非 true 直接 fail。
- **编译时间**：首次约 +10–20 分钟，增量约 +3–8 分钟。
- **首次开机**：多出 Google 账号登录 / SetupWizard，需要联网。

## 7. 风险

- **未认证**：可能提示 "This device isn't Play Protect certified"，需注册 GSF ID 才消失。
- **签名**：GApps APK 自带签名，构建不需要额外密钥；ROM 仍为 test-keys。
- **上游不支持**：odin 的设备树由第三方维护，GApps 组合未经验证；开机异常先看
  `logcat` 的 `privapp-permissions`。
- **分区**：`brunch` 报 `not enough space` 时需改设备树 `BOARD_SUPER_PARTITION_SIZE`。
