<template>
  <div class="chart-container">
    <h3 v-if="chartTitle" class="chart-title">{{ chartTitle }}</h3>
    <div class="chart">
      <canvas ref="chartCanvas" :width="width" :height="height"></canvas>
    </div>
    <div class="legend" v-if="showLegend">
      <div v-for="(color, index) in colors" :key="index" class="legend-item">
        <div class="color-box" :style="{ backgroundColor: color }"></div>
        <div class="legend-label">{{ legendLabels[index] }}</div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, defineProps } from "vue";

// props object로 타입 지정 및 기본값 설정
const props = defineProps({
  data: {
    type: Array,
    required: true,
  },
  width: {
    type: Number,
    default: 500,
  },
  height: {
    type: Number,
    default: 300,
  },
  chartTitle: {
    type: String,
    default: "",
  },
  showLegend: {
    type: Boolean,
    default: true,
  },
  legendLabels: {
    type: Array,
    default: () => ["legend_1", "legend_2"],
  },
  colors: {
    type: Array,
    default: () => ["#4CAF50", "#2196F3"],
  },
});
// const props = withDefaults(
//   defineProps(
//     {
//       data: Array,
//       width: Number,
//       height: Nymber,
//       chartTitle: String,
//       showLegend: Boolean,
//       legendLabels: Array,
//       clolors: Array,
//     },
//     {
//       width: 500,
//       height: 300,
//       chartTitle: "",
//       showLegend: true,
//       legendLabels: () => ["legend_1", "legend_2"],
//       colors: () => ["#4CAF50", "#2196F3"],
//     }
//   )
// );

const chartCanvas = ref(null);
let ctx = null;

const drawAxis = (padding, chartWidth, chartHeight) => {
  // Draw y-axis
  ctx.beginPath(); // 기존 경로 초기화 후 새로 선 그리기
  //Y축
  ctx.moveTo(padding, 10); //최상단 시작점
  ctx.lineTo(padding, chartHeight); //(padding, chartHeight)까지 선 그리기
  //X축
  ctx.lineTo(chartWidth - 10, chartHeight); //(padding, chartHeight)에서 chartWidth-10까지 선 그리기 (-10은 오른쪽 여백)
  ctx.strokeStyle = "#333";
  ctx.lineWidth = 2;
  ctx.stroke(); //실제 선을 그림
};

const drawScale = (padding, chartHeight, maxValue) => {
  // Draw horizontal lines for scale
  const scaleSteps = 5; // 눈금 5개로 설정
  ctx.lineWidth = 0.5; // 눈금선 두께
  ctx.strokeStyle = "#999"; // 눈금선 색상
  ctx.fillStyle = "#333"; // 텍스트 색상
  ctx.font = "12px Arial"; // 글꼴
  ctx.textAlign = "right"; // 정렬

  for (let i = 0; i <= scaleSteps; i++) {
    // Y축 눈금 위치 계산 (***위쪽이 0, 아래쪽이 최대값)
    //  chartHeight는 y축 0점, '-10'으로 라벨이 캔버스 이탈 방지, i=0일 때 y축 0점, i=5일 때 y축 최상단
    const y = chartHeight - (i * (chartHeight - 10)) / scaleSteps;

    //  눈금 값 계산
    const value = Math.round((i * maxValue) / scaleSteps); // 최대값을 눈금 수로 나눈 후 반올림

    // 눈금 선 그리기
    ctx.beginPath();
    ctx.moveTo(padding, y); // 눈금 시작점 (y축 기준선의 y위치에서 시작)
    ctx.lineTo(padding - 5, y); //  눈금 끝점 (왼쪽으로 5px 이동 후 눈금 그림)
    ctx.stroke(); // 선 그리기

    // Y축 숫자 표시
    ctx.fillText(value.toString(), padding - 10, y + 4); //text만 출력 가능, 눈금보다 10px 왼쪽, 글자 중앙 정렬(미세 조정)
  }
};

