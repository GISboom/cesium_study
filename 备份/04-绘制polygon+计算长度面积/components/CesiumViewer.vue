<template>
  <div id="cesiumContainer"></div>
  <div class="toolbar">
    <button @click="startDraw">开始绘制</button>
    <button @click="cancelDraw">结束绘制</button>
    <button @click="removeAllPolygons">删除所有多边形</button>
  </div>
  <!-- 测量信息 -->
  <div v-if="isDrawing" class="measure-info">
    <div>周长：{{ currentLength.toFixed(2) }} m</div>
    <div>面积：{{ currentArea.toFixed(2) }} m²</div>
  </div>
</template>
<script setup>
import { onMounted, onUnmounted, ref } from "vue";
import * as Cesium from "cesium";
import "cesium/Build/Cesium/Widgets/widgets.css";
let viewer;
let handler;

//面
let currentPositions = [];
let currentPolygon = null;
let mousePosition = null;

// 用于存储所有绘制的多边形实体
let allPolygons = [];

//预览线
let previewLineEntity = null;

//管理顶点
let vertexEntities = [];

//绘制状态
let isDrawing = ref(false);

//测量信息
const currentLength = ref(0);
const currentArea = ref(0);

function startDraw() {
  isDrawing.value = true;
}
function cancelDraw() {
  if (!isDrawing.value) {
    return;
  }
  isDrawing.value = false;
  cancelDrawing();
}

//Esc键取消绘制
function handleKeyDown(event) {
  if (event.key === "Escape") {
    cancelDrawing();
  }
}

//取消绘制当前polygon
function cancelDrawing() {
  if (currentPolygon) viewer.entities.remove(currentPolygon);
  //清空当前绘制数据
  currentPolygon = null;
  currentPositions = [];
  mousePosition = null;
  
  //移除预览线
  if (previewLineEntity) viewer.entities.remove(previewLineEntity);
  //删除所有顶点实体
  clearVertexEntities();
  updateMeasurement();
}

//删除所有Polygon
function removeAllPolygons() {
  // 先取消正在绘制的 Polygon
  cancelDrawing();
  // 删除所有已经完成的 Polygon
  allPolygons.forEach((entity) => {
    viewer.entities.remove(entity);
  });
  // 清空数组
  allPolygons.length = 0;
  // 删除所有顶点实体
  clearVertexEntities();
  updateMeasurement();
}

//删除所有顶点实体
function clearVertexEntities() {
  vertexEntities.forEach((entity) => {
    viewer.entities.remove(entity);
  });
  vertexEntities = [];
}

//封装点击事件处理函数
function handleLeftClick(click) {
  // 如果不是绘制状态，直接返回
  if (!isDrawing.value) return;
  const ray = viewer.camera.getPickRay(click.position);
  const cartesian = viewer.scene.globe.pick(ray, viewer.scene);
  if (!cartesian) return;
  currentPositions.push(cartesian); // 添加当前点

  // 创建预览线
  previewLineEntity = viewer.entities.add({
    polyline: {
      positions: new Cesium.CallbackProperty(() => {
        const tempPositions = [...currentPositions];
        if (mousePosition) {
          tempPositions.push(mousePosition);
        }
        return tempPositions;
      }, false),
      width: 2,
      material: Cesium.Color.RED,
    },
  });
  // 创建顶点
  const vertexEntity = viewer.entities.add({
    position: cartesian,
    point: {
      pixelSize: 8,
      color: Cesium.Color.RED,
      outlineColor: Cesium.Color.WHITE,
      outlineWidth: 2,
    },
  });
  // 将顶点添加到数组中
  vertexEntities.push(vertexEntity);
  //更新测量结果
  updateMeasurement();
}

//封装鼠标移动事件处理函数
function handleMouseMove(movement) {
  if (!isDrawing.value) {
    return;
  }
  const ray = viewer.camera.getPickRay(movement.endPosition);
  const cartesian = viewer.scene.globe.pick(ray, viewer.scene);
  if (!cartesian) return;
  mousePosition = cartesian; // 更新鼠标当前位置
  //更新测量结果
  updateMeasurement();
  
  if (currentPositions.length < 2) return;
  if (currentPolygon) viewer.entities.remove(currentPolygon);

  currentPolygon = viewer.entities.add({
    id: `polygon-${Date.now()}`,
    polygon: {
      //使用CallbackProperty动态更新多边形的顶点
      //这样就可以在鼠标移动时实时更新多边形的形状
      hierarchy: new Cesium.CallbackProperty(() => {
        const tempPositions = [...currentPositions];
        if (mousePosition) {
          tempPositions.push(mousePosition);
        }
        return new Cesium.PolygonHierarchy(tempPositions);
      }, false),
      material: Cesium.Color.YELLOW.withAlpha(0.3),
    },
  });

}

