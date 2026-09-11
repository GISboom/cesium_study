<template>
  <div id="cesiumContainer"></div>
</template>
<script setup>
import { onMounted, onUnmounted } from "vue";
import * as Cesium from "cesium";
import "cesium/Build/Cesium/Widgets/widgets.css";
let viewer;
let handler;

//面
let currentPositions = [];
let currentPolygon = null;
let mousePosition = null;

//预览线
let previewLineEntity = null;

onMounted(() => {
  viewer = new Cesium.Viewer("cesiumContainer");
  const camera = viewer.camera;
  camera.setView({
    destination: Cesium.Cartesian3.fromDegrees(116.397428, 39.90923, 10000.0),
  });

  //鼠标事件
  handler = new Cesium.ScreenSpaceEventHandler(viewer.scene.canvas);

  //左键添加点
  handler.setInputAction((click) => {
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
  }, Cesium.ScreenSpaceEventType.LEFT_CLICK);

  // 鼠标移动：更新临时点
  handler.setInputAction((movement) => {
    const ray = viewer.camera.getPickRay(movement.endPosition);
    const cartesian = viewer.scene.globe.pick(ray, viewer.scene);
    if (!cartesian) return;
    mousePosition = cartesian; // 更新鼠标当前位置

    if (currentPositions.length < 2) return;
    if (currentPolygon) viewer.entities.remove(currentPolygon);
    
    currentPolygon = viewer.entities.add({
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
  }, Cesium.ScreenSpaceEventType.MOUSE_MOVE);

  //右键结束绘制
  handler.setInputAction((click) => {
    if (currentPositions.length >= 3) {
      //把临时预览实体转为永久实体
      viewer.entities.add({
        polygon: {
          hierarchy: new Cesium.PolygonHierarchy(currentPositions),
          material: Cesium.Color.ORANGE.withAlpha(0.4),
          outline: true,
        },
      });
    }
    if (currentPolygon) viewer.entities.remove(currentPolygon);
    //清空当前绘制数据
    currentPolygon = null;
    currentPositions = [];
    mousePosition = null;
    //移除预览线
    if (previewLineEntity) viewer.entities.remove(previewLineEntity);
  }, Cesium.ScreenSpaceEventType.RIGHT_CLICK);
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
</style>
