# QGIS

## SymBolType

1. Marker：`Marker` 是用于表示点要素的符号样式。在 `QGIS` 中，你可以选择不同的标记样式，如圆圈、方块、箭头等，来表示地图中的点数据。这些标记通常用于标记地理位置或特定点的位置。

1. Line：`Line` 是用于表示线要素的符号样式。在地图中，线通常用于表示道路、河流、边界等线性要素。`QGIS` 允许你选择不同的线样式和宽度，以便更好地呈现线要素。

1. Fill：`Fill` 是用于表示面要素的符号样式。面要素通常用于表示地图上的区域，如国家边界、湖泊、林区等。通过填充符号样式，你可以为面要素选择不同的颜色、图案和透明度，以便更清晰地展示地图中的区域。

1. Hybird：`Hybrid` 是一种符号样式，结合了 `Marker`、`Line` 和 `Fill` 三种样式的特点。这种符号样式可以在一个地图要素中同时使用点、线和面的表现形式，以便更加细致地描述地图要素的属性。

## WKB

`Well-Known Binary`，通常用于在不同的 GIS 系统之间共享和交换几何数据。WKB 格式的数据结构包含几何类型（如点、线、多边形等）以及坐标数据，以便在计算机程序中能够准确地重建几何对象。

### WkbType

类型描述了几何对象的维度数量，以及每个单独顶点的维度数量。包括 x，y 和可选的 z，m。

描述了包括点 (`Point`)，线串 (`LineString`) 和多边形 (`Polygon`)。

## QgsAbstractGeometry

QGIS 中所有几何图形的抽象基类

## QgsGeometry

用于创建，编辑和分析矢量几何对象，实际几何对象在 `QgsAbstractGeometry` 中，可以通过 `get()` 以及 `set()` 的方式来访问实际的几何对象

### QgsPoint

继承自 `QgsAbstractGeometry`，可以用于 2 维，3 维，4 维的坐标系，包括 x，y，z，m（其中 z 和 m 可选）

#### snappedToGrid

将 `QgsPoint` 对象按照给定的水平、垂直、深度（如果有）和 M 坐标网格间距进行网格化，并返回一个新的经过网格化处理的 `QgsPoint` 对象。

### QgsPointXY

## QgsLayerTree

用于管理图层和图层组

### QgsLayerTreeNode

通过一个 `enum`，分为 `Layer` 和 `Group` 两种 `Node`

## QgsMapLayer

用于表示地图中的数据图层，可以是矢量图层、栅格图层或其他类型的图层。

![image-20241105144950939](D:/md/img/image-20241105144950939.png)