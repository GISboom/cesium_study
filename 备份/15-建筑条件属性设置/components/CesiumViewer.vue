<template>
  <div id="cesiumContainer"></div>
  <div class="toolbar"></div>
</template>
<script setup>
import { onMounted, onUnmounted, reactive, ref } from "vue";
import * as Cesium from "cesium";
import "cesium/Build/Cesium/Widgets/widgets.css";

let viewer;
let handler;
let terrainProvider;

//3dtileset
let buildingTileset = null;
let selectedBuildingName = ref(null);

let selectedFeature = null;
let originalColor = null;

//加载建筑物
async function loadBuildings(){
  try{
    buildingTileset = await Cesium.createOsmBuildingsAsync({
      url: "https://data.osmbuildings.org/0.2/rkc8w3l6/tile/{z}/{x}/{y}.json",
    });
    viewer.scene.primitives.add(buildingTileset);
  }catch(e){
    console.error("加载建筑物失败", e);
  }
}

function clickHandler(click) {
  const picked = viewer.scene.pick(click.position);
  //判断是否点击到建筑物
  if(picked instanceof Cesium.Cesium3DTileFeature){
    //获取属性字段
    const propertyIds = picked.getPropertyIds();
    const properties = {};
    //获取属性
    propertyIds.forEach((prop) => {
      properties[prop] = picked.getProperty(prop);
    });
    selectedBuildingName.value = properties.name;
    console.log("点击到建筑物", properties["cesium#estimatedHeight"]);
    // console.log("点击到建筑物", picked);

    //点击高亮
    highlightFeature(picked);
  } else {
    console.log("未点击到建筑物");
  }
}

//高亮显示
function highlightFeature(feature) {
  if (selectedFeature) {
    selectedFeature.color = originalColor;
  }
  selectedFeature = feature;
  originalColor = feature.color;
  feature.color = Cesium.Color.YELLOW;
}

// //条件样式：根据建筑物高度设置颜色
// function setBuildingColorByHeight() {
//   buildingTileset.style = new Cesium.Cesium3DTileStyle({
//     color: {
//       conditions: [
//         ["${feature['cesium#estimatedHeight']} >= 10", "color('red')"],
//         ["${feature['cesium#estimatedHeight']} >= 5", "color('orange')"],
//         ["true", "color('green')"],
//       ],
//     },
//   });
// }

// //根据高度控制显隐
// function setVisibilityByHeight() {
//   if (!buildingTileset) return;
//   buildingTileset.style = new Cesium.Cesium3DTileStyle({
//     show: "${feature['cesium#estimatedHeight']} >= 10",
//   });
// }

onMounted(async () => {
  //加载地形
  terrainProvider = await Cesium.createWorldTerrainAsync();
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
    terrainProvider,
  });
  // 开启地形光照（山体立体感更强）
  // viewer.scene.globe.enableLighting = true;

  await loadBuildings();
  // setBuildingColorByHeight();
  // setVisibilityByHeight();

  //相机视角
  const camera = viewer.camera;
  camera.setView({
    destination: Cesium.Cartesian3.fromDegrees(116.39068, 39.91283, 400.0),
    //方向角
    orientation: {
      heading: Cesium.Math.toRadians(0.0), // 方向角，单位为弧度
      pitch: Cesium.Math.toRadians(-45.0), // 俯仰角，单位为弧度
      roll: 0.0, // 翻滚角，单位为弧度
    },
  });

  //鼠标事件
  handler = new Cesium.ScreenSpaceEventHandler(viewer.scene.canvas);
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
