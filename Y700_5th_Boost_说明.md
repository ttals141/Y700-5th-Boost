# Y700 5th Boost —— 模块说明（Legion Y700 五代 GPU 1200MHz + 温度墙后移）

> 模块：`Y700_5th_Boost_v1.10.zip`
> 作者：暮光TvT & DeepSeek
> 适用：联想拯救者 Y700 五代（TB323FU / SM8850P / ZUXOS 2.0.12.088 / Android 16）
> root：KernelSU 3.3.1（uapi 5），hybrid_mount 元模块

---

## 1. 这台机器原本被限制在哪

### 1.1 三层模型

游戏生态里有三层独立的限制，作用在不同地方，互相不能替代：

| 层 | 由谁执行 | 在哪配置 | 改它管什么 | 模块是否触碰 |
|---|---|---|---|---|
| **L1 游戏配置** | `com.zui.tassistent`（游戏助手）读 `gameperfconfig.xml` | `/system/etc/gameperfconfig.xml` | **每个游戏的 GPU 上限**（`max_pwrlevel`） | 改 |
| **L2 厂商温控** | `thermal-engine-v2` 读 `thermal-engine_gpu_*.conf` | `/vendor/etc/` | **按机身温度压 GPU 上限**（`thermal_pwrlevel`） | 改（只挪阈值） |
| **L3 内核温控** | 内核 thermal 子系统 | `/sys/class/thermal/` | 105℃ 以上的硬保护 | 不动（兜底） |

**为什么 L1 和 L2 要分开看**：`max_pwrlevel` 只由 L1 写，`thermal_pwrlevel` 只由 L2 写，两者互相独立、同时生效，取更严格的那个。所以：

- 只改 L1 → GPU 上限能到 1200MHz，但机身一到 48℃ 照样被压回 902MHz；
- 只改 L2 → 温度墙推后了，但 `max_pwrlevel` 还卡在 3（1000MHz），照样上不去。

必须两层一起改，才有"持续跑满"的效果。

### 1.2 L1 具体长什么样

`gameperfconfig.xml` 的结构是：

```
<Game name="鸣潮" pkg="com.kurogame.mingchao,...">
  <Mode name="Savage" id="2">          ← 野兽模式
    <Policy>
      <TempLevel level="1" temp="0">   ← 温度档（机身温度 >= 0℃ 开始生效）
        <item type="freq" name="Gold">5_-1</item>    ← CPU 大核
        <item type="freq" name="Prime">4_-1</item>   ← CPU 超大核
        <item type="freq" name="Gpu">7_3</item>      ← GPU：只允许第 3~7 档
      </TempLevel>
      ...
```

`Gpu` 那两个数字是**频率表的下标**。GPU 频率表是**倒序**的 18 档：

| 下标 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|---|---|---|---|---|---|---|---|---|---|---|
| MHz | 1200 | 1100 | 1050 | 1000 | 902 | 826 | 726 | 646 | 578 | 500 |

| 下标 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 |
|---|---|---|---|---|---|---|---|---|
| MHz | 461 | 422 | 382 | 342 | 282 | 222 | 191 | 160 |

所以鸣潮野兽模式的 `7_3` = 只允许 {1000, 902, 826, 726, 646} MHz —— **上限 1000MHz**。这就是"锁 1000"的来源。

`0_-1` 表示"从第 0 档到最后"，对 CPU（**升序**表）就是全放开；但 GPU 表是倒序的，全放开的写法应该是 `17_0`（原厂自己就是这么写的，共 18 处）。

### 1.3 L2 具体长什么样

`/vendor/etc/thermal-engine_gpu_2.conf`：

```
[SS-GPU-2-0]
algo_type        ss
sampling         2000
sensor           quiet-therm       ← 机身外壳温度
device           gpu
set_point        48000             ← 48.0℃ 时开始压
set_point_clr    46000             ← 降到 46.0℃ 才恢复（2℃ 滞后）
device_max_limit 902000000         ← 压到 902MHz
```

四段阈值对应四个水位，是**阶梯**，越热压得越狠。`device_max_limit` 的值是写死的目标频率，**模块一个字都没改**——只把"什么时候开始压"往后推。

四套配置的用途（`thermal_user_case.conf` 决定用哪套）：

| case | 场景 | 加载的 GPU 配置 |
|---|---|---|
| 0 | 没有游戏在跑（默认） | `gpu_0` |
| 100 | 省电模式游戏 | `gpu_2` |
| 200 | 均衡模式游戏 | `gpu_2` |
| 300 | **野兽 / 超神模式游戏** | `gpu_2` |
| 400 | 小窗游戏 | `gpu_2` |
| 5 | 本机从不触发 | `gpu_5` |
| 700 | 本机从不触发 | `gpu_1` |

