# 中国历史疆域图 · 开发日志

> **项目地址**：https://github.com/ChanceChen12/china-history-map  
> **在线访问**：https://chancecheN12.github.io/china-history-map  
> **记录日期**：2025年6月28日  
> **开发方式**：Claude Code（AI辅助全程开发）

---

## 一、需求起点

**目标**：做一个覆盖秦朝至新中国（前221年—2024年）的历史疆域交互网页，帮助使用者快速查询：

- 某个地理位置在历史上的行政区划归属
- 某个历史时期的疆域范围及重大事件
- 不同朝代之间的地理变迁对比

**三个核心维度**（设计前期确立）：

```
地理位置（坐标）× 行政区划（随朝代变化）× 时间轴（前221—2024）
```

**关键设计原则**：
1. 行政区划按朝代切换，同一坐标在不同时代可能归属完全不同的区划
2. 历史事件坐标固定（地理位置本身不随时代变）
3. 现代底图（CartoDB）提供现代地理参考，无需额外"现代行政层"
4. 数据精准优先——无可靠来源的数据宁可不加

---

## 二、数据方案演进（踩坑记录）

### 2.1 第一版：手工绘制矩形多边形

**做法**：在 `data.js` 的 `ADMIN_DATA` 里手工定义每个行政区划为矩形多边形（如 `[[108.5,34],[112,34]...]`）

**问题**：精度极差，矩形无法代表任何真实历史边界，用户直接反馈"太扯了"

**教训**：历史地理数据必须来自学术可靠来源，不能自己编造

---

### 2.2 寻找数据源

#### 哈佛大学 CHGIS V6

**来源**：Harvard Dataverse × 复旦大学历史地理研究中心  
**访问**：https://dataverse.harvard.edu/dataverse/chgis

**下载的核心数据集**：
| 文件 | 大小 | 内容 |
|---|---|---|
| `v6_time_pref_pgn_utf_wgs84.zip` | 30MB | 时序府级多边形（-224年至1911年），**WGS84** |
| `v6_1911_prov_pgn_utf.zip` | 2.3MB | 1911年省级多边形（**西安1980投影**，坑！） |
| `v6_time_cnty_pts_utf_wgs84.zip` | 531KB | 时序县级坐标点（WGS84，用于生成Voronoi） |

**CHGIS的核心问题——覆盖偏东**：

```
唐代700年 CHGIS 数据分布：
  110°E 以东：104 个单元（占 90%）
  110°E 以西：11 个单元（仅 10%）

→ 西安（108.9°E）在唐代没有任何 CHGIS 多边形覆盖
→ 秦朝仅有 13 个郡（全部在东南：闽中、会稽、长沙等）
```

**根本原因**：CHGIS 基于方志文献数字化，东南沿海地区文献记录最完整，关中/陇右/蜀地数据尚未完成入库。

#### 哈佛 Hartwell China Historical GIS v5

**来源**：Harvard Dataverse，Robert Hartwell 遗作，2010年发布  
**访问**：https://dataverse.harvard.edu/dataset.xhtml?persistentId=doi:10.7910/DVN/29302

**关键发现**：Hartwell 有分政权数据！

```
1200年时间切片文件结构：
  v5_1200_chin_chn_1200_p.shp  → 南宋（宋）
  v5_1200_jin_pref_*_1200_p.shp → 金朝（独立分层！）
  v5_1200_indp_xia_*_1200_p.shp → 西夏
  v5_1200_indp_mng_*_1200_p.shp → 蒙古

1080年时间切片：
  v5_1080_pref_*.shp → 北宋
  v5_1080_liao_*.shp → 辽（南京道、东京道等五道！）
  v5_1080_indp_xxia_1080.shp → 西夏
```

**坑**：Hartwell 使用 **Xian_1980_GK_Zone_19（高斯-克吕格投影，米制）**，必须用 pyproj 转换为 WGS84

```python
from pyproj import Transformer, CRS
src_crs = CRS.from_wkt(prj_content)  # 读 .prj 文件
dst_crs = CRS.from_epsg(4326)
transformer = Transformer.from_crs(src_crs, dst_crs, always_xy=True)
lon, lat = transformer.transform(x_meters, y_meters)
```

