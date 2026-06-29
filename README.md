# 中国历史疆域图

**在线访问**：https://chancecheN12.github.io/china-history-map

一个覆盖**秦朝（前221年）至新中国（2024年）**的历史地理信息交互网页，支持行政区划查询、历史事件浏览和全球视角对比。

![预览图](https://raw.githubusercontent.com/ChanceChen12/china-history-map/main/preview.png)

---

## 功能概览

| 功能 | 说明 |
|---|---|
| **16个朝代** | 秦→汉→三国→晋→南北朝→隋→唐→五代→宋辽→南宋金→元→明→清→民国→新中国 |
| **并存政权分色** | 北宋/辽/西夏、南宋/金/西夏/蒙古 独立颜色同屏显示 |
| **历史事件** | 236条（含83条世界同期事件），地图标记+事件面板双向关联 |
| **区划查询** | 点击任意地图多边形，显示该地行政区划信息 |
| **事件↔区划联动** | 点击事件自动高亮所在行政区划；选中区划自动过滤该地事件 |
| **搜索** | 现代地名（含历史别名）+ CHGIS历史区划名全文检索 |
| **全球视角** | 同期世界政权版图（按朝代自动切换年份），83条世界历史事件 |
| **启动自检** | 数据完整性自动检测，异常时面板提示 |

---

## 数据来源

| 数据层 | 来源 | 许可 |
|---|---|---|
| 中国历史政区（秦汉） | [哈佛大学 CHGIS V6](https://chgis.fas.harvard.edu) | 学术使用 |
| 中国历史政区（唐宋元明） | [Hartwell China Historical GIS v5](https://dataverse.harvard.edu/dataset.xhtml?persistentId=doi:10.7910/DVN/29302) | 学术使用 |
| 中国历史政区（清民国） | CHGIS V6 时间切片 | 学术使用 |
| 新中国省级区划 | [阿里云 DataV / 国家测绘局](https://datav.aliyun.com) | 公开 |
| **世界历史政权** | [aourednik/historical-basemaps](https://github.com/aourednik/historical-basemaps) | GPL-3 |
| 现代国家边界 | [Natural Earth 110m](https://naturalearthdata.com) | 公域 |

---

## 本地运行

直接打开 `index.html` 即可（无需服务器，无需安装依赖）。

```bash
open index.html
```

> 注：geojson.js 约 8MB，首次加载需要 2-5 秒解析时间。

---

## 数据精确性说明

- **唐/宋/元/明**：Hartwell GIS v5，基于历史文献的学术重建，府级精度
- **秦汉三国西晋等早期朝代**：CHGIS V6 覆盖有限（以东部为主），西部区划无公开精确数据，应用内有🔴标注
- **世界政权**：aourednik/historical-basemaps 基于多部历史地图集数字化，边界为学术近似值

---

## 开发文档

详见 [DEVLOG.md](./DEVLOG.md)，包含：
- 数据方案演进历程
- 关键 Bug 记录与修复
- 各数据源的坑和使用方法

---

*技术栈：HTML/CSS/JS · Leaflet.js · Python（数据处理）*  
*AI 辅助：Claude Code*