**实测确认**：打鸣潮时 ThermalEngine 打印 `Found user case 300 ... gpu_2` → 加载 `thermal-engine_gpu_2.conf`。跑两个多小时日志，`gpu_1`（case 700）与 `gpu_5`（case 5）**一次都没被加载过**。

### 1.4 L3 为什么不动

内核 `quiet-therm` 的 trip point 是 125℃，`gpuss-0` 是 105℃/125℃/135℃。这些是"再热就真的危险了"的兜底，平时碰不到。改它们没有收益，只有风险。

---

## 2. 模块改了什么

### 2.1 gameperfconfig.xml —— 430 行

| 位置 | 原厂 | 模块 | 行数 |
|---|---|---|---|
| `Savage` / `SR_Savage` / `FI_Savage` / `SR_FI_Savage` 四种模式的 `Gpu` | 各种值（`7_3`、`12_4`、`10_7`…） | **`17_0`** | 414 |
| 和平精英 16 个 `<Scene>` 的 `Gpu` | `10_7`（646MHz） | **`17_0`** | 16 |

合计 **430 行**，全部是 `Gpu` 行，**一个非 GPU 字节都没动**。

`17_0` = 允许第 0~17 档全开 = **160 ~ 1200MHz**。上限到顶，下限也放到最低档。

**没有改**：`Balance`（均衡）、`PowerSave`（省电）、`chaoshen`（超神，本来多数游戏就是 1200）、王者荣耀两个 `0_0` 的 Scene、所有 `Gold`/`Prime`（CPU）行、`FpsAdjustLevel`/`JankAdjustLevel`、`<BindCore>`、`<PerfHint>`。

结构统计与原厂完全一致：`Game 46 / Mode 237 / TempLevel 936 / 频点 2862 / Scene 18`。

### 2.2 四个 thermal-engine_gpu_*.conf —— 28 组阈值（56 行）

只改 `set_point` 与 `set_point_clr`，**`device_max_limit` 不动**：

| 文件 | 档 | 原厂 | 模块 | 压到 |
|---|---|---|---|---|
| `gpu_0` | 1 | 44.0℃ | **55.0℃** | 902MHz |
| | 2 | 46.0℃ | **58.0℃** | 646MHz |
| | 3 | 48.0℃ | **61.0℃** | 539MHz |
| | 4 | 50.0℃ | **64.0℃** | 222MHz |
| `gpu_2` | 1 | 48.0℃ | **59.0℃** | 902MHz |
| | 2 | 50.0℃ | **62.0℃** | 726MHz |
| | 3 | 52.0℃ | **65.0℃** | 726MHz |
| | 4 | 54.0℃ | **68.0℃** | 726MHz |
| `gpu_1` | 1 | 58.0℃ | **69.0℃** | 443MHz |
| | 2 | 60.0℃ | **72.0℃** | 222MHz |
| `gpu_5` | 1 | 25.0℃ | **36.0℃** | 1000MHz |
| | 2 | 40.0℃ | **52.0℃** | 1000MHz |
| | 3 | 48.0℃ | **61.0℃** | 1000MHz |
| | 4 | 50.0℃ | **64.0℃** | 222MHz |

`set_point_clr` 始终 = `set_point - 2000`（2℃ 滞后），自动跟着平移。

`gpu_1`（case 700）和 `gpu_5`（case 5）在本机从不会被加载，改它们纯粹是"以防万一"。

### 2.3 游戏助手的私有副本

`/data/data/com.zui.tassistent/files/gameperfconfig.xml` 是游戏助手自己抄的一份**快照**（开机后约 70 秒生成，之后跨重启保留、不再重读 `/system`）。

它才是真正被读的那份——只改 `/system` 没用。模块会把它同步成模块版，属主/权限/上下文保持不变（`system:system 0600`）。

### 2.4 开机时做的事（post-fs-data.sh）

```
init -> 挂 /data -> 【post-fs-data：模块脚本】 -> class main（thermal-engine 启动）
     -> service.sh -> boot_completed
```

`post-fs-data` 这个时刻**上一次开机的挂载已经全部消失**，所以读到的 `/system`、`/vendor` 内容必然是**固件原文件**。对 5 个目标逐个判断：