//封装鼠标右键点击事件处理函数
function handleRightClick(click) {
  if (currentPositions.length >= 3) {
    //把临时预览实体转为永久实体
    const polygonEntity = viewer.entities.add({
      polygon: {
        hierarchy: new Cesium.PolygonHierarchy(currentPositions),
        material: Cesium.Color.ORANGE.withAlpha(0.4),
        outline: true,
      },
    });
    allPolygons.push(polygonEntity); // 将当前多边形实体添加到数组中
  }
  if (currentPolygon) viewer.entities.remove(currentPolygon);
  //清空当前绘制数据
  currentPolygon = null;
  currentPositions = [];
  mousePosition = null;
  //移除预览线
  if (previewLineEntity) viewer.entities.remove(previewLineEntity);
}

// 更新测量结果
function updateMeasurement() {
  const positions = [...currentPositions];

  // 鼠标当前位置加入临时计算
  if (mousePosition) {
    positions.push(mousePosition);
  }

  // 至少两个点才能计算长度
  if (positions.length >= 2) {
    currentLength.value = calculateLength(positions);
  } else {
    currentLength.value = 0;
  }

  // 至少三个点才能计算面积
  if (positions.length >= 3) {
    currentArea.value = calculateArea(positions);
  } else {
    currentArea.value = 0;
  }
}

// 计算 Polygon 周长
function calculateLength(positions) {
  let length = 0;
  // 计算每两个相邻点之间的距离
  //笛卡尔坐标 → 经纬度大地坐标 → 椭球体测地线距离
  for (let i = 0; i < positions.length - 1; i++) {
    const start = Cesium.Cartographic.fromCartesian(positions[i]);
    const end = Cesium.Cartographic.fromCartesian(positions[i + 1]);
    const geodesic = new Cesium.EllipsoidGeodesic(start, end);
    length += geodesic.surfaceDistance;
  }
  // 如果已经有三个点
  // 还要计算最后一个点 → 第一个点
  if (positions.length >= 3) {
    const start = Cesium.Cartographic.fromCartesian(
      positions[positions.length - 1],
    );
    const end = Cesium.Cartographic.fromCartesian(positions[0]);
    const geodesic = new Cesium.EllipsoidGeodesic(start, end);
    length += geodesic.surfaceDistance;
  }
  return length;
}

// 计算 Polygon 面积
function calculateArea(positions) {
  if (positions.length < 3) {
    return 0;
  }
  // 将笛卡尔坐标转换为经纬度坐标
  const cartographics = positions.map((position) =>
    Cesium.Cartographic.fromCartesian(position),
  );
  //获取地球半径（wgs84地球椭球体）
  const radius = Cesium.Ellipsoid.WGS84.maximumRadius;
  let area = 0;
  for (let i = 0; i < cartographics.length; i++) {
    const p1 = cartographics[i];
    // %取模为了让最后一个点重新连接第一个点，比如有四个点，i=3时，p2应该是第一个点，所以用(i + 1) % cartographics.length
    const p2 = cartographics[(i + 1) % cartographics.length];
    //球面 Polygon 面积计算公式
    area +=
      (p2.longitude - p1.longitude) *
      (2 + Math.sin(p1.latitude) + Math.sin(p2.latitude));
  }
  //abs()取绝对值是因为：polygon的顶点顺序可能不是顺时针的，导致计算出来的面积是负数
  //球面面积公式计算出来的面积是球面面积的比例，所以要乘以半径的平方
  area = (Math.abs(area) * radius * radius) / 2;
  return area;
}

onMounted(() => {
  viewer = new Cesium.Viewer("cesiumContainer", {
    selectionIndicator: false,
  });
  const camera = viewer.camera;
  camera.setView({
    destination: Cesium.Cartesian3.fromDegrees(116.397428, 39.90923, 10000.0),
  });

  //鼠标事件
  handler = new Cesium.ScreenSpaceEventHandler(viewer.scene.canvas);

  //左键添加点
  handler.setInputAction(
    handleLeftClick,
    Cesium.ScreenSpaceEventType.LEFT_CLICK,
  );

  // 鼠标移动：更新临时点
  handler.setInputAction(
    handleMouseMove,
    Cesium.ScreenSpaceEventType.MOUSE_MOVE,
  );

  //右键结束绘制当前polygon
  handler.setInputAction(
    handleRightClick,
    Cesium.ScreenSpaceEventType.RIGHT_CLICK,
  );

  //Esc键取消绘制当前polygon
  window.addEventListener("keydown", handleKeyDown);
});

onUnmounted(() => {
  if (handler) {
    handler.destroy();
    handler = null;
  }
  if (viewer) {
    viewer.destroy();
    viewer = null;
  }
  window.removeEventListener("keydown", handleKeyDown);
});
</script>
<style scoped>
#cesiumContainer {
  width: 100%;
  height: 100vh;
}
.toolbar {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 1000;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
}
button {
  font-size: 16px;
  width: 100px;
  height: 60px;
}
.measure-info {
  position: absolute;
  top: 2%;
  left: 10%;
  z-index: 1000;
  padding: 10px 15px;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  border-radius: 4px;
  line-height: 24px;
}
</style>
