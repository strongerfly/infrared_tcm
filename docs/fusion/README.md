# 后期融合占位（Fusion placeholder）

本文件只定义**以后**把四条轨道的分布图并列看的接口。融合阶段尚未开始。当前没有实现、没有样例数据、没有综合分数。

各轨的研究问题与成功标准仍以轨道文档为准：

- [../tracks/face/README.md](../tracks/face/README.md)
- [../tracks/tongue/README.md](../tracks/tongue/README.md)
- [../tracks/ear/README.md](../tracks/ear/README.md)
- [../tracks/hand/README.md](../tracks/hand/README.md)

边界见 [../LIMITS.md](../LIMITS.md)。

## 阶段前提

进入实现之前，进度板上四条轨道都应已经具备：

1. 冻结的几何分区词表
2. 质量门控旗标
3. 仓库外产物路径约定
4. 重复性指标的操作定义（数值可以仍为空，但字段含义要稳定）

缺任何一条，融合视图只显示“该轨未就绪”，不用另一轨的图填空。

## 并列，而不是同一坐标系

面、舌、耳、手默认**不**配准到一个共同解剖坐标。后期可视化是同一会话或同一假名下的小倍数（small multiples）和时间线。

允许的对齐只有轨道内部已经定义的对齐：例如同一个人的两次面部采集。跨轨道不插值、不叠图、不求一个“全身分布”。

## 占位记录

下面的结构是文档契约，版本号 `fusion.placeholder.v0`。它不是要在本仓库里运行的程序。字段用英文标识，便于以后若写代码时直接对照。

### TrackId

`face` | `tongue` | `ear` | `hand`

### AcquisitionMeta

| 字段 | 含义 |
| --- | --- |
| `session_id` | 一次采集会话 |
| `subject_pseudonym` | 不可逆假名 |
| `track` | `TrackId` |
| `view` | 轨道内视图。手：`palmar` / `dorsal`。耳：可空，侧别另记 |
| `laterality` | `left` / `right` / `midline` / `na` |
| `captured_at` | ISO 8601 时间 |
| `band` | `vis` / `nir` / `swir` / `mwir` / `lwir` / `other` |
| `wavelength_nm` | `center` 与 `fwhm`；热成像宽带可空，但 `band` 必须有 |
| `sensor_id` | 仓库外设备台账中的标识 |
| `distance_mm` | 工作距离，未知则空 |
| `ambient_c` | 环境温度，未知则空 |
| `emissivity_set` | 热像仪设定值，不是组织发射率真值 |
| `illumination_id` | NIR/VIS 照明配置；热成像可空 |
| `protocol_id` | 指向轨道文档中的方案版本 |

### QualityReport

| 字段 | 含义 |
| --- | --- |
| `passed` | 是否允许进入分布视图 |
| `flags` | 轨道文档里定义的旗标字符串 |
| `metrics` | 质量数值，键来自该轨道成功标准 |

`passed=false` 的产物可以出现在质量审查列表里，不进入分布小倍数。

### RegionStat 与 DistributionMap

`RegionStat`：`region_id`、`pixel_count`、`mean`、`std`、`p05`、`p50`、`p95`。`region_id` 必须属于该轨道词表。

`DistributionMap`：

| 字段 | 含义 |
| --- | --- |
| `uri` | 仓库外栅格 |
| `quantity` | `radiance` / `temperature_apparent` / `reflectance` / `index` |
| `unit` | 单位字符串 |
| `width`、`height` | 像素尺寸 |
| `mask_uri` | 统计所用掩膜 |
| `regions` | `RegionStat` 列表 |

`temperature_apparent` 表示仪器在当时发射率设定下的表观温度。

### TrackArtifact

一件产物 = `AcquisitionMeta` + `QualityReport` + `DistributionMap`。标识 `artifact_id`。原始影像 URI 可附在元数据，不进 git。

### FusionBoard

| 字段 | 含义 |
| --- | --- |
| `board_id` | 一块看板 |
| `artifact_ids` | 要并列的产物 |
| `layout` | `small_multiples` 或 `session_timeline` |
| `note` | 自由文本，供实验者记录协议偏差 |

`layout=small_multiples`：每个产物一格，格内是分布图加分区表。格子之间不共享色标，除非 `quantity` 与 `unit` 相同且笔记写明色标范围。不同波段禁止共用一个色标。

`layout=session_timeline`：横轴时间，纵轴按轨道分面。用于查看同一会话的采集顺序与质量旗标，不用于画出跨轨的连续曲线。

## 明确不出现的字段

契约拒绝下列字段名及同义项：`diagnosis`、`syndrome`、`zheng`、`efficacy`、`risk_score`、`triage`、`device_clearance`。评审接口草稿时，出现这些名字即退回。

也不设跨轨加权和、投票或“整体指数”。

## 可视化要回答的问题

后期只服务这三类描述性问题：

1. 这一会话里，四条轨道各有没有一张**已通过质量门控**的分布图？缺哪一轨？
2. 在单轨词表内，分区中位数与分位区间是多少？有效像素是否够该轨自己的下限？
3. 同一轨道、同一协议的重复会话，分区统计是否落在该轨定义的重复性散差之内？散差未测完时，图上标注“重复性未建立”。

视图不回答“此人是否有某病/某证”，也不比较不同轨道的数值谁更“异常”。

## 开放占位

实现开始前仍空着的决定，记在这里，避免各轨私自假设：

- [ ] 仓库外对象存储的 URI 方案与访问权限
- [ ] `protocol_id` 的版本命名
- [ ] 色标在同一 `quantity` 下是否固定范围
- [ ] 看板是静态导出还是交互页面（本仓库在决定之前保持为零代码）
