# eventlabel

**事件相机数据交互式异步标注工具。** 你只需在目标上放置跟踪 blob，逐事件扩展卡尔曼滤波（来自 [AEB Tracker](https://arxiv.org/abs/2307.10593)，IEEE TRO 2024）会跟随 blob 并把每个事件标注为目标（1）或背景（0）——支持按对象分配跟踪 id、多对象工作流、回看/撤销、AVI 视频渲染。异步标注方法来自 [ASUMOT](https://arxiv.org/abs/2607.11303)（[代码](https://github.com/Jia-Bao/ASUMOT)）。

[English documentation](README.md)

| 原始事件（极性视图） | 标注结果（id 着色视图） |
|:---:|:---:|
| ![raw view](docs/images/raw_view.png) | ![labeled view](docs/images/labeled_view.png) |

## 功能特性

- **异步标注 / 删除**——`A` 加跟踪 blob，`D` 删除（并回退其标签）。跟踪器以微秒级分辨率逐事件自动打标。
- **回看、修正、撤销**——随时暂停（`P`），增删 blob，回放已完成数据（`--view`）；删 blob 只回退它标的那些行；`X` 一键撤销整个对象。
- **多目标跟踪标注**——一轮标一个对象，可用任意多个 blob 覆盖；同一对象的所有 blob 共享一个 id；重新加载同一份原始文件标注下一个对象（id 永不冲突）。
- **目标消失/重现**——对象离开画面时 `R` 释放跟踪器但**保留**已标标签；重现后在新的位置重新 `A`，仍是同一个 id。
- **配色**——白色画布；原始事件红/蓝极性显示；非跟踪标注绿色；跟踪对象按 10 色调色板着色；帧率 30/60/120，运行中可切换。
- **首帧预标注**——启动时冻结画面，先放好 blob 再点 START 从 t = 0 完整处理，开头不漏标。
- **AVI 导出**——按最终标注渲染 MJPG 编码的 `*_unlabeled_view.avi`（极性）与 `*_labeled_view.avi`（id 着色）。
- **参数设置窗口（Windows）**——双击 exe：选择文件（多选队列）、调整全部参数，标完一个文件程序不退出，继续下一个。

## 标注模型

跟踪器跟踪的是事件 **blob**，而一个真实目标（如无人机）通常由**多个 blob 覆盖**——每个 blob 按一次 `A`。

- `track_mode: false`——单任务标注：所有 blob 标注的事件都写 `1`，无 id。
- `track_mode: true`——**一轮标注一个对象**：本轮放置的所有 blob 共享一个对象 id；按 `N` 切换下一个 id（或重新加载）；id 唯一且永不复用。

已标注的事件在数据关联前直接跳过，因此后续轮次中一个对象的 blob **抢不走**其他对象的事件——多个对象可以在画面中自由重叠。

## 存储：原始 CSV 永不重写

```
events.csv           原始事件（只读，校验和不变）
events_labels.csv    旁注索引：每个已标注事件一行 "行号,track_id"
events_unlabeled_view.avi / events_labeled_view.avi     渲染视图
events_labeled.csv   全量 6 列合并 CSV（按需，E 键）
```

多对象标注时每轮都加载**同一份原始文件**，只累积很小的旁注索引（KB–MB，而不是每轮重写 GB 级数据）。旁注是纯文本，可直接查看或编辑。运行中每秒自动保存。

旧版全量标注 CSV（带第 5/6 列）首次加载时自动导入旁注索引。

## 使用

### 图形界面（推荐）

不带参数启动 `eventlabel.exe`：

- **Browse...** 选择输入 CSV——**支持多选**形成队列；每个文件标完后设置窗口自动回来（程序不退出），按 START 载入下一个；路径也可直接粘贴。
- 窗口上可调：宽/高、列序、帧率（30/60/120）、关联门限、预标注时长，以及跟踪模式 / 首帧预标注 / 视频导出 / 合并 CSV 导出 / 仅回看 复选框。
- 设置持久化在 exe 旁的 `settings.yaml`；EKF 噪声参数来自 `configs/uav.yaml`。

### 命令行（进阶用法）

```
eventlabel [-c configs/uav.yaml] [--view]
```

### 键位（标注模式）

| 键 | 功能 |
|---|---|
| `A` | 加 blob（点击；一个对象可放多个） |
| `D` | 删 blob 并**回退**其全部标签（标错了用这个） |
| `R` | 释放 blob 但**保留**其标签（对象离开画面时用这个） |
| `P` / 空格 | 暂停 / 继续（暂停中可用 A/D/R/N/X） |
| `S` / 回车 | 开始（预标注阶段）/ 继续 |
| `N` | 下一个对象 id（跟踪模式） |
| `X` | 撤销最近标注的对象（回退其全部行） |
| `F` | 帧率 30 → 60 → 120 循环 |
| `V` | 立即导出两路 AVI |
| `E` | 立即导出全量合并 CSV |
| `ESC` | 结束当前文件（标签保存；设置窗口返回） |

回看模式：`F` / `V` / `E` / `ESC`。预标注阶段：`A`、`N`、START 按钮或 `S`。

## 配置（`configs/uav.yaml`）

| 键 | 说明 | 默认 |
|---|---|---|
| `input_folder_path` / `input_data_name` | 输入 = 路径 + 名称 + `.csv` | – |
| `width` / `height` | 画布 = 传感器分辨率（坐标 0 基） | 1280/720 |
| `data_format` | `0`: ts,c,r,p[,...] · `1`: c,r,p,ts[,...] | 1 |
| `dist_threshold` | 关联门限下限（px） | 10 |
| `framerate` | 显示/导出帧率：30/60/120 | 30 |
| `track_mode` | 对象 id 标注开关 | false |
| `pre_annotate` | 首帧预标注 + START | true |
| `pre_annotate_time_s` | 冻结预览时长（宜小） | 0.05 |
| `export_videos` | 结束自动导出两路 AVI | true |
| `export_csv` | 结束自动导出合并 CSV（一般用 `E` 按需） | false |
| `var_*` / `q_*` | EKF 噪声参数（AEB 默认值） | 见文件 |

缺省键自动取默认值；旧键名 `publish_framerate` 仍被识别。

## 数据格式

输入（无表头事件 CSV；第 5/6 列可省略——有则视为已有标注并自动导入）：

```
data_format=0: ts(us), c, r, p[, label[, track_id]]
data_format=1: c, r, p, ts(us)[, label[, track_id]]
```

旁注 `*_labels.csv`：`idx,track_id`（`idx` = 输入文件数据行的 0 基序号，跳过表头；非跟踪标注为 `-1`）。

合并 CSV（`E`）：`c,r,p,ts(us),label,track_id`——与输入逐行对齐，时间戳为整数微秒。

> 标注过程中不要修改原始 CSV——旁注行号与其数据行绑定。

## 多对象标注工作流示例

```
第 1 轮：track_mode=true，输入 events.csv（原始数据）
        预标注：A 放 blob 覆盖对象1（可多个）→ START → 跑完
        → 生成 events_labels.csv（id 0）
第 2 轮：输入仍是 events.csv，旁注自动加载（id 0 按其颜色显示）
        新放的 blob 自动把对象2 标注为 id 1
第 N 轮：依此类推；按 X 重做最近的对象，D/R 修正单个 blob，
        --view（或"仅回看"复选框）检查结果
最终   ：按一次 E 导出全量 CSV 供训练使用
```

对象离开画面？`P` → `R` 释放其 blob（标签保留）→ 继续；重现后 `P` → 在新位置 `A`（同一 id）。

## 从旧版本迁移

- 坐标不再 +1（旧版的偏移会导致回灌逐轮漂移）。`width/height` 请填实际分辨率（如 1281×992 → 1280×992）。
- 旧输出（`*_label_opint.csv`、5/6 列全量 CSV）可直接加载，首次加载时标注列自动迁入旁注索引。
- 旧格式时间戳超过 1 秒后精度受损（仅 6 位有效数字）；按行号索引的旁注天然免疫。

## 致谢与引用

异步标注方法来自 ASUMOT：

> Baofeng Jia, Xiaoyu Chen, Jingyuan Zhang, Zongze Wu, Haochen Li, Jing Han, Lianfa Bai,
> "ASUMOT: Motion-Consistency-Based Asynchronous UAV Detection and Tracking with Event Cameras",
> arXiv:2607.11303, 2026。[论文](https://arxiv.org/abs/2607.11303) · [代码](https://github.com/Jia-Bao/ASUMOT)

跟踪内核派生自 AEB Tracker：

> Ziwei Wang, Timothy Molloy, Pieter van Goor and Robert Mahony, "Asynchronous Blob Tracker for Event Cameras", *IEEE Transactions on Robotics*, 2024. [arXiv:2307.10593](https://arxiv.org/abs/2307.10593)

使用本工具时请引用 ASUMOT 论文（标注方法）与 AEB 论文（跟踪内核），并附本发布链接。

## 许可

仅限学术用途——详见 [LICENSE](LICENSE)（英文）。上游 AEB Tracker 以 "for academic use only" 发布，衍生跟踪内核继承该限制。
