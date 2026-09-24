<template>
  <div id="cesiumContainer"></div>
  <div class="toolbar">
    <!-- <button @click="addPolygonPrimitive">添加 Polygon Primitive</button>

    <button @click="removePolygonPrimitive">删除 Polygon Primitive</button> -->
  </div>
</template>
<script setup>
import { onMounted, onUnmounted, ref } from "vue";
import * as Cesium from "cesium";
import "cesium/Build/Cesium/Widgets/widgets.css";
let viewer;
let handler;

let primitive = null;

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

  //批量绘制多个图形
  //1.多边形
  const polygonPositions = [
    116.35, 39.9, 116.45, 39.9, 116.45, 39.95, 116.35, 39.95,
  ];
  const polygonGeometry = new Cesium.PolygonGeometry({
    polygonHierarchy: new Cesium.PolygonHierarchy(
      Cesium.Cartesian3.fromDegreesArray(polygonPositions),
    ),
    vertexFormat: Cesium.PerInstanceColorAppearance.VERTEX_FORMAT,
  });
  const polygonInstance = new Cesium.GeometryInstance({
    id: "polygon",
    geometry: polygonGeometry,
    attributes: {
      color: Cesium.ColorGeometryInstanceAttribute.fromColor(
        Cesium.Color.RED.withAlpha(0.6),
      ),
    },
  });

  //2.圆
  const circleGeometry = new Cesium.CircleGeometry({
    center: Cesium.Cartesian3.fromDegrees(116.3, 39.92),
    radius: 1000.0,
    vertexFormat: Cesium.PerInstanceColorAppearance.VERTEX_FORMAT,
  });
  const circleInstance = new Cesium.GeometryInstance({
    id: "circle",
    geometry: circleGeometry,
    attributes: {
      color: Cesium.ColorGeometryInstanceAttribute.fromColor(
        Cesium.Color.BLUE.withAlpha(0.6),
      ),
    },
  });

  //3.折线
  const polylinePositions = [
    116.35, 39.89, 116.46, 39.9, 116.46, 39.95, 116.35, 39.96,
  ];
  //普通折线：PolylineGeometry和PolylineColorAppearance
  //注意：折线贴地需使用：GroundPolylineGeometry和PolylineMaterialAppearance和Cesium.GroundPolylinePrimitive
  const polylineGeometry = new Cesium.GroundPolylineGeometry({
    positions: Cesium.Cartesian3.fromDegreesArray(polylinePositions),
    width: 10.0,
    vertexFormat: Cesium.PerInstanceColorAppearance.VERTEX_FORMAT,
  });
  const polylineInstance = new Cesium.GeometryInstance({
    id: "polyline",
    geometry: polylineGeometry,
    attributes: {
      color: Cesium.ColorGeometryInstanceAttribute.fromColor(
        Cesium.Color.GREEN.withAlpha(0.8),
      ),
    },
  });

  const apprearance = new Cesium.PerInstanceColorAppearance({
    translucent: false, //允许透明
    closed: false, //把几何对象按照封闭表面来处理
  });

  //GroundPremitive
  //GroundPremitive不适合建筑物、高空对象
  primitive = new Cesium.GroundPrimitive({
    geometryInstances: [polygonInstance, circleInstance],
    appearance: apprearance,
  });
  viewer.scene.primitives.add(primitive);
  const polylinePremitive = new Cesium.GroundPolylinePrimitive({
    geometryInstances: polylineInstance,
    appearance: new Cesium.PolylineMaterialAppearance({
      material: Cesium.Material.fromType("Color", {
        color: Cesium.Color.RED,
      }),
    }),
  });
  viewer.scene.primitives.add(polylinePremitive);
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
