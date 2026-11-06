# 跨平台实现

工作流适用于 Windows、macOS、Linux，包括无桌面服务器。此仓库提供代理技能与实现参考；代理按用户环境实现下载程序。

## 后端选择

- 推荐 Python 3.10+：requests 负责 HTTP；Pillow 验证瓦片可解码、尺寸和 RGB；pyproj 与 Shapely 处理坐标和区域；Rasterio 或 GDAL 处理栅格、掩膜及 GeoTIFF。
- 环境已有 GDAL 命令行时，可组合 HTTP 下载器、VRT、gdal_translate 和 gdalwarp。使用 subprocess 参数数组调用程序，避免拼接 shell 命令。
- PowerShell 7 可作为三平台上的可选 HTTP/进程编排方式。

## 安装环境

三平台均可使用 conda-forge 创建隔离环境：

```text
conda create -n wayback -c conda-forge python=3.12 requests pillow pyproj shapely rasterio gdal
conda activate wayback
python -c "import requests, PIL, pyproj, shapely, rasterio; print(rasterio.__gdal_version__)"
gdalinfo --version
```

也可使用虚拟环境与适合当前 Python/CPU 的 pip wheels，或 OS 包管理器。GDAL Python bindings 与本机 GDAL 库需要兼容；优先让 GDAL、PROJ 和相关数据来自同一环境。macOS 需确认 arm64/x64，Linux 需确认 glibc/musl 和可用 wheels。Windows 无需依赖盘符、注册表或 DLL 硬编码路径。

## 可移植实现约定

- 使用 pathlib.Path、UTF-8 文件及系统临时目录；不假定盘符、反斜杠、用户目录名或固定代理端口。
- HTTP 超时、有限退避、User-Agent 和 Referer 显式设置。继承标准代理环境变量，或使用用户提供的配置；检查 requests 的 trust_env 与直连选择一致。
- TLS 保持证书验证。用户所在组织需要自定义 CA 时，通过运行环境配置可信证书。
- always_xy=True 转换地理坐标；经纬度采用经度、纬度顺序。Web Mercator 限制纬度约 ±85.05112878°，跨日期变更线区域分开规划。
- 使用 GDAL/Rasterio 的正常库加载机制；按真实安装布局设置 GDAL_DATA/PROJ_DATA，保持已有正确配置。
- 图像验证使用 Pillow/Rasterio，空间检查使用 pyproj/Shapely；避免 System.Drawing、COM、Windows 专用进程和绝对共享库路径。
- 通过 GeoJSON 或数据文件读取区域，显式记录 CRS。服务器分页、版本日期、拍摄日期和缺块规则见 [project-workflow.md](project-workflow.md)。

## 小范围验收

对用户区域的一小块先完成目录解析、至少一块瓦片下载与解码、地理定位、GeoTIFF 输出与窗口读取，再扩大范围。若筛选拍摄年份，另验证元数据分页与等积面积统计。保存服务快照与运行版本；网络和服务访问仍取决于实际环境。

官方入口：[Wayback](https://livingatlas.arcgis.com/wayback/)；[发布目录](https://s3-us-west-2.amazonaws.com/config.maptiles.arcgis.com/waybackconfig.json)；[GDAL](https://gdal.org/)；[Rasterio](https://rasterio.readthedocs.io/)。