| 当前文件是什么 | 处理 |
|---|---|
| md5 == `stock.md5` 里记的原厂值 | **备份**到 `/data/adb/y700g5_backup/` -> 挂载模块文件 |
| md5 == 模块文件自己 | 直接挂载（覆盖安装时的幂等） |
| 都不是（固件被 OTA 换过） | **备份**（另存）-> 就地重建 -> 挂载重建结果 |
| md5 == `accepted.md5` 里记的值（上次 OTA 后已经重建过） | 直接挂载重建结果（有 `build/` 缓存就用缓存） |
| 被别的模块抢先挂载了（设备号不同） | **SKIP 让位**，不动 |

**重要（v1.10）**：第三行那个「就地重建」的结果会**写回模块自己的文件**（旧版备份成 `build/<key>.prev`），而不是只挂一个临时的 `build/<key>`。原因是 KernelSU 的 `/system/etc`、`/vendor/etc` overlay 在 `post-fs-data` **之后**才挂上，会盖住我们做的逐文件 bind mount —— 平时两者内容一样看不出差别，但 OTA 后如果不写回，系统读到的仍是模块里的旧文件，重建等于白做。现在 bind mount 只作为「没有 overlay 时」的兜底。

最后一个分支靠"比较目标路径与上一级目录的设备号"判断：`/system` 与 `/vendor` 是只读分区，路径号没变就说明没被覆盖。

结果写进 `mount.log`，并写 `xml_state`（`ok` / `skip`）给下面用。

`accepted.md5` 是 v1.8 新增的「已确认固件」记录：OTA 后第一次成功重建时，把那 5 个文件当时的新 md5 记进去。之后每次开机先查它——命中就直接沿用对应的重建结果，**不再重新备份、也不再打 `UPDATED`**，所以 `mount.log` 里一次 OTA 只会留一条记录。文件在 `/data/adb/modules/y700g5_gpu1200/accepted.md5`，格式 `<md5>  <xml|gpu_0|gpu_1|gpu_2|gpu_5>`。

### 2.5 service.sh（开机 45 秒后）

- `xml_state` 不是 `ok` -> 直接退出，不碰副本（固件变了就不要再把旧配置塞进去）；
- 否则比对副本与在用 XML，不同就 `cp -f`（post-fs-data 阶段 `/data/data` 还没挂上，取不到副本，只能靠这一步兜底）；
- 最后把 `tools.sh` 和 `lib/rebuild.sh` 复制到 `/sdcard/Y700G5_工具/`。

**只跑一次，不常驻、不轮询、不写 sysfs。**

### 2.6 工具箱 /sdcard/Y700G5_工具/tools.sh

放在 sdcard 上，卸载模块后依然存在（模块目录会被整个删掉，所以脚本必须自我复制出来）。

| 菜单 | 作用 |
|---|---|
| 1 状态 | 版本、`xml_state`、在用 XML、5 个目标的 当前md5/原厂md5/判定、助手副本、`max_pwrlevel`、备份清单 |
| 2 导出备份 | `/data/adb/y700g5_backup/*.orig` -> `/sdcard/Y700G5_工具/backup/` |
| 3 清理残留 | 把助手副本还原成原厂（**卸载模块后**用这个） |
| 4 重建 | 系统更新后手动重建一版到模块目录 |
| 5 帮助 | 命令与文件说明 |

### 2.7 明确没有改的东西

- **CPU 调频 / 调度**：`scaling_governor`、`sched_*`、cpuset、`thermal-engine_cpu_*`、`<BindCore>`、`<PerfHint>` —— 全都没碰。
- **触控**：`gameperfconfig.xml` 里没有任何触控标签；触控设置实际在 `/data/data/com.zui.game.service/shared_prefs/GameHelperSp.xml`（`key_toucht_adjust`、各游戏的 `click_sensitive`、边缘防误触），模块脚本引用 `game.service` 的次数是 **0**。
- **内核 trip point**：不动。
- **build.prop / persist 属性 / sysfs**：不动。

---

## 3. 为什么这么改

### 3.1 为什么写 17_0 而不是 0_-1

GPU 表是**倒序**的，`0_-1` 字面意思是"从 0 到 -1(最后)"，方向是反的。原厂文件自己要放开 GPU 时写的就是 `17_0`（18 处），要放开 CPU 时写 `0_-1` —— 两种写法泾渭分明，不存在歧义。

