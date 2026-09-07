<template>
  <div id="cesiumContainer"></div>
</template>
<script setup>
import { onMounted } from "vue";
import * as Cesium from "cesium";
import "cesium/Build/Cesium/Widgets/widgets.css";

onMounted(() => {
  const viewer = new Cesium.Viewer("cesiumContainer");
  const camera = viewer.camera;
  //   camera.setView({
  //     //Cesium 的 Cartesian3 是三维笛卡尔坐标
  //     //所以需要进行转换:fromDegrees,将经纬度坐标转换为笛卡尔坐标
  //     destination: Cesium.Cartesian3.fromDegrees(116.39, 39.90, 100000.0),
  //   });
  camera.flyTo({
    destination: Cesium.Cartesian3.fromDegrees(116.39, 39.9, 10000.0),
    duration: 5, //飞行时间，单位为秒

    //方向角
    orientation: {
      heading: Cesium.Math.toRadians(0.0), // 方向角，单位为弧度
      pitch: Cesium.Math.toRadians(-90.0), // 俯仰角，单位为弧度
      roll: 0.0, // 翻滚角，单位为弧度
    },
  });

  //坐标转换
  //将经纬度坐标转换为笛卡尔坐标
  const cartesian = Cesium.Cartesian3.fromDegrees(116.39, 39.9, 1000);
//   console.log(cartesian);
  //将笛卡尔坐标转换为经纬度坐标，但是经纬度坐标是弧度制的，需要转换为角度制
  const cartographic = Cesium.Cartographic.fromCartesian(cartesian);
//   console.log(cartographic);
  //将弧度制的经纬度坐标转换为角度制
  const longitude = Cesium.Math.toDegrees(cartographic.longitude);
  const latitude = Cesium.Math.toDegrees(cartographic.latitude);
//   console.log(`Longitude: ${longitude}, Latitude: ${latitude}`);
});
</script>
<style scoped>
#cesiumContainer {
  width: 100%;
  height: 100vh;
}
</style>
