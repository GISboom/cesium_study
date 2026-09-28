<template>
  <div id="cesiumContainer"></div>
  <div class="toolbar"></div>
</template>
<script setup>
import { onMounted, onUnmounted, ref } from "vue";
import * as Cesium from "cesium";
import "cesium/Build/Cesium/Widgets/widgets.css";
let viewer;
let handler;

// const geojson = {
//   type: "FeatureCollection",

//   features: [
//     {
//       type: "Feature",

//       properties: {
//         name: "测试区域",
//         type: "area",
//       },

//       geometry: {
//         type: "Polygon",

//         coordinates: [
//           [
//             [116.35, 39.9],
//             [116.45, 39.9],
//             [116.45, 39.95],
//             [116.35, 39.95],
//             [116.35, 39.9],
//           ],
//         ],
//       },
//     },
//   ],
// };

const landuse = {
  type: "FeatureCollection",
  features: [
    {
      type: "Feature",
      properties: {
        id: 1,
        name: "林地区域",
        landType: "forest",
      },
      geometry: {
        type: "Polygon",
        coordinates: [
          [
            [116.35, 39.9],
            [116.38, 39.9],
            [116.38, 39.93],
            [116.35, 39.93],
            [116.35, 39.9],
          ],
        ],
      },
    },
    {
      type: "Feature",
      properties: {
        id: 2,
        name: "水域区域",
        landType: "water",
      },
      geometry: {
        type: "Polygon",
        coordinates: [
          [
            [116.39, 39.9],
            [116.42, 39.9],
            [116.42, 39.93],
            [116.39, 39.93],
            [116.39, 39.9],
          ],
        ],
      },
    },
    {
      type: "Feature",
      properties: {
        id: 3,
        name: "建设区域",
        landType: "building",
      },
      geometry: {
        type: "Polygon",
        coordinates: [
          [
            [116.43, 39.9],
            [116.46, 39.9],
            [116.46, 39.93],
            [116.43, 39.93],
            [116.43, 39.9],
          ],
        ],
      },
    },
  ],
};

function clickHandler(click) {
  const pick = viewer.scene.pick(click.position);
  if (!pick) return;
  console.log(pick.id);
}

function getLandTypeColor(landType) {
  const colors = {
    forest:
      Cesium.Color.GREEN.withAlpha(0.5),
    water:
      Cesium.Color.BLUE.withAlpha(0.5),
    building:
      Cesium.Color.RED.withAlpha(0.5),
    farmland:
      Cesium.Color.YELLOW.withAlpha(0.5),
  };

  return (
    colors[landType] ??
    Cesium.Color.GRAY.withAlpha(0.5)
  );
}
function styleGeoJsonEntities(dataSource){
  //获取加载geojson的entities,记住要.values
  const entities = dataSource.entities.values;
  entities.forEach((entity) => {
    if(!entity.polygon) return;
    //取到属性
    const landType = entity.properties.landType.getValue();
    //根据属性渲染样式
    if (!landType) return;
    entity.polygon.material = getLandTypeColor(landType);
    entity.polygon.outline = true;//显示边框线，与clampToGround贴地冲突
    entity.polygon.outlineColor = Cesium.Color.WHITE;
    entity.polygon.outlineWidth = 10;
  })
}
onMounted(async () => {
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

  //加载geojson
  // const geojsonDataSource = await Cesium.GeoJsonDataSource.load(geojson, {
  //   stroke: Cesium.Color.YELLOW,

  //   fill: Cesium.Color.RED.withAlpha(0.4),

  //   strokeWidth: 3,

  //   clampToGround: true, //贴地
  // });

  // viewer.dataSources.add(geojsonDataSource);
  // viewer.flyTo(geojsonDataSource);

  // //获取加载geojson的entity
  // const entities = geojsonDataSource.entities.values[0];
  // // console.log(entities.properties.name.getValue());

  // const ChinaGeoJSON = await Cesium.GeoJsonDataSource.load(
  //   "public/provience.geojson",
  //   {
  //     stroke: Cesium.Color.YELLOW,
  //     fill: Cesium.Color.RED.withAlpha(0.1),
  //     strokeWidth: 10,
  //     clampToGround: true, //贴地
  //   },
  // );
  // viewer.dataSources.add(ChinaGeoJSON);
  // viewer.flyTo(ChinaGeoJSON);

  //加载landuse
  const landuseDataSource = await Cesium.GeoJsonDataSource.load(landuse, {
    // clampToGround: true, //贴地,与outline冲突
  });
  viewer.dataSources.add(landuseDataSource);
  viewer.flyTo(landuseDataSource);

  //根据属性渲染样式
  styleGeoJsonEntities(landuseDataSource);


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