**Hartwell 覆盖范围**（最终使用的时间切片）：

| 朝代 | 使用切片 | 单元数 |
|---|---|---|
| 唐 | 741年 | 675 |
| 北宋/辽 | 1080年 | 301（宋）+107（辽）|
| 南宋/金 | 1200年 | 201（南宋）+190（金）|
| 元 | 1290年 | 280 |
| 明 | 1391年 | 610 |

#### 世界历史政权：aourednik/historical-basemaps

**来源**：GitHub，GPL-3 开源  
**访问**：https://github.com/aourednik/historical-basemaps

**关键价值**：全球历史政权 GeoJSON，从公元前100年到2010年，每100年一个切片，每文件约1MB，~253个政权/切片

```
geojson/
  world_bc100.geojson   # 公元前100年（秦/西汉）
  world_bc1.geojson     # 公元前1年（西汉）
  world_100.geojson     # 100年（东汉）
  world_200.geojson     # 200年（三国）
  ...
  world_1279.geojson    # 1279年（元朝）
  world_1492.geojson    # 1492年（明/哥伦布时代）
  ...
  world_1960.geojson    # 1960年（新中国）
```

**字段**：`NAME`（英文名）+ `SUBJECTO`（归属）+ `BORDERPRECISION`

#### 其他数据源

| 来源 | 用途 | 格式 | 访问 |
|---|---|---|---|
| 阿里 DataV | 新中国35个省级单元 | GeoJSON | `geo.datav.aliyun.com` 公开API |
| Natural Earth 110m | 现代176国边界 | GeoJSON | `naturalearthdata.com` 公域 |
| CHGIS 县级时序点 | 生成 Voronoi（后废弃） | Shapefile+WGS84 | Harvard Dataverse |

---

## 三、技术架构

### 3.1 数据处理流水线

```
原始数据（Shapefile/GeoJSON）
    ↓
坐标转换（pyproj: Xian1980 → WGS84）
    ↓
几何简化（Douglas-Peucker，epsilon=0.03~0.5°）
    ↓
环/孔洞处理（Shoelace有向面积判断外环/内环）
    ↓
输出 GeoJSON（geojson/ 目录）
    ↓
打包（geojson.js，~8MB，const CHGIS_GEOJSON）
```

**关键脚本**：
- `chgis/convert.py`：CHGIS V6 时序数据 → GeoJSON（秦汉等早期朝代）
- `chgis/convert_hartwell.py`：Hartwell GIS → GeoJSON（唐宋元明）
- `chgis/convert_voronoi.py`：县级点 → Voronoi 面（**已废弃，精度差**）

### 3.2 前端架构

```
单文件 HTML（index.html）
├── CSS（暗色主题，变量系统）
├── Leaflet.js（地图渲染）
├── data.js（朝代元信息 + 历史事件，~236条）
└── geojson.js（所有地理数据，~8MB）
    ├── 中国行政区划（16朝代）
    ├── 并存政权（辽/金/西夏/蒙古）
    ├── 世界历史底图（16时间切片）
    └── 现代国家边界（Natural Earth）
```

### 3.3 地图层架构（三层独立）

```
Layer 1: 中国历史行政区划层（随朝代切换，CHGIS/Hartwell）
         ↳ 支持并存政权分色展示（宋/辽/西夏各自颜色）

Layer 2: 历史事件层（坐标固定，按朝代过滤）
         ↳ 中国事件 + 世界同期事件

Layer 3: 世界历史政权层（aourednik，随朝代切换时间切片）
         ↳ 勾选显示，切换朝代自动切换对应年份底图
```

### 3.4 关键功能

| 功能 | 实现方式 |
|---|---|
| 朝代时间轴 | 顶部横向滚动按钮，点击切换 |
| 年份滑块 | `-221` 至 `2024`，拖动自动切换朝代 |
| 点击多边形查区划 | L.geoJSON onEachFeature + 点在多边形检测（射线法）|
| 事件↔区划双向关联 | featureLayerMap 缓存图层引用，onEventClick 高亮多边形 |
| 搜索 | 内置60+城市库（含历史别名）+ CHGIS全文检索 + Nominatim API |
| 搜索标记 | 固定坐标 pin，切换朝代自动更新历史标注 |
| 并存政权显示 | CONCURRENT_REGIMES 定义，独立颜色分层渲染 |
| 全球视角 | 83条世界事件 + aourednik世界政权版图 |
| 启动自检 | runSelfTest()，数据缺失时面板红色 banner |