const drawBars = (padding, chartWidth, chartHeight, maxValue) => {
  const dataLength = props.data.length; //  데이터 갯수
  const legendKeys = Object.keys(props.data[0]).filter((key) =>
    key.startsWith("legend_")
  );
  const numLegends = legendKeys.length; // 1개 당 bar 갯수

  // 막대 너비 및 간격 계산
  const groupWidth = (chartWidth - padding - 20) / dataLength; // 전체 너비에서 여백 padding+20 제외, 한 그룹이 차지하는 너비
  const barWidth = (groupWidth * 0.8) / numLegends; // 각 막대의 너비 (그룹 내 막대 너비 80%)
  const barSpacing = (groupWidth * 0.2) / (numLegends + 1); // 막대 간의 간격 (전체 너비의 20%를 막대 간 간격), 막대 1개만큼 양쪽 여백 추가

  // Draw each data group
  props.data.forEach((item, dataIndex) => {
    legendKeys.forEach((key, legendIndex) => {
      const value = item[key];
      const barHeight = (value / maxValue) * (chartHeight - 10);

      // Calculate x position for this bar
      const x =
        padding +
        dataIndex * groupWidth +
        (legendIndex + 1) * barSpacing +
        legendIndex * barWidth;
      const y = chartHeight - barHeight;

      // Draw the bar
      ctx.fillStyle = props.colors[legendIndex % props.colors.length];
      ctx.fillRect(x, y, barWidth, barHeight);

      // Draw value on top of the bar
      ctx.fillStyle = "#333";
      ctx.font = "12px Arial";
      ctx.textAlign = "center";
      ctx.fillText(value.toString(), x + barWidth / 2, y - 5);

      // Draw x-axis label if it's the first legend
      if (legendIndex === 0) {
        ctx.fillStyle = "#333";
        ctx.font = "12px Arial";
        ctx.textAlign = "center";
        ctx.fillText(
          `항목 ${dataIndex + 1}`,
          x + groupWidth / 2,
          chartHeight + 20
        );
      }
    });
  });
};

const drawChart = () => {
  if (!chartCanvas.value) return;

  // canvas 2d context 생성
  ctx = chartCanvas.value.getContext("2d");

  // 초기화
  ctx.clearRect(0, 0, props.width, props.height);

  // 캔버스 크기 설정
  const chartWidth = props.width;
  const chartHeight = props.height - 40; // 라벨 표시 공간 확보
  const padding = 40;

  // 데이터 중 최댓값 찾기
  let maxValue = 0;
  props.data.forEach((item) => {
    Object.keys(item).forEach((key) => {
      if (key.startsWith("legend_") && item[key] > maxValue) {
        maxValue = item[key];
      }
    });
  });

  // 최댓값이 10의 배수가 되도록 올림
  maxValue = maxValue * 1.1;

  // Draw the axis : x,y 축 그리기
  drawAxis(padding, chartWidth, chartHeight);

  // Draw the scale : y축 라벨 숫자
  drawScale(padding, chartHeight, maxValue);

  // Draw the bars  : 실제 그래프 그리기
  drawBars(padding, chartWidth, chartHeight, maxValue);
};

onMounted(() => {
  drawChart();
});

watch(
  () => props.data,
  () => {
    drawChart();
  },
  { deep: true }
);

watch([() => props.width, () => props.height], () => {
  drawChart();
});
</script>

<style scoped>
.chart-container {
  font-family: Arial, sans-serif;
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 20px 0;
}

.chart-title {
  text-align: center;
  margin-bottom: 15px;
}

.chart {
  position: relative;
}

.legend {
  display: flex;
  justify-content: center;
  margin-top: 20px;
  flex-wrap: wrap;
}

.legend-item {
  display: flex;
  align-items: center;
  margin: 0 10px;
}

.color-box {
  width: 15px;
  height: 15px;
  margin-right: 5px;
}
</style>
