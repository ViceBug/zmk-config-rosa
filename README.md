# Keymap Editor - Rosa

This is a customization of the ZMK config for the Rosa keyboard with machine
readable layout and keymap definitions for use with @nickcoutsos' [keymap-editor](https://github.com/nickcoutsos/keymap-editor) tool.

![Screenshot](https://i.imgur.com/6Ny3WK8.png)

# Usage

* Navigate to the [keymap-editor](https://nickcoutsos.github.io/keymap-editor/) web app.
* In the `Source` drop-down menu, select GitHub.
* For the repository, input this repository's location: `pierrechevalier83/zmk-config-rosa`
* For the branch, select `main`.
* Modify the keymap as desired.
  * Refer to the [zmk documentation](https://zmk.dev/docs) for a description of the various [behaviours](https://zmk.dev/docs/behaviors/key-press) and [codes](https://zmk.dev/docs/codes).
* Once you're happy with the changes, click "Commit Changes" at the bottom right of the page.
* Give it a few minutes (should be less than 10 minutes) to build the new firmware. There will be a button at the bottom right of the page that you can use to downlad the firmware.
* Download the firmware and extract it somewhere.
* Plug your rosa to your computer and quickly double tap the reset button.
  * The left LED should be flashing blue, indicating the keyboard is in bootloader mode.
  * The keyboard should appear as a new drive on your system, with name: `NICENANO`
* As superuser, copy the `rosa_nice_nano_v2.uf2` file that you extracted from the downloaded zip file to the root of the `NICENANO` drive.
* Type a few letters with the Rosa. It should now be using the updated keymap.

# Usage for other users than yours truly

For the keymap-editor to do its job, it will need write access to the github repo it's pointing at.

If you want to use the keymap-editor for your Rosa keyboard and you are not this repository's author, please fork this repo first and substitute any reference to `pierrechevalier83` in the README with you own username.

# 本地构建（Linux）

> 说明：当前实际构建目标是 `h65` shield + `nrfmicro_13_52833` 板（见 `build.yaml`）。
> 以下步骤与 CI 完全等价：CI 使用 zmk `v0.3` 的 `build-user-config.yml` +
> `zmk-build-arm:stable` 容器，其对应版本组合为 **Zephyr 3.5 + Zephyr SDK 0.16.9 + CMake 3.31.6**。

## 1. 安装基础工具

```bash
pip install --user west ninja 'cmake==3.31.6'
export PATH="$HOME/.local/bin:$PATH"
```

> CMake 必须用 3.x（建议锁 3.31.6，与 CI 容器一致）。CMake 4.x 移除了对旧
> `cmake_minimum_required` 的兼容，zephyr 3.5 的部分第三方模块会配置失败。

## 2. 初始化 west 工作区并拉取依赖

```bash
git clone git@github.com:ViceBug/zmk-config-rosa.git
cd zmk-config-rosa
west init -l config          # 使用 config/west.yml（zmk 锁定 v0.3 → zephyr v3.5.0+zmk-fixes）
west update
west zephyr-export
pip install --user -r zephyr/scripts/requirements-base.txt
```

注意：`west update` 会把 zmk、zephyr、modules 等克隆到本仓库目录下（约 3.5 GB），
这些目录不要提交，也不属于 git 管理，排查完可整体删除。

## 3. 安装 Zephyr SDK 0.16.9（含 ARM 工具链）

```bash
wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.9/zephyr-sdk-0.16.9_linux-x86_64.tar.xz
tar xJf zephyr-sdk-0.16.9_linux-x86_64.tar.xz -C "$HOME"
export ZEPHYR_SDK_INSTALL_DIR="$HOME/zephyr-sdk-0.16.9"
"$HOME/zephyr-sdk-0.16.9/setup.sh" -t arm-zephyr-eabi -h -c
```

`PATH` 和 `ZEPHYR_SDK_INSTALL_DIR` 每开一个新 shell 都要重新 export（或写进 `~/.bashrc`）。

## 4. 构建（与 CI 参数一致）

```bash
west build -s zmk/app -d build -b nrfmicro_13_52833 -- -DZMK_CONFIG="$PWD/config" -DSHIELD=h65
```

- 产物：`build/zephyr/zmk.uf2`（约 380 KB，可直接刷机）；`zmk.bin` 为 bin 格式。
- 构建成功后 Kconfig 汇总会打印 `h65` 相关配置；关键项：
  `CONFIG_ZMK_KEYBOARD_NAME="h65"`、`CONFIG_SHIELD_H65=y`、`CONFIG_ZMK_BLE=y`、`CONFIG_ZMK_USB=y`。

# 常见问题排查

**1. CI 报 `KeyError: 'qualifiers'`（`zephyr/scripts/west_commands/boards.py`）**
原因：`.github/workflows/build.yml` 引用的 ZMK workflow 与 `config/west.yml` 锁定的
ZMK 版本属于不同世代。ZMK main（zephyr 4.1 世代）的 workflow 会执行
`west boards --format "{qualifiers}"`，而 v0.3（zephyr 3.5 世代）不支持该参数。
修复：两处引用必须指向**同一个**版本 tag（当前均为 `v0.3`）。
另注意 ZMK 的 tag 都带 `v` 前缀（`v0.3.0`/`v0.3`），写 `@0.3.0` 会直接 startup_failure。

**2. CI 报 `grep: .../zephyr/.config: No such file or directory`**
这是上一步 `west build` 失败导致 `.config` 没生成（该检查步骤带 `if: always()` 会照跑）。
去日志里找 "West Build" 步骤的真实报错。本仓库已知的坑：`h65.conf` 里**不能开启**
`CONFIG_ZMK_USB_LOGGING=y`——它会 select 整条 SERIAL/UART_CONSOLE 依赖链，而
nrfmicro 板的 uart0 未启用，且其默认引脚 P0.08/P0.06 被 h65 矩阵用作列线，
Kconfig 依赖告警在 ZMK 构建中按错误处理，会直接中止构建。

**3. 升级 ZMK 版本时**
- zephyr 4.1 世代（ZMK main 及之后的 release）里，板名从 HWMv1 改为 HWMv2 风格：
  `nrfmicro_13_52833` → `nrfmicro_nrf52833`，升级时需同步改 `build.yaml`。
- 升级后重新检查 `h65.conf` 里各 `CONFIG_ZMK_*` 是否仍存在（Kconfig 选项会随版本改名）。
- 升级时务必同时改 workflow 的 `@ref` 和 west.yml 的 `revision`（见问题 1）。

**4. 本地构建报 `dtc` 相关错误**
zephyr 需要 device tree compiler。SDK 的 `setup.sh -h` 会装好；若在受限环境
（无 root/不能执行安装脚本），可从 SDK 的 hosttools 包手动提取 `dtc` 并放入 PATH
（zephyr 的 `FindDtc` 只是普通的 `find_program`，走 PATH 查找）。