---

## 四、重大 Bug 记录

### Bug 1：Xian1980 投影坐标当作 WGS84 使用

**现象**：民国省级数据下载后，坐标变成 `17856960.35` 这样的大数，多边形渲染到太平洋外  
**根因**：`v6_1911_prov_pgn_utf.zip` 使用 Xian_1980_GK_Zone_19（米制），不是 WGS84  
**修复**：用 pyproj 转换，或改用时序数据（已含 `_wgs84` 后缀）

### Bug 2：`style.display = ''` 无法显示 CSS 初始隐藏元素

**现象**：点击多边形无任何反应，loc-card、search-results、搜索框均不显示  
**根因**：CSS 中 `#loc-card{display:none}` + JS 用 `element.style.display = ''`（移除内联样式），反而回退到 CSS 的 `none`  
**影响范围**：8处 `display=''`，导致信息面板、搜索结果、图例均无法正常显示  
**修复**：全部改为明确值：`'block'`、`'inline-block'`、`'inline'`

### Bug 3：TDZ（临时死区）导致行政区划全部不渲染

**现象**：切换朝代后地图一片空白，无任何多边形  
**根因**：
```javascript
// ❌ 错误：onEachFeature 在 L.geoJSON() 执行期间调用
// 此时 const layer 处于 Temporal Dead Zone
const layer = L.geoJSON(gj, {
  onEachFeature(feature, fl) {
    featureLayerMap.set(p.name, { fl, layer }); // ReferenceError!
    fl.on('mouseout', () => layer.resetStyle(fl)); // 同上
  }
});
```
**修复**：
```javascript
let layer;  // 先声明
layer = L.geoJSON(gj, {
  onEachFeature(feature, fl) {
    // 不在此处引用 layer
  }
});
// L.geoJSON 返回后 layer 已赋值，再填 map
layer.eachLayer(fl => featureLayerMap.set(fl.feature.properties.name, { fl, layer }));
```

### Bug 4：Voronoi 县域面积巨大

**现象**：用县级坐标点生成 Voronoi 后，西部省份（甘肃、新疆）一个"县"的覆盖面积相当于整个省  
**根因**：西部地区县治稀疏，Voronoi 剖分后每个格子面积超大，与真实县界无关  
**决策**：废弃 Voronoi 方案，回归 CHGIS 原始多边形数据，对覆盖缺口如实标注

### Bug 5：搜索"北京"无结果

**现象**：输入"北京"无任何结果显示  
**根因1**：CHGIS 数据里没有"北京"字段——北京在唐叫幽州、金叫中都、元叫大都、明清叫顺天府  
**根因2**：Nominatim API 从 `file://` 加载时 User-Agent 被浏览器拦截，API 调用静默失败  
**修复**：内置60+城市库（含历史别名），如北京→`['燕京','大都','北平']`，本地即时匹配

---

## 五、数据精准性声明

### 有精确数据的朝代

| 朝代 | 数据来源 | 精确程度 |
|---|---|---|
| 唐（741年） | Hartwell GIS v5 | 县级，基于历史文献 |
| 北宋/辽/西夏（1080年） | Hartwell GIS v5 | 府路级，辽国五道独立 |
| 南宋/金/西夏/蒙古（1200年）| Hartwell GIS v5 | 府级，四政权分色 |
| 元（1290年） | Hartwell GIS v5 | 路级 |
| 明（1391年） | Hartwell GIS v5 | 县级精度 |
| 清（1820年） | CHGIS V6 | 省级 |
| 民国（1911年切片）| CHGIS V6 | 府级 |
| 新中国 | 阿里DataV/国家测绘局 | 省级（官方标准） |

### 数据有限的朝代（如实标注）

| 朝代 | CHGIS 实际覆盖 | 限制原因 |
|---|---|---|
| 秦 | 13郡（均在东南） | 方志记录稀缺 |
| 西汉 | 18郡（偏东） | 同上 |
| 东汉 | 16郡 | 同上 |
| 三国/西晋 | 25-38郡（偏东） | 同上 |
| 南北朝/隋/五代 | 69-130单元 | 中等覆盖 |