v1.7 曾经改用 `7_0`（保留 646MHz 下限），想避免 GPU 掉到 160MHz 时帧率突变。但**实测证明这是错的**：下限抬到 646MHz 后 GPU 闲时也降不下去，结温长期挂在高位，反而触发 GPU 固件热管理把频率压到 1000/902/826 —— 结果 1200MHz 一次都没跑到。v1.9 起已改回 `17_0`，v1.10 沿用。

想要收窄下限（比如只放到 500MHz，避开极端轻载），把 `17_0` 改成 `9_0` 即可（`lib/rebuild.sh` 里改一处）。

### 3.2 为什么温度墙是 +11/+12/+13/+14

不是拍脑袋的固定值，而是**等比拉伸**：原来 44℃ 开始压，四档铺在 44->50℃（跨度 6℃）。现在第一档推到 55℃，四档铺到 55->64℃（跨度 9℃），每档间隔从 2℃ 拉到 3℃。

这样做的原因：机身温度是连续上升的，如果把四档全挤到 58~59℃，实际表现就是"50℃ 不压、59℃ 一下压到 222MHz"，反而更容易卡顿。拉开间隔可以让降频更平缓。

### 3.3 为什么 device_max_limit 不动

那是"压到多少"的目标值，跟"什么时候压"是两件事。模块只解决"太早压"，不解决"压得太狠"——后者改了会让高温时更烫、更耗电，收益不明确。

### 3.4 为什么不动 FpsAdjustLevel

有过一种说法：`FpsAdjustLevel` 第三个数字是 GPU 步进，会定期把 `max_pwrlevel` 压回去。

实测证伪了：四个游戏、**1951 次采样**，`max_pwrlevel` 一次都没变过。

- 鸣潮（文件里**没有** `FpsAdjustLevel`）：430 次，全 0；
- 崩坏:星穹铁道（`5,7,3`）：313 次，全 0；
- 和平精英（根节点 `2,2,4` + Savage `5,7,3`）：575 次，全 0；
- 三角洲行动（`3,6,3`）：633 次，全 0。

而且原厂自己就让"GPU 上限 1200MHz"和"`FpsAdjustLevel 3,4,3`"共存（绝区零）。结论：不动它是正确的。

### 3.5 为什么不动内核 trip point

105℃ 以上的保护是最后一道防线。模块只处理"40~60℃ 这个区间压得太早"的问题，不该动真正的高温保护。

### 3.6 为什么必须直接改游戏助手的副本

模块用 bind mount 把文件"盖"在真实分区上。但实测发现：**App 的挂载命名空间对所有后来的挂载都免疫**——`init`、`zygote`、`system_server` 能看到挂载（`mountinfo` 里有 overlay），但 `com.zui.tassistent`、`com.kurogame.mingchao` 这些 App 进程看到的一直是原厂文件（`grep -c overlay /proc/<pid>/mountinfo` = 0）。

`thermal-engine` 是 init 启动的，能看到挂载，所以 L2 那层用 bind mount 是有效的；但游戏助手读不到，只能直接改它自己的那份副本。

### 3.7 为什么备份要放在 post-fs-data，而不是安装时

`customize.sh` 在**系统运行中**执行，此时旧版模块的挂载**还活着**。读 `/vendor/etc/thermal-engine_gpu_2.conf` 拿到的是**旧模块的文件**，不是原厂——存下来标成"原厂"，备份就废了，`stock.md5` 也跟着错。

只有开机 `post-fs-data` 那一刻，上一次开机的挂载已经消失，读到的才是真正的固件文件。

### 3.8 为什么 OTA 要"让位 + 重建"

模块带的是**固定版本的配置副本**。系统更新后，真实分区换成了新固件，模块如果继续挂旧文件，就等于"用旧配置喂新系统"——新收录的游戏没有策略，格式变了还可能不被识别。


所以 v1.7 的做法是：

1. 开机发现目标文件的 md5 **既不是原厂记录值、也不是模块文件** -> 判定"固件更新了"；
2. 先把新原厂文件**备份**（另存，不覆盖旧备份）；
3. 用 `lib/rebuild.sh` 的规则**就地重建**一份新配置并挂载；
4. 重建前后都做结构自检，失败就不挂（`xml_state=skip`），退回原厂行为。

最坏情况是"模块不生效"，不会变砖。

---

## 4. 实测数据

### 4.1 改之前（原厂，野兽模式玩鸣潮）

| 项目 | 数值 |
|---|---|
| 游戏启动瞬间 | `max_pwrlevel` 0 -> **3**（1000MHz 上限） |
| 温控介入前（<=47.8℃，153 次采样） | GPU 只在 {1000, 902, 826, 726, 646}，**1000MHz 占 80%** |
| 47.97℃ | `thermal_pwrlevel` -> 1（1100MHz） |
| 48.27℃ | -> 4（**902MHz**） |
| 50.26℃ | -> 6（**726MHz**） |
| 机身峰值 | 50.36℃ |

