<template>
  <div id="cesiumContainer"></div>
  <div class="toolbar"></div>
  <!-- 高程剖面图 -->
  <div class="profile-panel">
    <div ref="profileChartRef" class="profile-chart"></div>
  </div>
</template>
<script setup>
import { onMounted, onUnmounted, reactive, ref } from "vue";
import * as Cesium from "cesium";
import "cesium/Build/Cesium/Widgets/widgets.css";
import * as echarts from "echarts";

//起点与终点，等分100份，101个点，再批量查询地形高度，形成地形剖面

let viewer;
let handler;
let terrainProvider;

let profileChartRef = ref(null);
let profileChart = null;

//起点与终点坐标
const start = Cesium.Cartographic.fromDegrees(115.35, 28.9);
const end = Cesium.Cartographic.fromDegrees(115.45, 28.95);

const sampleCount = 100;

function clickHandler(click) {
  const ray = viewer.camera.getPickRay(click.position);
  const cartesian = viewer.scene.globe.pick(ray, viewer.scene);
  if (!cartesian) return;
  const cartographic = Cesium.Cartographic.fromCartesian(cartesian);
  getHeight(cartographic);
}
async function getHeight(cartographic) {
  const positions = [cartographic];
  //通过经纬度查询地形高程
  //Cesium.sampleTerrainMostDetailed得到的结果是Promise,需要await
  const result = await Cesium.sampleTerrainMostDetailed(
    viewer.scene.terrainProvider,
    positions,
  );
  //result包括了经纬度和地形高程
  console.log(result[0].height.toFixed(2));
}

//获取地形剖面
async function getTerrainProfile(start, end, sampleCount) {
  //创建测地线
  const geodesic = new Cesium.EllipsoidGeodesic(start, end);
  //计算距离
  const distance = geodesic.surfaceDistance;
  //生成采样点
  const positions = [];
  for (let i = 0; i <= sampleCount; i++) {
    //第i个采样点（等分点）
    const fraction = i / sampleCount;
    //采样点经纬度interpolateUsingFraction
    const position = geodesic.interpolateUsingFraction(fraction);
    positions.push(position);
  }

  //查询Terrain高度
  const sampledPositions = await Cesium.sampleTerrainMostDetailed(
    viewer.scene.terrainProvider,
    positions,
  );

  //整理剖面数据
  const profile = sampledPositions.map((position, index) => {
    const fraction = index / sampleCount;
    return {
      distance: distance * fraction, //距离
      height: position.height, //高度
      latitude: Cesium.Math.toDegrees(position.latitude), //纬度,将弧度转为角度
      longitude: Cesium.Math.toDegrees(position.longitude), //经度
    };
  });
  return profile;
}

//初始化图表
function initProfileChart() {
  if (!profileChartRef.value) return;
  profileChart = echarts.init(profileChartRef.value);
  const option = {
    title: {
      text: "请选择 A、B 两点进行剖面分析",
      left: "center",
      top: 10,
      textStyle: {
        fontSize: 14,
      },
    },
    tooltip: {
      trigger: "axis",
    },
    grid: {
      left: 50,
      right: 20,
      top: 60,
      bottom: 40,
    },
    xAxis: {
      type: "category",
      name: "距离 (km)",
      data: [],
    },
    yAxis: {
      type: "value",
      name: "高程 (m)",
    },
    series: [
      {
        type: "line",
        data: [],
        smooth: true,
        areaStyle: {},
      },
    ],
  };
  profileChart.setOption(option);
}
//更新剖面图
function renderProfileChart(profileData) {
  if (!profileChart) return;
  if (!profileData || profileData.length === 0) return;
  // 横轴：距离，转成 km
  const xData = profileData.map((item) => (item.distance / 1000).toFixed(2));
  // 纵轴：高程
  const yData = profileData.map((item) => Number(item.height.toFixed(2)));
  const option = {
    title: {
      text: "Terrain 高程剖面图",
      left: "center",
      top: 10,
      textStyle: {
        fontSize: 14,
      },
    },
    tooltip: {
      trigger: "axis",
      formatter: function (params) {
        const index = params[0].dataIndex;
        const item = profileData[index];
        return `
          距离：${(item.distance / 1000).toFixed(2)} km<br/>
          高程：${item.height.toFixed(2)} m<br/>
          经度：${item.longitude.toFixed(6)}<br/>
          纬度：${item.latitude.toFixed(6)}
        `;
      },
    },
    grid: {
      left: 60,
      right: 20,
      top: 60,
      bottom: 50,
    },
    xAxis: {
      type: "category",
      name: "距离 (km)",
      data: xData,
    },
    yAxis: {
      type: "value",
      name: "高程 (m)",
    },
    series: [
      {
        name: "高程",
        type: "line",
        data: yData,
        smooth: true,
        areaStyle: {},
        symbol: "none",
      },
    ],
  };
  profileChart.setOption(option);
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
    sceneModePicker: false,
    navigationHelpButton: false,
    // baseLayerPicker: false,
    terrainProvider,
  });
  // 开启地形光照（山体立体感更强）
  // viewer.scene.globe.enableLighting = true;
  const camera = viewer.camera;
  camera.setView({
    destination: Cesium.Cartesian3.fromDegrees(115.397428, 28.91923, 800.0),
    //方向角
    orientation: {
      heading: Cesium.Math.toRadians(0.0), // 方向角，单位为弧度
      pitch: Cesium.Math.toRadians(-45.0), // 俯仰角，单位为弧度
      roll: 0.0, // 翻滚角，单位为弧度
    },
  });

  const profile = await getTerrainProfile(start, end, sampleCount);
  const linePositions = profile.map((item) => {
    return Cesium.Cartesian3.fromDegrees(
      item.longitude,
      item.latitude,
      item.height,
    );
  });
  viewer.entities.add({
    polyline: {
      positions: linePositions,
      width: 3,
      material: Cesium.Color.RED,
    },
  });
  //鼠标事件
  handler = new Cesium.ScreenSpaceEventHandler(viewer.scene.canvas);
  //点击事件
  handler.setInputAction(clickHandler, Cesium.ScreenSpaceEventType.LEFT_CLICK);
  initProfileChart();
  renderProfileChart(profile);
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
  if (profileChart) {
    profileChart.dispose();
    profileChart = null;
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
.profile-panel {
  position: absolute;
  right: 20px;
  bottom: 20px;
  width: 620px;
  height: 280px;
  background: rgba(255, 255, 255, 0.95);
  border-radius: 8px;
  z-index: 1000;
  overflow: hidden;
}
.profile-chart {
  width: 100%;
  height: 100%;
}
</style>
