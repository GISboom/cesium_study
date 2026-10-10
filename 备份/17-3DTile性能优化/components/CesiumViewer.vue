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
async function loadBuildings() {
  try {
    buildingTileset = await Cesium.createOsmBuildingsAsync({
      // 设置最大屏幕空间误差，降低值可以提高细节，但可能影响性能
      //数字越小，建筑物的细节越高，但可能会影响性能。根据实际需求调整该值。
      maximumScreenSpaceError: 64,
      //让远处 Tile 使用较低分辨率，从而减少加载量。
      dynamicScreenSpaceError: true,
      // 设置缓存大小，单位为字节
      cacheBytes: 256 * 1024 * 1024,
      //允许当前必要 Tile 在缓存预算上额外超出一部分。
      maximumCacheOverflowBytes:256 * 1024 * 1024,
    });
    viewer.scene.primitives.add(buildingTileset);
    viewer.scene.debugShowFramesPerSecond = true; // 显示帧率
    // 监听建筑物加载完成事件allTilesLoaded
    buildingTileset.allTilesLoaded.addEventListener(() => {
      const memoryMB = buildingTileset.totalMemoryUsageInBytes / 1024 / 1024;
      console.log(
        "maximumScreenSpaceError：",
        buildingTileset.maximumScreenSpaceError,
      );
      console.log("GPU内存估算：", memoryMB.toFixed(2), "MB");
    });
  } catch (e) {
    console.error("加载建筑物失败", e);
  }
}

function clickHandler(click) {
  const picked = viewer.scene.pick(click.position);
  //判断是否点击到建筑物
  if (picked instanceof Cesium.Cesium3DTileFeature) {
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
    // sceneModePicker: false,
    navigationHelpButton: false,
    // baseLayerPicker: false,
    terrainProvider,
  });
  // 开启地形光照（山体立体感更强）
  // viewer.scene.globe.enableLighting = true;
  terrainProvider.maximumScreenSpaceError = 64; // 设置最大屏幕空间误差，降低值可以提高地形细节，但可能影响性能

  await loadBuildings();
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
