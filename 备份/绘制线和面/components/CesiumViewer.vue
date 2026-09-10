<template>
  <div id="cesiumContainer"></div>
</template>
<script setup>
import { onMounted, onUnmounted } from "vue";
import * as Cesium from "cesium";
import "cesium/Build/Cesium/Widgets/widgets.css";
let viewer;
let handler;
// //折线
// let polylinePositions = [];
// let polylineEntity = null;
//面
let polygonPositions = [];
let polygonEntity = null;
onMounted(() => {
  viewer = new Cesium.Viewer("cesiumContainer");
  const camera = viewer.camera;
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

  //鼠标事件
  handler = new Cesium.ScreenSpaceEventHandler(viewer.scene.canvas);
  //监听鼠标左键点击事件
  handler.setInputAction((click) => {
    const cartesian = viewer.camera.pickEllipsoid(
      click.position,
      viewer.scene.globe.ellipsoid,
    );
    // //保存
    // polylinePositions.push(cartesian);
    // // 至少两个点才能形成线
    // if (polylinePositions.length < 2) {
    //   return;
    // }
    // // 如果已经存在折线实体，先移除它
    // if (polylineEntity) {
    //   viewer.entities.remove(polylineEntity);
    // }
    // // 创建新的折线实体
    // polylineEntity = viewer.entities.add({
    //   polyline: {
    //     positions: polylinePositions,
    //     width: 5,
    //     material: Cesium.Color.RED,
    //   },
    // });

    if (!cartesian) {
      return;
    }

    //保存
    polygonPositions.push(cartesian);
    if (polygonPositions.length < 3) {
      return;
    }
    // // 如果已经存在面实体，先移除它
    // if (polygonEntity) {
    //   viewer.entities.remove(polygonEntity);
    // }
    // 创建新的面实体
    if (!polygonEntity) {
      polygonEntity = viewer.entities.add({
        name:'动态绘制面',
        polygon: {
          //动态获取hierarchy
          //Entity本身没有重新创建
          //false表示：这个 Property 的值是不是永远不变？
          hierarchy: new Cesium.CallbackProperty(() => {
            return new Cesium.PolygonHierarchy([...polygonPositions]);
          }, false),
          material: Cesium.Color.BLUE.withAlpha(0.5),
          outline: true,
          // outlineWidth: 20,//设置不了边界宽度
          outlineColor: Cesium.Color.BLACK,
        },
      });
    }
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