> 应用内对每个朝代显示数据质量指示器（🟢/🟡/🔴），覆盖不全的区域**如实显示空白而非编造多边形**

---

## 六、当前技术栈

```
前端框架：纯 HTML/CSS/JavaScript（无构建工具依赖）
地图库：Leaflet.js 1.9.4
底图：CartoDB Voyager（含现代城市/省份/河流标注）
数据格式：GeoJSON（WGS84）
数据工具：Python 3.13 + pyshp + pyproj + scipy
部署：GitHub Pages（静态托管）
```

---

## 七、项目文件结构

```
history-map/
├── index.html              # 主应用（~1650行，含所有交互逻辑）
├── data.js                 # 朝代元信息 + 236条历史事件（中国+世界）
├── geojson.js              # 所有地理数据打包（~8MB）
│
├── geojson/                # 各朝代 GeoJSON 原始文件
│   ├── qin.geojson         # 秦（CHGIS V6，13郡）
│   ├── tang.geojson        # 唐（Hartwell 741年，675单元）
│   ├── liao.geojson        # 辽（并存政权，Hartwell 1080年）
│   ├── world_700.geojson   # 700年世界政权（aourednik）
│   └── ...（共31个文件）
│
├── chgis/                  # 数据处理脚本和原始数据
│   ├── convert.py          # CHGIS Shapefile → GeoJSON
│   ├── convert_hartwell.py # Hartwell GIS → GeoJSON（含坐标转换）
│   ├── convert_voronoi.py  # ⚠ 已废弃：Voronoi方案
│   ├── v6_pref/            # CHGIS V6 时序府级多边形（67MB .shp）
│   ├── hartwell/           # Hartwell GIS v5（79MB .zip）
│   └── ne_110m_countries.json  # Natural Earth 世界国家
│
├── world_maps/             # 世界历史底图原始文件（20个切片，27MB）
│
└── DEVLOG.md               # 本文档
```

---

## 八、遗留问题 & 待优化

### 数据层面
- [ ] 秦汉早期朝代的西部区划（关中、陇右、河西走廊）无精确公开数据
  - 等待 CHGIS V7 发布（复旦团队仍在持续更新）
  - 或联系复旦历史地理研究中心申请完整数据集访问
- [ ] Hartwell 方法论说明：使用现代县域边界"拼接"历史行政区域，本质是近似值
- [ ] 世界政权底图（aourednik）精度：使用者应知晓该数据基于历史地图集数字化，古代边界本身就存在不确定性

### 性能层面
- [ ] `geojson.js` 达 8MB，初始解析约 2-5秒（建议按朝代按需加载）
- [ ] 明代（610个单元）/新中国世界底图（1300个政权）渲染较慢

### 功能层面
- [ ] 历史行政区划"历代沿革"功能：同一地点各朝代名称变迁的自动查询
- [ ] 移动端适配
- [ ] 三国、南北朝的并存政权多层分色（目前未做）

---

## 九、参考资料

| 资源 | 链接 | 说明 |
|---|---|---|
| CHGIS V6 | https://chgis.fas.harvard.edu | 中国历史GIS主数据库 |
| Harvard Dataverse CHGIS | https://dataverse.harvard.edu/dataverse/chgis | 数据下载入口 |
| Hartwell GIS | https://doi.org/10.7910/DVN/29302 | 唐宋元明分政权数据 |
| aourednik/historical-basemaps | https://github.com/aourednik/historical-basemaps | 全球历史政权GeoJSON |
| Natural Earth | https://naturalearthdata.com | 现代世界地图（公域） |
| 阿里云DataV | https://datav.aliyun.com | 中国省级行政区划 |
| CShapes 2.0 | https://icr.ethz.ch/data/cshapes | 1886年后全球国家边界 |
| Leaflet.js | https://leafletjs.com | 地图渲染库 |
| pyproj | https://pyproj4.github.io | 坐标系转换工具 |

---

*文档生成时间：2025年6月28日*  
*AI 辅助工具：Claude Code (claude-opus-4-8)*
