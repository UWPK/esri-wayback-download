# Wayback 实现参考

本仓库提供技能与可移植工作流说明。下载实现需要 PowerShell 7 和 GDAL，按运行环境配置。

## 瓦片网格计算

令 `O=20037508.342789244`、级别为 z：

```text
span = 2*O / 2^z
resolution_3857 = span / 256
x = floor((X + O)/span)
y = floor((O - Y)/span)
left = -O + x*span
top = O - y*span
GeoTransform = [left, resolution_3857, 0, top, 0, -resolution_3857]
```

瓦片列为 x、行为 y，向下增大。裁剪范围检查有效索引边界；跨日期变更线区域应拆分规划。Z17 的投影像元约 1.1943 米；纬度 φ 附近地表尺度近似乘 cos(φ)，并结合元数据 SRC_RES 解释实际影像分辨率。

VRT 为每个 RGB 波段写 SourceFilename、SrcRect、DstRect；XML 路径转义，小数使用 invariant culture。缺块在拼接时保留有效区信息，避免默认零填充成为有效影像。

## 版本与拍摄时间查询

tilemap 单瓦片探测入口：

```text
https://wayback.maptiles.arcgis.com/arcgis/rest/services/World_Imagery/MapServer/tilemap/{release}/{z}/{y}/{x}/1/1
```

本项目以 valid/data 判断可用性，用 select 返回值归一底层版本；需检查实际响应结构。多个发布日期可能共用该位置同一影像，归一结果限定在所查询的位置和级别。

从目录 metadataLayerUrl 查服务图层结构。本项目使用 `/6/query`，新任务先核验该图层支持的字段和查询能力。推荐参数：

```text
f=json
where=MinMapLevel <= z AND MaxMapLevel >= z
geometry=<区域EPSG:3857外接矩形>
geometryType=esriGeometryEnvelope
inSR=3857
outSR=3857
spatialRel=esriSpatialRelIntersects
outFields=OBJECTID,SRC_DATE,SRC_DATE2,MinMapLevel,MaxMapLevel,DrawOrder,NICE_DESC
returnGeometry=true
orderByFields=OBJECTID
resultRecordCount=1000
resultOffset=<分页偏移>
```

检查 error、geometryType、CRS、exceededTransferLimit；偏移量按本页实际条数增加，截断但空页时报错，防止遗漏或死循环。服务器分页能力变化时采用对象 ID 分批。SRC_DATE2 解析常见日期字符串，SRC_DATE 依实际字段类型处理。

在本地裁剪原边界后转 EPSG:6933 等积统计。同年份斑块先 union；跨年份重叠作为歧义区单列。本项目保守下界扣除跨年重叠，上界为该年份斑块并集；未知日期面积另报。分母为整个原始区域。按单一档期中的目标年份覆盖率选优，避免把跨档期并集写成某一版实际覆盖。案例 70% 为任务阈值，可按新需求调整。

## 运行与恢复要点

- 原始脚本可先 Plan，再 Run -MaxChunks 1 小范围核验，最后完整运行；Status 读取状态。调用前先检查全部固定参数与隐含行为。
- 多年脚本默认 Workers=48，且包含自动增加并发；初版为 2 并发和每请求 2–4 秒延迟。按实际网络与服务器响应选择参数，检查自动升并发逻辑是否符合本次计划。
- STOP 文件用于保留缓存暂停；恢复前检查任务进程/锁及暂停原因，再处理标记。限流暂停与正常 STOP 分开记录。
- 历史脚本记录 missing404，缓存一定时间后再查；2017 逻辑可能使用 2016-12-20 / 18966 回填，必须保留每块来源。原脚本默认参数之间存在任务特定年份组合，执行前检查目录确认对应关系。
- 计划中的所有瓦片与分块都应归入已完成或已明确缺失。缺块分块可延后；带缺块完成需保留缺失清单与无效掩膜，避免把暂缓分块遗漏成完整交付。
- 原脚本完成后会清除缓存；需要进一步修复或核验时，先调整清理时机以保留恢复材料。

## 校验结果的表述

项目交付检查使用 5×5 分散位置读取 RGB，共 75 次波段读取，另读掩膜并核对已知缺瓦片中心。这提供结构与抽样可读证据；涉及全覆盖完整性时，还需瓦片/分块清单核对和有效区统计。海南案例固定宽高与文件列表不能直接沿用其他区域。

