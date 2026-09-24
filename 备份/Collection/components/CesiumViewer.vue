<template>
  <div id="cesiumContainer"></div>
  <div class="toolbar">
    <button @click="addPoints">100个点不要重复添加</button>
    <button @click="addBillBoard">添加图标和标签</button>
  </div>
</template>
<script setup>
import { onMounted, onUnmounted, ref } from "vue";
import * as Cesium from "cesium";
import "cesium/Build/Cesium/Widgets/widgets.css";
let viewer;
let handler;

let pointCollection = null;

let billboardCollection = null;

let labelCollection = null;

function clickHandler(click) {
  const pick = viewer.scene.pick(click.position);
  if (!pick) return;
  console.log(pick.id);
}
function addPoints() {
  //PointPrimitiveCollection
  pointCollection = new Cesium.PointPrimitiveCollection();
  viewer.scene.primitives.add(pointCollection);
  //添加1000个点
  for (let i = 0; i < 1000; i++) {
    pointCollection.add({
      id: "point" + i,
      position: Cesium.Cartesian3.fromDegrees(
        116.397428 + Math.random() * 0.1,
        39.90923 + Math.random() * 0.1,
      ),
      pixelSize: 10,
      color: Cesium.Color.RED,
    });
  }
}

function addBillBoard() {
  //BillboardCollection
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

  billboardCollection = new Cesium.BillboardCollection();
  viewer.scene.primitives.add(billboardCollection);
  labelCollection = new Cesium.LabelCollection();
  viewer.scene.primitives.add(labelCollection);
  const position = Cesium.Cartesian3.fromDegrees(116.397428, 39.90923);
  billboardCollection.add({
    id: "billboard",
    position: position,
    image: "public/img/坐标-fill.png",
    width: 100,
    height: 100,
    verticalOrigin: Cesium.VerticalOrigin.BOTTOM,
    scale: 0.5,
  });
  labelCollection.add({
    id: "label",
    position: position,
    text: "站点",
    font: "16px sans-serif",
    fillColor: Cesium.Color.WHITE,

    outlineColor: Cesium.Color.BLACK,

    outlineWidth: 2,

    style: Cesium.LabelStyle.FILL_AND_OUTLINE,

    pixelOffset: new Cesium.Cartesian2(-10, 30),
  });

  //点击事件
  handler.setInputAction(clickHandler, Cesium.ScreenSpaceEventType.LEFT_CLICK);
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
