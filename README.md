# android-screen-automation（独立副本）

把一台 Android 手机变成"可被脚本作答"的终端：**无 OCR、纯像素 + adb**，
读题/判选中态/找翻页按钮全靠截图分析，模型只负责"看图给答案"。

这是从实际项目里固化出来的版本 —— 踩过的坑都留在 `references/pitfalls.md` 里，
那些坑**全部是静默失败型**（脚本照跑、结果照出，但答案是错的）。

## 依赖

| 组件 | 说明 |
|---|---|
| `adb` | Android platform-tools。脚本用 `--adb` 或 `$ADB` 指定；写成裸名则靠 PATH |
| Python 3 | 脚本用 `--py` 或 `$PY` 指定；写成裸名则靠 PATH |
| Pillow | `pix.py` 依赖：`pip install Pillow` |

Windows 上路径可以直接写 `C:/...`（脚本里用四层 `cd` 归一化），WSL 亦可。

## 五阶段流程

```bash
# 开跑前先过模式闸门: 复用 key.txt 还是从零遍历截图?
#   key   = 跳过读题, 省约 33k token (但仍要跑 collect 采导航坐标)
#   fresh = 遍历截图 -> 思考出键 -> 倒序作答
bash scripts/start.sh            # 交互式; 或 --mode key / --mode fresh

# 0. 环境监测：设备/屏幕/唤醒/前台 App/三项耗时/节点数/DevTools/路线判定
bash scripts/env-probe.sh --serial <SERIAL>

# 1. 选项点击尝试：自动找出"该点哪一列 x"  (输出 TAPX=...)
bash scripts/tap-probe.sh --serial <SERIAL> --marker blue

# 2. 采集遍：逐页截图 + 记录每页的选项纵坐标与翻页按钮纵坐标 -> index.txt
bash scripts/collect-run.sh --serial <SERIAL> --dir work --pages 196 --marker blue

# 3. （由模型完成）看采集下来的裁剪图，产出 key.txt：每行 "<页号> <字母>"
#    多选写成 "<页号> ACD"

# 4. 作答遍：幂等自校正，只点该点的
bash scripts/answer-run.sh --serial <SERIAL> --dir work --key work/key.txt \
     --from 1 --to 196 --marker blue
```

作答遍**可以反复跑**：状态已对就不点。第二遍报 `修正 0` 是收敛的证明。

## 输出约定

`pix.py` 的所有输出都是"给其它脚本读"的：

```
READ x0 y0 x1 y1 scale          # 自适应裁剪参数（整页压进 512x512 = 1 个视觉 tile）
NAV ycenter=1996 x_next=1162 x_prev=277
MAP device_x = 48 + px/0.3327 ; device_y = 451 + py/0.3327
OPTS A=1044 B=1274 C=1505 D=1736
SEL  A=0 B=0 C=0 D=1            # 1 = 实心(已选中)
FP   <md5>                      # 页面指纹, 用来判断"这一跳到底动没动"
RED  x=703 y=313                # 红色高亮圆点(题号条当前题)
```

## 六条铁律（违反必错，且错得看不出来）

1. **先探测再动手** —— 先用 `env-probe.sh` 判定路线（截图 / 可访问性树 / DevTools）。
2. **绝不盲点** —— 屏幕上没有蓝色的东西 ≠ 蓝色按钮是灰的 ≠ 按钮还在原位。候选点击必须**验证效果**。
3. **翻页必点** —— 无论本题答没答上，本页的翻页必须点。跳过翻页 = 整卷错位 = 全错。
4. **先清后设** —— 单选是 radio 语义：点新选项会自动清掉旧的。按"点击前的快照"先加后清，会把答案顶掉。
5. **不重复点已选项** —— 多选的 checkbox 语义下，重复点同一项 = **取消**选中。
6. **自校正只保证"手机 == 你的 key"**，**不保证 key 是对的**。key 的正确性只能靠人工/官方答案核对。

## 已知边界

- `NAV` 的纵坐标**每页都在变**（实测跨 1399~2313）→ 绝不能缓存一个固定 y 反复用。
- 导航按钮纵向容差约 **±100px**，相邻页可共用 y；跨度超了就必然丢。
- 连点翻页的最快可用间隔实测 **≈400ms**（250ms 开始丢）。
- 单页成本 ≈ `screencap 564ms` + 像素检测（纯 Python 逐像素约 2200ms，numpy 化可降到 ~500ms）+ 点击与等待。
- 锁屏/息屏时 `uiautomator dump` 会失败，**别当成"这个页面没有可访问性内容"**，先看唤醒状态。

## 目录

```
安装教程.txt              在另一台电脑上从零安装的完整教程(依赖/路径/故障排查)
key.txt                  本卷答案键(196 行, 题号->答案) —— 换设备可直接复用
key_status.md            key.txt 的可信度分层 + 换设备复用说明
SKILL.md                 技能主文档(带 frontmatter, 可放回 ~/.dsh/skills/)
scripts/start.sh         入口 / 模式闸门(先问用不用 key.txt)
scripts/lib.sh           ADB/PY 解析、shot/tap/batch_taps/field
scripts/pix.py           裁剪/选项+选中态/指纹/找红点
scripts/env-probe.sh     阶段0
scripts/tap-probe.sh     阶段1
scripts/collect-run.sh   阶段2
scripts/answer-run.sh    阶段4
scripts/nav-bench.sh     翻页速度基准
references/two-routes.md 路线判定
references/pitfalls.md   23+ 条实测坑
```

---

## 可选：装成 DSH 插件（`dsh-phone-tools/`）

本文件夹里的脚本可以直接 bash 跑，够用。另有一个**可选的 DSH 插件**，作用只有一件：
把 6 个脚本挂成**常驻工具**，这样在新会话里它们一定出现在工具表里 ——
不再依赖 skill 的 description 被"想起"（description 一旦被 YAML 引号截断，
skill 会静默消失且不报错，这是踩过的坑）。

```bash
# 装（把整个文件夹拷到目标机后，在 DSH 会话里）
plugin_manager(action="install_bundle", target="link:<dsh-phone-tools 绝对路径>")
# 然后必须重启 DSH
```

装完会多出：`phone_probe` / `phone_gate` / `phone_read` / `phone_tap` / `phone_collect` / `phone_answer`。

插件**不含第二份实现** —— 逻辑仍只有本文件夹 `scripts/` 里那一份；它只负责
"解析路径 → 起 bash → 递回输出"，并且会自动认出同级的 `scripts/`，所以整个文件夹
拷到哪都能用。三个安装坑（peerDependencies 会挡住激活、必须 `export const inject = ['tools']`、
改代码不会自动重载）详见 `安装教程.txt` 第九节。
