<template>
  <div id="cesiumContainer"></div>
  <div class="toolbar">
    <button @click="addPolygonPrimitive">添加 Polygon Primitive</button>

    <button @click="removePolygonPrimitive">删除 Polygon Primitive</button>
  </div>
</template>
<script setup>
import { onMounted, onUnmounted, ref } from "vue";
import * as Cesium from "cesium";
import "cesium/Build/Cesium/Widgets/widgets.css";
let viewer;
let handler;

let premitive = null;
let geometryInstance = null;
let geometry = null;
let apprearance = null;

//存储已选择的
let selectedPremitive = null;
let selectedInstanceId = null;
//初始颜色
let selectedOriginColor = null;

function addPolygonPrimitive() {
  if (premitive) {
    removePolygonPrimitive();
    return;
  }
  //几何属性
  geometry = new Cesium.RectangleGeometry({
    rectangle: Cesium.Rectangle.fromDegrees(116.3, 39.9, 116.4, 40.0),
    //这个 Geometry 最后需要生成哪些顶点属性
    vertexFormat: Cesium.PerInstanceColorAppearance.VERTEX_FORMAT,
  });

  //样式
  apprearance = new Cesium.PerInstanceColorAppearance({
    translucent: true, //允许透明
    closed: true, //把几何对象按照封闭表面来处理
  });

  //几何实例
  geometryInstance = new Cesium.GeometryInstance({
    geometry: geometry,
    attributes: {
      color: Cesium.ColorGeometryInstanceAttribute.fromColor(
        Cesium.Color.RED.withAlpha(0.6),
      ),
    },
  });

  //
  premitive = new Cesium.Primitive({
    geometryInstances: geometryInstance,
    appearance: apprearance,
  });
  viewer.scene.primitives.add(premitive);
}

function removePolygonPrimitive() {
  if (premitive) {
    viewer.scene.primitives.remove(premitive);
    premitive = null;
  }
}

function getGeometryID(click) {
  // console.log(click);
  const picked = viewer.scene.pick(click.position);
  if (!picked){
    clearPicked();
    return;
  } 
  const id = picked.id;
  const pickedPremitive = picked.primitive;
  if (!pickedPremitive || !id) {
    return;
  }
  clearPicked();
  selectedPremitive = pickedPremitive;
  selectedInstanceId = id;

  //核心API
  // console.log(pickedPremitive.getGeometryInstanceAttributes(id).color);
  //获取到几何实例的属性
  const attributes = pickedPremitive.getGeometryInstanceAttributes(id);
  if (!attributes) return;
  selectedOriginColor = attributes.color;
  //修改颜色
  attributes.color = Cesium.ColorGeometryInstanceAttribute.toValue(
    Cesium.Color.BLUE.withAlpha(0.6),
  );
}
//取消选中，恢复颜色
function clearPicked() {
  if (
    !selectedPremitive ||
    selectedInstanceId === null ||
    selectedInstanceId === undefined ||
    !selectedOriginColor
  ) {
    return;
  }
  //已点击的几何实例的属性
  const attributes = selectedPremitive.getGeometryInstanceAttributes(selectedInstanceId);
  if (!attributes) return;
  attributes.color = selectedOriginColor;
  selectedPremitive = null;
  selectedInstanceId = null;
  selectedOriginColor = null;
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
  const polylineGeometry = new Cesium.PolylineGeometry({
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
    translucent: true, //允许透明
    closed: true, //把几何对象按照封闭表面来处理
  });

  //Premitive
  //不是什么 Geometry 都可以往一个 Primitive 里面塞
  //多个GeometryInstance它们的顶点结构兼容，能够共同使用同一个Appearance，才能放进同一个Primitive
  premitive = new Cesium.Primitive({
    geometryInstances: [polygonInstance, circleInstance],
    appearance: apprearance,
  });
  viewer.scene.primitives.add(premitive);
  const polylinePremitive = new Cesium.Primitive({
    geometryInstances: polylineInstance,
    appearance: new Cesium.PolylineColorAppearance(),
  });
  viewer.scene.primitives.add(polylinePremitive);

  //绑定点击事件
  handler.setInputAction(getGeometryID, Cesium.ScreenSpaceEventType.LEFT_CLICK);
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
