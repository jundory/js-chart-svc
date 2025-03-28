<template>
  <div class="app">
    <CanvasBarChart
      :data="barData"
      :width="800"
      :height="500"
      chartTitle="Canvas 막대 차트"
      :legends="barLegends"
      :maxBarsLength="3"
    />
    <!-- :colors="['#FF6384', '#36A2EB']" -->
    <!-- <WebGLBarChart
      :data="chartData"
      :width="800"
      :height="500"
      chartTitle="WebGL 2D 막대 차트"
      :legendLabels="['범례 1', '범례 2']"
      :colors="['#FF6384', '#36A2EB']"
    />
    <Bar3DChart
      :data="chartData"
      :width="800"
      :height="500"
      chartTitle="WebGL 3D 막대 차트"
      :legendLabels="['범례 1', '범례 2']"
      :colors="['#FF6384', '#36A2EB']"
    /> -->

    <!-- <PixiBarChart
      :data="chartData"
      :width="800"
      :height="500"
      chartTitle="PixiJS 막대 차트"
      :legendLabels="['범례 1', '범례 2']"
      :colors="['#FF6384', '#36A2EB']"
    /> -->

    <RadarChart
      :data="radarData"
      :categories="categories"
      :width="800"
      :height="500"
      chartTitle="Canvas 레이더 차트"
      :colors="['#FF6384', '#36A2EB']"
    />
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import CanvasBarChart from "./components/CanvasBarChart.vue";
import WebGLBarChart from "./components/WebGLBarChart.vue";
import Bar3DChart from "./components/Bar3DChart.vue";
// import PixiBarChart from "./components/PixiBarChart.vue";
import RadarChart from "./components/RadarChart.vue";

import BarData from "./mockData/BarData.json";

const barData = BarData.data;
const barLegends = ref();
const radarData = ref([
  { label: "항목1", values: [80, 50, 30, 70, 60] },
  { label: "항목2", values: [60, 70, 40, 20, 10] },
  { label: "항목3", values: [40, 75, 50, 30, 40] },
  { label: "항목4", values: [20, 60, 30, 60, 50] },
  { label: "항목5", values: [10, 25, 40, 10, 20] },
]);
const categories = ref(["속도", "안정성", "사용성", "디자인", "기능성"]);

onMounted(() => {
  barLegends.value = BarData.legends.map((legend) => legend.label);
  console.log("barLegends", barLegends.value);
});
</script>

<style scoped>
.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}

.app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
  max-width: 800px;
  margin-left: auto;
  margin-right: auto;
}
</style>
