# Geo/Geoplot 使用说明（中文）

本 README 针对使用 Python 的 geoplot 库（基于 geopandas、matplotlib、cartopy）来绘制地理空间图形的常见输出与具体方法进行详细说明。内容包含数据准备、常见图类型、参数解释、示例代码、投影/底图处理、色标/图例设置、输出保存与调优建议，旨在帮助你理解每种图的含义以及如何调整参数获得期望的可视化结果。

目录
- 介绍
- 安装与依赖
- 数据准备（GeoDataFrame、CRS）
- 通用绘图流程（创建 fig/ax、投影）
- 常见图表类型与输出解释
  - Choropleth（分级着色/行政区统计图）
  - Polygon / Polyplot（面/边界可视化）
  - Pointplot（点图：点位与属性大小/颜色）
  - KDE（核密度估计图：点的空间密度）
  - Hexbin（六边形分箱）
  - Cartogram（面积变形图；若可用）
- 投影与底图（Cartopy / Contextily）
- 颜色、归一化与图例
- 输出与保存
- 性能与调优建议
- 常见问题与解决
- 参考文献

---

## 介绍
geoplot 是一个用于制作高质量地理可视化的高级库，封装了许多针对地理数据（如 GeoDataFrame）的绘图方法。它提供更便捷的可视化接口，比直接使用 matplotlib + geopandas 更方便地控制投影、色标、密度估计等地理专用样式。

本 README 重点讲解输出（图像）如何解读、每种方法的典型参数与如何准备数据以获得正确的图像。

---

## 安装与依赖
推荐使用 conda 环境或 pip 安装。常见依赖：
- geoplot
- geopandas
- matplotlib
- cartopy
- contextily（用于瓦片底图）
- descartes（旧版本的依赖，取决于环境）

示例（pip）：
```
pip install geoplot geopandas matplotlib cartopy contextily
```

有时在安装 cartopy 或 geopandas 时需要系统包（proj、gdal）。建议用 conda 在大多数系统上更顺利：
```
conda install -c conda-forge geoplot geopandas cartopy contextily
```

---

## 数据准备（GeoDataFrame、CRS）
所有 geoplot 的方法通常输入为 GeoPandas 的 GeoDataFrame（简称 gdf）或 geometry 列（GeoSeries）。关键点：
- 确保有 `geometry` 列且 geometry 类型正确（Point, LineString, Polygon 等）。
- 确保坐标参考系（CRS）匹配你所用的投影或底图。常见：
  - WGS84（经纬度）: `EPSG:4326`
  - Web Mercator（用于瓦片底图）: `EPSG:3857`
- 常用操作：
  - 查看 CRS: `gdf.crs`
  - 设定 CRS（若无）: `gdf = gdf.set_crs("EPSG:4326")`
  - 变换 CRS: `gdf = gdf.to_crs("EPSG:3857")`
  - 点的中心: `gdf.geometry.centroid`（注意：在经纬度上计算面积/距离不准确，先投影到合适 CRS）

---

## 通用绘图流程（示例）
1. 导入并准备数据
2. 创建 figure 和 axis（可以指定 cartopy 投影）
3. 调用 geoplot 的绘图函数并传入 ax
4. 添加颜色条、注释、底图、保存

示例框架：
```
import geopandas as gpd
import geoplot as gp
import matplotlib.pyplot as plt
import cartopy.crs as ccrs

gdf = gpd.read_file("data/regions.geojson").to_crs("EPSG:4326")

fig, ax = plt.subplots(figsize=(10, 8), subplot_kw={'projection': ccrs.PlateCarree()})
gp.choropleth(gdf, hue='population_density', cmap='OrRd', ax=ax, linewidth=0.5, edgecolor='gray')
ax.set_title("示例：人口密度分布")
plt.show()
```

---

## 常见图表类型与输出解释

### 1) Choropleth（分级着色）
用途：按行政区或栅格单元对数值属性进行着色（如人口密度、失业率）。
输出如何解读：
- 每个多边形颜色表示其属性值（由 colormap 控制）。
- 比较不同区域的颜色深浅可以判断相对大小。
- 若使用分类分组（如 quantiles），则颜色表示分组区间而非连续量。
关键参数：
- hue（或 `column`）：用于着色的字段名
- cmap：colormap，常用 'OrRd', 'viridis', 'coolwarm' 等
- k：若希望将数值分成 k 个类别（分箱）
- vmin/vmax：控制颜色映射范围
示例：
```
gp.choropleth(gdf, hue='median_income', cmap='viridis', ax=ax, linewidth=0.4, edgecolor='k')
```

注意：当数据分布高度偏斜时，考虑对数变换或使用分类（分位数）。

---

### 2) Polygon / Polyplot（面/边界可视化）
用途：显示地理边界、行政区边界或自定义多边形。
输出：
- 通常用于展示几何形状、边界重心、几何关系。
常用参数：facecolor、edgecolor、linewidth、alpha。
示例：
```
gp.polyplot(gdf, facecolor='lightgray', edgecolor='black', linewidth=0.6, ax=ax)
```
如果只想显示边界而不填充：
```
gdf.boundary.plot(ax=ax, color='black', linewidth=0.5)
```

---

### 3) Pointplot（点图）
用途：展示点要素（例如 POI、站点、事件位置），可以用点大小或颜色编码额外属性（例如人口、流量）。
输出：
- 点的位置对应地理坐标，点的大小/颜色表示属性的大小/类别。
参数：
- scale 或 `s`：点的大小，可使用字段进行缩放（例如 `s=gdf['population'] / 1000`）
- hue：用于分类着色
- marker、alpha、zorder：控制风格
示例：
```
gp.pointplot(gdf_points, hue='category', cmap='tab10', scale='population', ax=ax)
# 或使用 matplotlib 风格
ax.scatter(gdf_points.geometry.x, gdf_points.geometry.y,
           s=gdf_points['population'] / 1000, c=gdf_points['value'], cmap='Reds', alpha=0.7)
```