### 4.2 改之后（v1.9/v1.10 相同，150 秒连续采样，每 3 秒一点）

| 频率 | 驻留 | 占比 |
|---|---|---|
| 1200 MHz | 11.9 s | **9.0%** |
| 1100 MHz | 28.9 s | 21.9% |
| 1050 MHz | 33.1 s | 25.1% |
| 1000 MHz | 33.7 s | 25.6% |
| 902 MHz | 16.8 s | 12.7% |
| 826 MHz | 5.9 s | 4.5% |
| 726 MHz 及以下 | 1.7 s | 1.3% |
| **>=1050 合计** | **73.8 s** | **56.0%** |
| 平均频率 | | **1029 MHz** |

- `max_pwrlevel` / `min_pwrlevel`：50 次采样**全部**是 `0` / `17`，没有一个采样点被改写
- `thermal_pwrlevel > 0`：只有 2/50（4%），每次 1 秒级
- 机身 44.5 -> 48.6℃，GPU 结温 79.5 -> 101℃

**ThermalEngine 日志决定性证据**（v1.5 时期那段 8 分钟记录）：整段没有任何 `ACTION: GPU` 行，`SS-GPU-2-0..3` 四个段全程 `disable`。对比原厂 48.27℃ 就压到 902MHz —— 温度墙确实从 48℃ 推到了 59℃。

### 4.3 一条很反直觉的结论

> 和平精英 575 次采样里 `max_pwrlevel` 全 0、`gpu_busy` 70~80%，但 `gpuclk` 只在 160~282MHz。

不是被限制了——**Adreno 的调频器把 80% 占用当目标值**，它会选"刚好让 GPU 忙到 80%"的频率：

- 和平精英：160MHz 就够 80%；
- 三角洲行动：578MHz 才够 80%；
- 鸣潮：要 1050~1200MHz 才够 80%。

所以"GPU 频率低"不等于"GPU 被卡住"。放开上限后，鸣潮立刻用到了 1200MHz，说明上限确实解除了。

### 4.4 还没解决的部分：CPU 瓶颈（v1.9 之后继续排查，v1.10 不变）

把鸣潮的「野兽」和「超神」两个模式逐行拆开对比：`ThermalSceneCode`、`PerfHint` 三个 opcode、`BindCore`、`DelayTime`、四个温度点 `0/30/44/47℃` —— **完全一样**。唯一差别是每个温档里 `Gold` / `Prime` / `Gpu` 三列的值。

| 47℃ 那一档 | Gold（大核） | Prime（超大核） | Gpu |
|---|---|---|---|
| 原厂野兽 `Savage` | `5_13` -> 最高 **1.90GHz** | `3_10` -> 最高 **2.11GHz** | `7_3` |
| 原厂超神 `chaoshen` | `5_27` -> 最高 **3.63GHz** | `3_-1` -> 最高 **4.61GHz** | `9_0` |
| **本模块（v1.9/v1.10）** | `5_13` -> **1.90GHz** | `3_10` -> **2.11GHz** | `17_0` |

也就是说：**超过 47℃ 以后，模块把 GPU 放开了，但 CPU 仍按野兽的原厂值被砍到 1.9 / 2.1GHz。**

佐证：那 150 秒里，GPU 跑 1200MHz 时 `gpu_busy` 是 82~85%，掉到 902/826MHz 时只有 79~81% —— **从来没到 99%**。GPU 不是自己不够快，是在等 CPU 喂命令。Adreno 调频器以 80% 占用为目标，CPU 供不上它就把频率降下来，看起来像"锁频"，实际是"没活干"。

所以"超神能跑满"大概率不是 `7_0` 的功劳，而是超神在 47℃ 以上把 CPU 留在满血。

**为什么没有继续改 CPU：**

1. `thermal-engine_cpu_3.conf`（野兽用的那份）阈值是 `35 / 43 / 45 / 47 / 49 / 50.5℃`，**50.5℃ 直接断崖到 384MHz**（cluster1 是 1017600）。
2. CPU 放开 -> 机身更快冲到 50.5℃ -> 撞上更狠的断崖，可能比现在更差。
3. 真要修好，得连 `thermal-engine_cpu_3.conf` 一起改，那就不是"只动 GPU"了，功耗和温度都会明显上升。

