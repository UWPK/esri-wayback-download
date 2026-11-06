# Wayback 实现参考

本仓库提供技能与可移植工作流说明。可使用 Python 3 的 requests、Pillow、pyproj、Shapely 及 Rasterio/GDAL 在 Windows、macOS、Linux 实现。也可直接调用 GDAL 命令行。按运行环境选择可用后端。

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

按服务的 valid/data 字段判断可用性，用 select 返回值归一底层版本；需检查实际响应结构。多个发布日期可能共用该位置同一影像，归一结果限定在所查询的位置和级别。

从目录 metadataLayerUrl 查服务图层结构。先查询 metadataLayerUrl 的服务目录，选择含所需字段的图层，再构造 `/{layerId}/query` 并核验查询能力。推荐参数：

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

在本地裁剪原边界后转 EPSG:6933 等积统计。同年份斑块先 union；跨年份重叠作为歧义区单列。覆盖率保守下界扣除跨年重叠，上界为该年份斑块并集；未知日期面积另报。分母为整个原始区域。按单一档期中的目标年份覆盖率选优，避免把跨档期并集写成某一版实际覆盖。覆盖率验收阈值由具体任务确定。

## 运行与恢复要点

- 分离计划、执行和状态查询；先运行一个分块，验证坐标、图像、掩膜及元数据后扩展。
- 默认低并发，依据成功率调整。为 429 尊重 Retry-After，有限重试；持续 403/429 时保存状态并停止请求。
- 状态用 UTF-8 JSON，记录区域、版本、级别、输出网格和参数。恢复前逐项比较；缓存按版本/级别/行/列隔离。
- 临时文件与目标位于同一文件系统，关闭句柄后原子替换。单任务锁使用独占创建；跨平台实现可采用 Python os.open 的 O_CREAT | O_EXCL。PID 可能重用，检查任务身份后再处理遗留锁。
- 使用 STOP 标记在分块边界暂停，保留缓存。确保暂停、错误与完成状态明确。
- 所有计划瓦片归入完成或明确缺失；缺块保持无效掩膜。跨版本修补仅在任务允许时使用，并为每块记录来源。
- 清理缓存需等交付和恢复需求确认后进行。

## 验证与结果表述

核查尺寸、RGB 波段、CRS、仿射变换、像元大小、内部掩膜和已知缺块。读取分散窗口，并核验区域外无效、有效黑色像素保留。抽样读取提供局部可读证据；完整覆盖需要计划/下载清单对账及有效区面积统计。记录实际执行的平台和依赖版本。
