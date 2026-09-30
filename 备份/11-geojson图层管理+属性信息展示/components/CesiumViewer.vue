<template>
  <div id="cesiumContainer"></div>
  <div class="toolbar"></div>
  <div class="layer-panel">
    <!-- 图层 -->
    <div class="layer-title">
      <label>
        <input type="checkbox" v-model="layerVisible" @change="toggleLayer" />
        土地利用图层
      </label>
    </div>
    <!-- 图例 -->
    <div class="legend-title">图例</div>
    <div
      v-for="(style, type) in landTypeStyles"
      :key="type"
      class="legend-item"
    >
      <input
        type="checkbox"
        :disabled="!layerVisible"
        v-model="landTypeVisibel[type]"
        @change="toggleLandType(type)"
      />
      <span
        class="legend-color"
        :style="{
          backgroundColor: style.cssColor,
        }"
      >
      </span>
      <span>
        {{ style.label }}
      </span>
    </div>
  </div>
  <!-- 属性信息 -->
  <div class="info">
    <div class="property-header">
      <span>要素属性</span>
    </div>
    <div
      v-for="(value, key) in selectedProperties"
      :key="key"
      class="property-row"
    >
      <span class="property-key">
        {{ key }}
      </span>
      <span class="property-value">
        {{ formatPropertyValue(value) }}
      </span>
    </div>
  </div>
</template>
<script setup>
import { onMounted, onUnmounted, reactive, ref } from "vue";
import * as Cesium from "cesium";
import "cesium/Build/Cesium/Widgets/widgets.css";
let viewer;
let handler;
//geojson
let landuseDataSource;
//记录每个Entity属于什么类型
const entityLandTypeMap = new Map();

//控制图层显示
const layerVisible = ref(true);
const landTypeVisibel = reactive({
  forest: true, //林地
  water: true, //水域
  building: true, //建筑
});

//地图样式和图例共用一套配置
const landTypeStyles = {
  forest: {
    label: "林地",
    cesiumColor: Cesium.Color.GREEN.withAlpha(0.5),
    cssColor: "#008000",
  },

  water: {
    label: "水域",
    cesiumColor: Cesium.Color.BLUE.withAlpha(0.5),
    cssColor: "#0000ff",
  },

  building: {
    label: "建筑",
    cesiumColor: Cesium.Color.RED.withAlpha(0.5),
    cssColor: "#ff0000",
  },
};
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

async function loadLanduse() {
  landuseDataSource = await Cesium.GeoJsonDataSource.load(landuse, {
    // clampToGround: true, //贴地,与outline冲突
  });
  viewer.dataSources.add(landuseDataSource);
  viewer.flyTo(landuseDataSource);

  //根据属性渲染样式
  styleGeoJsonEntities(landuseDataSource);
}

function styleGeoJsonEntities(dataSource) {
  //获取加载geojson的entities,记住要.values
  const entities = dataSource.entities.values;
  entities.forEach((entity) => {
    if (!entity.polygon) return;
    //取到属性
    const landType = entity.properties.landType.getValue();
    //根据属性渲染样式
    if (!landType) return;

    //样式
    const style = landTypeStyles[landType];
    if (!style) return;

    //设置颜色
    entity.polygon.material = style.cesiumColor;

    //记录entity属于什么类型
    entityLandTypeMap.set(entity, landType);

    entity.polygon.outline = true; //显示边框线，与clampToGround贴地冲突
    entity.polygon.outlineColor = Cesium.Color.WHITE;
    entity.polygon.outlineWidth = 10;
  });
}

//整个图层显隐
function toggleLayer() {
  if (!landuseDataSource) return;
  landuseDataSource.show = layerVisible.value;
}

//某个类型显隐
function toggleLandType(type) {
  if (!landuseDataSource) return;
  const entities = landuseDataSource.entities.values;
  entities.forEach((entity) => {
    //获取entity属于什么类型
    const landType = entityLandTypeMap.get(entity);
    if (landType !== type) {
      return;
    }
    //entity.show
    entity.show = landTypeVisibel[type];
  });
}

