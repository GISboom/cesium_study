<template>
  <div id="cesiumContainer"></div>
  <div class="toolbar">
    <button @click="startDraw">开始绘制</button>
    <button @click="cancelDraw">结束绘制</button>
    <button @click="removeAllPolygons">删除所有多边形</button>
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

// 当前正在拖动
let isDragging = false;
// 正在拖动的顶点索引
let dragVertexIndex = null;
// 正在编辑的Polygon数据
let dragPolygon = null;

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
}

//删除所有Polygon
function removeAllPolygons() {
  // 先取消正在绘制的 Polygon
  cancelDrawing();
  // 删除所有已经完成的 Polygon
  allPolygons.forEach((polygon) => {
    viewer.entities.remove(polygon.entity);
  });
  // 清空数组
  allPolygons.length = 0;
  // 删除所有顶点实体
  clearVertexEntities();
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
  if (!isDrawing.value) {
    // console.log("not drawing")
    // 只有在非绘制状态下才允许选择多边形
    selectPolygon(click);
  } else {
    // console.log("drawing")
    // 在绘制状态下，添加顶点
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
  }
}

//封装鼠标移动事件处理函数
function handleMouseMove(movement) {
  if (!isDrawing.value) {
    //非绘制状态，实现点的拖动
    handleDragMove(movement);
  } else {
    // 绘制状态，实现多边形的预览
    const ray = viewer.camera.getPickRay(movement.endPosition);
    const cartesian = viewer.scene.globe.pick(ray, viewer.scene);
    if (!cartesian) return;
    mousePosition = cartesian; // 更新鼠标当前位置

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
}

//封装鼠标右键点击事件处理函数
function handleRightClick(click) {
  if (currentPositions.length >= 3) {
    //定义数据存储格式
    const polygonData = {
      positions: [...currentPositions],
      vertexEntities: [],
      entity: null,
    };
    //
    polygonData.entity = viewer.entities.add({
      polygon: {
        hierarchy: new Cesium.CallbackProperty(() => {
          return new Cesium.PolygonHierarchy(polygonData.positions);
        }, false),
        material: Cesium.Color.ORANGE.withAlpha(0.4),
        outline: true,
      },
    });
    allPolygons.push(polygonData); // 将当前多边形数据添加到数组中
  }
  if (currentPolygon) viewer.entities.remove(currentPolygon);
  //清空当前绘制数据
  currentPolygon = null;
  currentPositions = [];
  mousePosition = null;
  //移除预览线
  if (previewLineEntity) viewer.entities.remove(previewLineEntity);
  clearVertexEntities();
}

//结束绘制，鼠标左键点击选择多边形
function selectPolygon(click) {
  if (isDrawing.value) return; // 如果正在绘制，不进行选择
  const picked = viewer.scene.pick(click.position);
  // console.log(picked);

  if (!picked || !picked.id) return;
  const entity = picked.id;

  const polygon = allPolygons.find((polygon) => polygon.entity === entity);
  if (!polygon) return;

  showVertices(polygon);
}

//选中多边形，显示顶点
function showVertices(polygon) {
  // 防止重复显示
  clearVertexEntities();
  polygon.positions.forEach((position, index) => {
    const point = viewer.entities.add({
      position: new Cesium.CallbackProperty(() => {
        return polygon.positions[index];
      },false),
      
      point: {
        pixelSize: 8,
        color: Cesium.Color.RED,
        outlineColor: Cesium.Color.WHITE,
        outlineWidth: 2,
      },
    });
    point.type = "vertex";
    // 第几个点
    point.vertexIndex = index;
    // 属于哪个Polygon
    point.parentPolygon = polygon;
    polygon.vertexEntities.push(point); // 将顶点添加到数组中
    vertexEntities.push(point);
  });
}

//拖动顶点
function handleLeftDown(click) {
  const picked = viewer.scene.pick(click.position);
  if (!picked || !picked.id) {
    return;
  }
  const entity = picked.id;
  if (entity.type !== "vertex") return;
  isDragging = true;
  dragVertexIndex = entity.vertexIndex;
  dragPolygon = entity.parentPolygon;
  enableCameraControl(false); //关闭相机控制
}

//拖动顶点
function handleDragMove(movement) {
  if (!isDragging) {
    return;
  }
  const ray = viewer.camera.getPickRay(movement.endPosition);
  const cartesian = viewer.scene.globe.pick(ray, viewer.scene);

  if (!cartesian) {
    return;
  }
  // 修改顶点
  dragPolygon.positions[dragVertexIndex] = cartesian;
}

//封装鼠标左键抬起事件处理函数
function handleLeftUp() {
  isDragging = false;
  dragVertexIndex = null;
  dragPolygon = null;
  enableCameraControl(true); //开启相机控制
}

//关闭cesium默认的相机控制,用在拖动顶点时
function enableCameraControl(enable) {
  //左键旋转地球
  viewer.scene.screenSpaceCameraController.enableRotate = enable;
  //中键平移
  viewer.scene.screenSpaceCameraController.enableTranslate = enable;
  //滚轮缩放
  viewer.scene.screenSpaceCameraController.enableZoom = enable;
  //倾斜
  viewer.scene.screenSpaceCameraController.enableTilt = enable;
  //视角查看
  viewer.scene.screenSpaceCameraController.enableLook = enable;
}
onMounted(() => {
  viewer = new Cesium.Viewer("cesiumContainer", {
    selectionIndicator: false,
    infoBox: false,
    animation: false,
    timeline: false,
    fullscreenButton: false,
    geocoder: false,
    homeButton: false,
    sceneModePicker: false,
    navigationHelpButton: false,
    // baseLayerPicker: false,
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

  // 鼠标移动：更新临时点+绘制预览功能+顶点拖动
  handler.setInputAction(
    handleMouseMove,
    Cesium.ScreenSpaceEventType.MOUSE_MOVE,
  );

  //右键结束绘制当前polygon
  handler.setInputAction(
    handleRightClick,
    Cesium.ScreenSpaceEventType.RIGHT_CLICK,
  );

  //拖动顶点
  handler.setInputAction(handleLeftDown, Cesium.ScreenSpaceEventType.LEFT_DOWN);

  //左键抬起
  handler.setInputAction(handleLeftUp, Cesium.ScreenSpaceEventType.LEFT_UP);
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
</style>