解释输出：
- 通过点的半径/颜色观察热点与聚集程度；
- 若点高度重叠，考虑抖动、透明度或使用密度图。

---

### 4) KDE（Kernel Density Estimate，核密度估计）
用途：基于点要素估计空间密度（热点图），适合显示事件聚集模式。
输出：
- 一张光滑的密度面图，颜色/等值线代表单位面积上的相对密度。
关键参数：
- bw (bandwidth)：平滑程度，较大值更平滑
- cmap、levels：色图和等值线数
示例：
```
gp.kdeplot(gdf_points, clip=gdf.geometry, cmap='Reds', shade=True, bw_adjust=1, ax=ax)
```
解释：
- 高密度区域被渲染为颜色更深的区域；
- 选择不同带宽可揭示局部热点或更广泛趋势。

---

### 5) Hexbin（六边形分箱）
用途：把点聚合到固定网格（六边形），统计每个箱内点数或某属性的汇总，适用于大规模点数据。
输出：
- 网格化图，每个六边形颜色表示箱内统计值。
关键参数：
- gridsize：网格大小（影响分辨率）
- extent：数据范围
示例（使用 geoplot 或 matplotlib 的 hexbin）：
```
ax.hexbin(x, y, gridsize=100, cmap='YlOrRd', mincnt=1)
```

---

### 6) Cartogram（面积变形图）
用途：按属性（如人口）扭曲区域面积以反映数量差异（可视化人口或选票等权重）。
注意：
- 生成 cartogram 通常是耗时且可能破坏地理直观对应关系。需要专门的算法（如 Gastner-Newman）。
- geoplot 提供或集成的工具可能有限，实际使用可借助外部包（如 `cartogram_geopandas`、`geocart` 等）或预先生成 cartogram 数据再用 geoplot 绘制。
示例思路：
1. 计算或读取 cartogram 变形后的几何
2. 使用 polyplot / choropleth 绘制变形后的图像

---

## 投影与底图（Cartopy / Contextily）
- geoplot 常与 cartopy 的投影配合：在创建 subplot 时传入 `subplot_kw={'projection': ccrs.Mercator()}` 等。
- 使用瓦片底图（如 OSM）常需将数据转换为 Web Mercator（EPSG:3857），然后用 contextily.add_basemap(ax, source=...) 添加：
```
import contextily as ctx
gdf_3857 = gdf.to_crs(epsg=3857)
ax = gdf_3857.plot(figsize=(10, 10), alpha=0.6)
ctx.add_basemap(ax, source=ctx.providers.Stamen.TonerLite)
```
注意 CRS 匹配：底图通常为 EPSG:3857。

---

## 颜色、归一化与图例
- 连续数据使用 colormap（cmap），并用 Normalize 或 BoundaryNorm 控制映射：
```
from matplotlib import colors
norm = colors.Normalize(vmin=0, vmax=100)
gp.choropleth(gdf, hue='value', cmap='viridis', norm=norm, ax=ax)
```
- 类别数据使用离散 colormap 或 ListedColormap。
- 添加 colorbar：
```
sm = plt.cm.ScalarMappable(cmap='viridis', norm=norm)
sm._A = []
cbar = fig.colorbar(sm, ax=ax, fraction=0.03, pad=0.04)
```

---

## 输出与保存
使用 matplotlib 的 `savefig`：
```
fig.savefig("output/map.png", dpi=300, bbox_inches='tight')
```
保存为矢量格式（SVG/PDF）可在后续排版中放大不会失真：
```
fig.savefig("output/map.svg")
```

---

## 性能与调优建议
- 数据量大时：
  - 使用 `geometry.simplify(tolerance)` 降低顶点数量（在不影响视觉的前提下）。
  - 使用空间索引（rtree）做空间查询和裁剪，避免渲染超出视窗的要素。
  - 使用分箱（hexbin）或密度估计代替直接绘制大量点。
  - 考虑使用 datashader（与 geopandas/geoviews 配合）进行百万级点的渲染。
- 避免在经纬度（EPSG:4326）上做面积计算或距离度量，应先投影到等距或等面积 CRS。

---

## 常见问题与解决
- 图为空白或数据看不到：
  - 检查 geometry 是否为空：`gdf.is_empty` / `gdf.geometry.notnull()`
  - 检查 CRS 是否匹配：地理数据 vs 投影轴
- 点全部叠在一起：
  - 检查是否用错了列（如把经度当作纬度）
  - 使用透明度、旁路或抖动
- 颜色条范围不对：
  - 明确设置 `vmin` / `vmax` 或对数据做变换（log）
- Cartopy 地图缩放或切片异常：
  - 确保投影与数据 CRS 匹配，检查 axis 的 `set_extent`

---

## 参考
- geoplot 官方文档（查看最新 API）：https://residentmario.github.io/geoplot/
- GeoPandas 文档（数据准备与操作）：https://geopandas.org/
- Cartopy 文档（投影与底图）：https://scitools.org.uk/cartopy/docs/latest/
- Contextily（瓦片底图）：https://contextily.readthedocs.io/

---

## 示例脚本与示例数据（在 examples/ 目录）
仓库包含一个 `examples/plot_examples.py` 脚本，它会自动下载/加载 geopandas 自带的示例数据（naturalearth_lowres、nybb 等），并生成多张示例地图（choropleth, pointplot, kde, hexbin）。请参阅 `examples/README.md` 获取运行说明和输出路径。