**以后想继续，按这个顺序：**

1. 先做对照实验：进游戏后把 `cpu0` / `cpu7` 的 `scaling_max_freq` 手动拉满，看 GPU 能否稳定在 1200。稳住了才说明 CPU 是瓶颈。
2. 确认后优先用「抄超神」方案——把野兽四档的 `Gold` / `Prime` 换成同一个游戏 `chaoshen` 的值，厂商自己在用，风险最低。
3. 同时评估要不要把 `cpu_3.conf` 的阈值一起后移，否则只是把断崖提前。

---

## 5. 安装

1. 把 `Y700_5th_Boost_v1.10.zip` 放进平板；
2. KernelSU 管理器 -> 模块 -> 从本地安装 -> 选这个 zip；
3. **重启**；
4. 开机后跑一遍验证（见第 8 节）。

覆盖安装不用先卸载，同一个 id 直接装即可。

---

## 6. 系统更新（OTA）后怎么办

### 6.1 v1.7 开始是自动的

开机时 `post-fs-data.sh` 会自己判断：

- **目标文件 == 原厂记录值** -> 正常，备份后挂载（日志 `OK (原厂未变，已备份)`）；
- **目标文件 == 模块文件** -> 正常（`OK (本来就是模块文件)`）；
- **都不是** -> 判定固件更新，**自动重建**（日志 `UPDATED ... (固件内容变了)` + `OK (已自动重建)`）；
- **被别的模块占用** -> 让位（`SKIP ... (被其它挂载覆盖: dev X vs Y)`）。

重建结果缓存在 `/data/adb/modules/y700g5_gpu1200/build/`，成功后才 `mount`。所以**大多数情况下什么都不用做，重启一次就好**。

### 6.2 怎么确认自动重建成功了

```bash
su -c "cat /data/adb/modules/y700g5_gpu1200/mount.log"
```

看这几点：

- 5 行都以 `OK` 开头；
- 出现 `UPDATED` 说明检测到了固件变化并重建；
- `--- 校验（init 视角）---` 下面 5 个 md5 与包里的一致；
- `cat /data/adb/modules/y700g5_gpu1200/xml_state` 是 `ok`。

如果 `xml_state` 是 `skip`，说明这一层没生效——跑一次工具箱的「1 状态」看具体原因。

### 6.3 如果自动重建失败

用工具箱（`/sdcard/Y700G5_工具/tools.sh`）菜单 4 手动重建，或者用电脑上的 `D:\Y700G5_原厂备份\rebuild\rebuild.sh`。

**电脑端重建（推荐，因为能看见完整输出）**：

```bash
# 1. 从平板抓新固件的 5 个原厂文件（模块让位状态下抓到的就是真原厂）
adb shell "su -c 'cat /system/etc/gameperfconfig.xml'" > gameperfconfig.xml
for n in 0 1 2 5; do
  adb shell "su -c 'cat /vendor/etc/thermal-engine_gpu_$n.conf'" > thermal-engine_gpu_$n.conf
done

# 2. 放进一个目录，跑重建
cd "D:/Y700G5_原厂备份/rebuild"
./rebuild.sh <放这5个文件的目录>

# 3. 产物：D:\Y700G5_原厂备份\rebuild\Y700_5th_Boost_v1.10_rebuild.zip
```

重建脚本和模块开机时用的是**同一份规则**（`template/lib/rebuild.sh`），产出一致。

### 6.4 重建也失败怎么办（兜底）

