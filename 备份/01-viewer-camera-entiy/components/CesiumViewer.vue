<template>
  <div id="cesiumContainer"></div>
</template>
<script setup>
import { onMounted, onUnmounted } from "vue";
import * as Cesium from "cesium";
import "cesium/Build/Cesium/Widgets/widgets.css";
let viewer;
let handler;

onMounted(() => {
  viewer = new Cesium.Viewer("cesiumContainer");
  const camera = viewer.camera;
  //   camera.setView({
  //     //Cesium 的 Cartesian3 是三维笛卡尔坐标
  //     //所以需要进行转换:fromDegrees,将经纬度坐标转换为笛卡尔坐标
  //     destination: Cesium.Cartesian3.fromDegrees(116.39, 39.90, 100000.0),
  //   });
  camera.flyTo({
    destination: Cesium.Cartesian3.fromDegrees(116.39, 39.9, 10000.0),
    duration: 2, //飞行时间，单位为秒

    //方向角
    orientation: {
      heading: Cesium.Math.toRadians(0.0), // 方向角，单位为弧度
      pitch: Cesium.Math.toRadians(-90.0), // 俯仰角，单位为弧度
      roll: 0.0, // 翻滚角，单位为弧度
    },
  });

  //坐标转换
  //将经纬度坐标转换为笛卡尔坐标
  //   const cartesian = Cesium.Cartesian3.fromDegrees(116.39, 39.9, 1000);
  //   console.log(cartesian);
  //将笛卡尔坐标转换为经纬度坐标，但是经纬度坐标是弧度制的，需要转换为角度制
  //   const cartographic = Cesium.Cartographic.fromCartesian(cartesian);
  //   console.log(cartographic);
  //将弧度制的经纬度坐标转换为角度制
  //   const longitude = Cesium.Math.toDegrees(cartographic.longitude);
  //   const latitude = Cesium.Math.toDegrees(cartographic.latitude);
  //   console.log(`Longitude: ${longitude}, Latitude: ${latitude}`);

  //鼠标事件
  handler = new Cesium.ScreenSpaceEventHandler(viewer.scene.canvas);
  //监听鼠标左键点击事件
  handler.setInputAction((click) => {
    // // 获取鼠标点击位置的屏幕坐标
    // console.log(click.position);
    //获取鼠标点击位置的笛卡尔坐标
    const cartesian = viewer.camera.pickEllipsoid(
      click.position,
      viewer.scene.globe.ellipsoid,
    );
    //将笛卡尔坐标转换为经纬度坐标(弧度制)
    const cartographic = Cesium.Cartographic.fromCartesian(cartesian);
    //将弧度制的经纬度坐标转换为角度制
    const longitude = Cesium.Math.toDegrees(cartographic.longitude);
    const latitude = Cesium.Math.toDegrees(cartographic.latitude);
    //高度为0,
    //pickEllipsoid() 得到的是地球椭球面上的位置，
    // 所以 Cartographic.height 基本就是 0，它不是实际地形高程。
    const height = cartographic.height;

    //在鼠标点击位置添加一个点
    const entity = {
      //笛卡尔坐标
      position: cartesian,
      billboard: {
        image: "public/img/坐标-fill.png",
        width: 32,
        height: 32,
      },
      label: {
        text: `Longitude: ${longitude.toFixed(2)},\n Latitude: ${latitude.toFixed(2)}, \nHeight: ${height.toFixed(2)}`,
        font: "16px sans-serif",
        pixelOffset: new Cesium.Cartesian2(0, -50),
      },
    };
    viewer.entities.add(entity);
  }, Cesium.ScreenSpaceEventType.LEFT_CLICK);
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
