<template>
  <div id="cesiumContainer"></div>
  <div class="toolbar"></div>
</template>
<script setup>
import { onMounted, onUnmounted, reactive, ref } from "vue";
import * as Cesium from "cesium";
import "cesium/Build/Cesium/Widgets/widgets.css";
/*
 * TerrainProvider：专门给 Cesium 提供地形高程瓦片的数据源，viewer.terrainProvider
 * WGS84椭球高，地形高，对地高度
 * sampleTerrainMostDetailed：给定一些经纬度位置，查询 Terrain 上对应的高程
 * Terrain 和 clampToGround：让线/面贴着 Globe/Terrain 表面
 * Terrain 和 HeightReference：
 *    1.Cesium.HeightReference.NONE：使用对象本身的高度
 *    2.Cesium.HeightReference.CLAMP_TO_GROUND：直接贴地
 *    3.Cesium.HeightReference.RELATIVE_TO_GROUND：在地形表面基础上，再增加一定高度
 */

let viewer;
let handler;
function clickHandler(click) {
  //点击处的笛卡尔坐标click.position
  //将笛卡尔坐标转为经纬度坐标
  // //方法1：
  // const cartesian1 = viewer.camera.pickEllipsoid(
  //   click.position,
  //   viewer.scene.globe.ellipsoid,
  // );
  // console.log("cartesian1:", cartesian1);

  //方法2：
  const ray = viewer.camera.getPickRay(click.position);
  const cartesian = viewer.scene.globe.pick(ray, viewer.scene);
  if (!cartesian) return;
  const cartographic = Cesium.Cartographic.fromCartesian(cartesian);
  getHeight(cartographic)
}
async function getHeight(cartographic) {
  const positions = [cartographic]
  //通过经纬度查询地形高程
  //Cesium.sampleTerrainMostDetailed得到的结果是Promise,需要await
  const result = await Cesium.sampleTerrainMostDetailed(
    viewer.scene.terrainProvider,
    positions,
  );
  //result包括了经纬度和地形高程
  console.log(result[0].height.toFixed(2));
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
    terrain: Cesium.Terrain.fromWorldTerrain({
      requestVertexNormals: true, // 开启地形光照，山体明暗效果
      requestWaterMask: true, // 水面掩膜，水域显示平整
    }),
  });
  // 开启地形光照（山体立体感更强）
  // viewer.scene.globe.enableLighting = true;
  const camera = viewer.camera;
  camera.setView({
    destination: Cesium.Cartesian3.fromDegrees(115.597428, 28.90923, 800.0),
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