卸载模块 + 重启，系统就完全回到原厂状态。需要时再用 `D:\Y700G5_原厂备份\` 里的文件对照。

**最稳的习惯：系统更新前先卸载模块。**

---

## 7. 卸载

KernelSU 管理器 -> 模块 -> 删除 -> **重启**。

- `/system/etc/gameperfconfig.xml` 与 `/vendor/etc/*.conf` 立即恢复原厂（真实分区从未被写，模块只是挂载覆盖）；
- **但游戏助手那份副本不会自动恢复**——它已经被改成模块版，而 KernelSU 3.3.1 **不会执行** `uninstall.sh`（先删模块目录再 exec 里面的脚本，必然失败）。

所以卸载后请手动跑一次：

```bash
su -c "sh /sdcard/Y700G5_工具/tools.sh"
# 选 3（清理残留）
```

不想用脚本就三条命令：

```bash
su -c "cp -f /data/adb/y700g5_backup/xml.ce849bb0c9497faff1311f37fd2fd189.orig /data/data/com.zui.tassistent/files/gameperfconfig.xml"
su -c "chown system:system /data/data/com.zui.tassistent/files/gameperfconfig.xml"
su -c "chmod 600 /data/data/com.zui.tassistent/files/gameperfconfig.xml"
```

原厂 md5 是 `ce849bb0c9497faff1311f37fd2fd189`。

---

## 8. 验证清单

```bash
# 1. 模块是否装上
su -c "grep -E '^version|^author' /data/adb/modules/y700g5_gpu1200/module.prop"

# 2. 挂载日志（应 5 行 OK）
su -c "cat /data/adb/modules/y700g5_gpu1200/mount.log"

# 3. XML 状态（应为 ok）
su -c "cat /data/adb/modules/y700g5_gpu1200/xml_state"

# 4. 副本是否同步（应为模块 XML 的 md5）
su -c "md5sum /data/data/com.zui.tassistent/files/gameperfconfig.xml"
#   模块 XML 的 md5 是 8e82646c5e38f1e1fa3144f82a07a7d1

# 5. 进游戏后读上限（应为 0）
su -c "cat /sys/class/kgsl/kgsl-3d0/max_pwrlevel"

# 6. 看 GPU 有没有真的上过 1200MHz
su -c "cat /sys/class/kgsl/kgsl-3d0/gpu_clock_stats"
#   第一行 1200000000 后面的数字（单位微秒）明显变大就对了
```

---

## 9. 文件清单与 md5

### 模块内文件（v1.10，共 17 个）

| 文件 | md5 |
|---|---|
| `system/etc/gameperfconfig.xml` | `8e82646c5e38f1e1fa3144f82a07a7d1` |
| `system/vendor/etc/thermal-engine_gpu_0.conf` | `1a92a76398912f3a4f486a846e5d26fb` |
| `system/vendor/etc/thermal-engine_gpu_1.conf` | `3b53951971b139604e862b8a2cc97978` |
| `system/vendor/etc/thermal-engine_gpu_2.conf` | `107fe02aa0f2943b454b2c9d2ab4888d` |
| `system/vendor/etc/thermal-engine_gpu_5.conf` | `f6b5007dc5e490e76ca4d4e58d20c9cd` |
| `backup/gameperfconfig.xml.orig` | `ce849bb0c9497faff1311f37fd2fd189` |
| `backup/gameperfconfig.xml.orig.md5` | `734fa63f7608573e0460d20ecb2e7e18` |
| `stock.md5` | `493fb9b0e2e78ca2a6c0b169260d9e08` |
| `post-fs-data.sh` | `3b43064490690af93a75bf8e0417da55` |
| `service.sh` | `641dc9b1c9eeb09a8553bb47b16ad8fc` |
| `tools.sh` | `8fa0ee6af3611fed608e1124b23d7e2c` |
| `uninstall.sh` | `012a30e45e6e86d59061e2489631566f` |
| `lib/rebuild.sh` | `5b47ad3d2f4ab6e01966a03579cbc797` |
| `customize.sh` | `985f0615332721df533249c937c8d10e` |
| `module.prop` | `e23d2043411475d778325183fdbd6e4a` |
| `META-INF/com/google/android/update-binary` | `28674067d011d04ecff270ea1cbf044d` |
| `META-INF/com/google/android/updater-script` | `b39cd178923d60d67de5c5b81a78c242` |

### 原厂文件（`stock.md5` 里记的值）

| 文件 | md5 | 大小 |
|---|---|---|
| `gameperfconfig.xml` | `ce849bb0c9497faff1311f37fd2fd189` | 395162 |
| `thermal-engine_gpu_0.conf` | `9ba9e783f57003855cfd0480ff3c6698` | 771 |
| `thermal-engine_gpu_1.conf` | `3bc7fbb8fabd92d4deceef8d33c156a5` | 385 |
| `thermal-engine_gpu_2.conf` | `b470d0e3215293e298869cddf4c8f9dc` | 771 |
| `thermal-engine_gpu_5.conf` | `53e13ee8ddb1fe8101b7067a56a91315` | 774 |

### 电脑上的备份

- `D:\Y700G5_原厂备份\` —— 5 个原厂文件 + `README.txt`
- `D:\Y700G5_原厂备份\rebuild\` —— 重建工具（`rebuild.sh` + `template/`）

---

## 10. 已知限制

1. **只对四种高性能模式生效**：省电（`PowerSave`）与均衡（`Balance`）的 GPU 上限完全没动，省电模式下鸣潮仍然只有 342MHz。
2. **温控配置是全局的**：4 个 conf 不只对游戏生效。不过阈值推到 55℃ 以上，日常使用到不了，实际无感。
3. **助手副本需要开机 45 秒后才同步**：开机后前 45 秒内进游戏，可能还是旧配置（`service.sh` 的 `sleep 45`）。
4. **清空游戏助手数据后要重启两次**：第一次 `post-fs-data` 时副本还不存在、`service.sh` 第 45 秒跑完时助手还没生成副本（约 70 秒），所以那一轮会锁 1000MHz。
5. **`uninstall.sh` 在 KernelSU 3.3.1 上不会执行**：这是 KernelSU 的顺序 bug，卸载后请用 `tools.sh` 菜单 3 清理残留。
6. **`sync.log` 会逐渐变长**：每开机一次追加一行，没有自动清理，可以随手删。
7. **GPU 下限已放到最低档**：`17_0` 允许掉到 160MHz。实测这是好事——闲时降得下去，结温才低，重载才给得到 1200MHz（见 3.1 的 A/B 数据）。
8. **App 命名空间免疫**：如果某个游戏或工具自己缓存了配置，模块的 bind mount 对它无效，只能像游戏助手这样直接改副本。
9. **剩下的瓶颈不在 GPU，在 CPU**：47℃ 以上野兽模式会把 CPU 砍到 1.9 / 2.1GHz，GPU 常常在等 CPU（`gpu_busy` 只有 79~85%，从不到 99%），所以掉帧不是频率被锁。详见 4.4。没有继续动 CPU 是刻意的取舍。
10. **逐文件 bind mount 会被 overlay 盖住**（v1.10 已修）：KernelSU 的 `/system/etc`、`/vendor/etc` overlay 挂载时机晚于 `post-fs-data`，会盖住逐文件 bind mount。现在改为把重建结果写回模块文件，bind mount 仅作兜底。

---

## 11. 变更记录

| 版本 | 主要变化 |
|---|---|
| v1.0 | 初版：XML 全放开（`17_0`）+ 4 个温控 conf 阈值后移 + 标准 META-INF |
| v1.1 | 移除 `gpu_1`/`gpu_5`（本机不加载），只留 `gpu_0`/`gpu_2` |
| v1.2 | 加回 `gpu_1`/`gpu_5`（以防万一） |
| v1.3 | 目录改为规范布局 `system/vendor/etc`；补 `vendor_configs_file` 标签；补 `META-INF` |
| v1.4 | 加 `post-fs-data.sh` 用 bind mount（解决 overlay 对 App 不可见的问题）——**失败** |
| v1.5 | 改为直接同步游戏助手私有副本 + `uninstall.sh` 还原 + `backup/` 随包原厂备份 |
| v1.6 | OTA 保护（`mount_guard` + `stock.md5` + `xml_state`）；作者署名；文档与重建工具 |
| **v1.7** | **`17_0` -> `7_0`（保留 646MHz 下限）；开机自动备份原厂文件；OTA 后自动重建；新增 `lib/rebuild.sh` 与 `/sdcard/Y700G5_工具/tools.sh`；`customize.sh` 结构检查改为只提示** |
| **v1.8** | **改名 `Y700 5th Boost`；新增 `accepted.md5`，OTA 后只记录一次，消除每次开机重复 `UPDATED` 的日志噪音；备份日志文案改为「备份已就绪」；`tools.sh` 版本横幅同步到 v1.8** |
| **v1.9** | **`17_0` 回滚：v1.7 的 `7_0` 实测导致 1200MHz 完全跑不满（同局 A/B：>=1050MHz 驻留 7.5% vs 48.2%，`thermal_pwrlevel` 触发率 35% vs 0%）；`lib/rebuild.sh` / `tools.sh` / `customize.sh` 的 GPU 值同步回 `17_0`；版本号 10、简介与文档更新** |
| **v1.10** | **修复 overlay 遮蔽：重建结果写回模块文件（旧版留在 `build/<key>.prev`），不再依赖被 KSU overlay 盖住的逐文件 bind mount；`devof` 加 `-L`、`ANCHOR` 改用绝对父目录（修 CWD 相关的设备号误判）；`customize.sh` 补回 `system_file` / `vendor_configs_file` 标签；`tools.sh` 新增「重建版（待重启生效）」判定并在重建后写 `accepted.md5`；简介与帮助的温度改为游戏 59/62/65/68℃、非游戏 55/58/61/64℃ |

---

*本文档随模块一起维护。改了模块行为请同步改这里。*
