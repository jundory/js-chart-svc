<template>
  <div class="app">
    <BarChart
      :data="chartData"
      :width="800"
      :height="500"
      chartTitle="바 테스트 차트"
      :legendLabels="['범례 1', '범례 2']"
      :colors="['#FF6384', '#36A2EB']"
    />
    <RadarChart
      :data="radarData"
      :categories="categories"
      :width="800"
      :height="800"
      chartTitle="레이더 더미 차트"
      :colors="['#FF6384', '#36A2EB']"
    />
  </div>
</template>

<script>
import { ref } from "vue";
import BarChart from "./components/BarChartTest.vue";
import RadarChart from "./components/RadarChart.vue";

export default {
  name: "App",
  components: {
    BarChart,
    RadarChart,
  },
  setup() {
    const chartData = ref([
      { legend_1: 10, legend_2: 90 },
      { legend_1: 20, legend_2: 80 },
      { legend_1: 30, legend_2: 70 },
      { legend_1: 40, legend_2: 60 },
      { legend_1: 50, legend_2: 50 },
    ]);
    const radarData = ref([
      { label: "항목1", values: [80, 50, 30, 70, 60] },
      { label: "항목2", values: [60, 70, 40, 20, 10] },
      { label: "항목3", values: [40, 75, 50, 30, 40] },
      { label: "항목4", values: [20, 60, 30, 60, 50] },
      { label: "항목5", values: [10, 25, 40, 10, 20] },
    ]);
    const categories = ref(["속도", "안정성", "사용성", "디자인", "기능성"]);

    return {
      chartData,
      categories,
      radarData,
    };
  },
};
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