let pickedEntity;
let originalColor;//保存原来的样式
let originalOutlineColor;//保存原来的样式
let selectedProperties = ref(null);

function clearFeatureSelection() {
  // 恢复上一个Polygon原来的样式
  if (
    pickedEntity &&
    pickedEntity.polygon &&
    originalColor &&
    originalOutlineColor
  ) {
    pickedEntity.polygon.material = originalColor;
    pickedEntity.polygon.outlineColor = originalOutlineColor;
  }
  pickedEntity = null;
  originalColor = null;
  originalOutlineColor = null;
  selectedProperties.value = null;
}

//选中高亮显示
function highlightEntity(entity) {
  //高亮
  //选中唯一性,清除上一个选中
  clearFeatureSelection();
  pickedEntity = entity;
  //保存原样式
  originalColor = pickedEntity.polygon.material;
  originalOutlineColor = pickedEntity.polygon.outlineColor;
  //设置高亮样式
  pickedEntity.polygon.material = Cesium.Color.YELLOW.withAlpha(0.5);
  pickedEntity.polygon.outlineColor = Cesium.Color.YELLOW;
}

//显示属性信息
function showInfo(entity) {
  selectedProperties.value = entity.properties.getValue();
}

//属性格式化,因为 GeoJSON properties 不一定全是简单字符串
function formatPropertyValue(value) {
  //当属性为空时，返回 "-"
  if (value === null || value === undefined) {
    return "-";
  }
  //当属性为对象时，返回 JSON 字符串
  if (typeof value === "object") {
    return JSON.stringify(value);
  }
  return String(value);
}

function clickHandler(click) {
  const pick = viewer.scene.pick(click.position);
  if (!pick || !pick.id) {
    clearFeatureSelection();
    return;
  }
  const entity = pick.id;
  // 判断是不是当前GeoJSON图层
  if (!landuseDataSource || !landuseDataSource.entities.contains(entity)) {
    clearFeatureSelection();
    return;
  }

  // 目前只处理Polygon
  if (!entity.polygon) {
    return;
  }

  // 如果点击的还是同一个
  if (pickedEntity === entity) {
    return;
  }

  //高亮
  highlightEntity(entity);

  //获取属性信息
  showInfo(entity);
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
  //点击事件
  handler.setInputAction(clickHandler, Cesium.ScreenSpaceEventType.LEFT_CLICK);

  loadLanduse();
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
.layer-panel {
  position: absolute;
  top: 20px;
  left: 20px;
  z-index: 1000;
  width: 180px;
  padding: 15px;
  background: rgba(30, 30, 30, 0.85);
  color: white;
  border-radius: 6px;
  font-size: 14px;
}
.layer-title {
  font-weight: bold;
  margin-bottom: 15px;
}
.legend-title {
  margin-bottom: 10px;
  font-weight: bold;
}
.legend-item {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}

.legend-color {
  display: inline-block;
  width: 20px;
  height: 12px;
  border: 1px solid white;
}
.info {
  position: absolute;
  bottom: 2%;
  left: 20px;
  z-index: 1000;
  width: 180px;
  padding: 15px;
  background: rgba(30, 30, 30, 0.85);
  color: white;
  border-radius: 6px;
  font-size: 16px;
}
.property-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-bottom: 10px;
  font-weight: bold;
  border-bottom:1px solid rgba(255, 255, 255, 0.2);
}
.property-content {
  padding: 10px 15px;
  max-height: 430px;
  overflow-y: auto;
}
.property-row {
  display: flex;
  padding: 8px 0;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}
.property-key {
  width: 110px;
  flex-shrink: 0;
  color: #ccc;
}
.property-value {
  flex: 1;
  word-break: break-all;
}
</style>
