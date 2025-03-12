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

const chartCanvas = ref(null);
let ctx = null;

// 축
const drawAxis = (chartPadding, chartWidth, chartHeight) => {
  ctx.beginPath(); // 기존 경로 초기화 후 새로 선 그리기
  //Y축
  ctx.moveTo(chartPadding, 10); //최상단 시작점
  ctx.lineTo(chartPadding, chartHeight); //(chartPadding, chartHeight)까지 선 그리기
  //X축
  ctx.lineTo(chartWidth - 10, chartHeight); //(chartPadding, chartHeight)에서 chartWidth-10까지 선 그리기 (-10은 오른쪽 여백)
  ctx.strokeStyle = "#333";
  ctx.lineWidth = 2;
  ctx.stroke(); //실제 선을 그림
};

// 눈금 & y축 레이블
const drawScale = (chartPadding, chartHeight, maxValue) => {
  const scaleSteps = 5; // 눈금 5개로 설정
  ctx.lineWidth = 0.5; // 눈금선 두께
  ctx.strokeStyle = "#999"; // 눈금선 색상
  ctx.fillStyle = "#333"; // 텍스트 색상
  ctx.font = "12px Arial"; // 글꼴
  ctx.textAlign = "right"; // 정렬

  for (let i = 0; i <= scaleSteps; i++) {
    // Y축 눈금 위치 계산 (***위쪽이 0, 아래쪽이 최대값)
    //  chartHeight = y축 0점, '-10'으로 라벨이 캔버스 위로 이탈 방지
    const y = chartHeight - (i * (chartHeight - 10)) / scaleSteps;

    //  눈금 값 계산
    const value = Math.round((i * maxValue) / scaleSteps); // 최대값을 눈금 수로 나눈 후 반올림

    // 눈금 선 그리기
    ctx.beginPath();
    ctx.moveTo(chartPadding, y); // (축 기준 눈금 시작점, 기울기)
    ctx.lineTo(chartPadding - 5, y); //  눈금 끝점 (왼쪽으로 5px 이동 후 그림, 기울기)
    ctx.stroke(); // 선 그리기

    // Y축 레이블 숫자 표시
    ctx.fillText(value.toString(), chartPadding - 10, y + 4); //text만 출력 가능, 눈금보다 10px 왼쪽, 글자 중앙 정렬(미세 조정)
  }
};

// 범례 & x축 레이블
const drawBars = (chartPadding, chartWidth, chartHeight, maxValue) => {
  const legendKeysArr = Object.keys(props.data[0]); // return ['legend_1', 'legend_2']
  // 막대 너비 및 간격 계산
  const groupWidth = (chartWidth - chartPadding - 20) / props.data.length;
  const barWidth = (groupWidth * 0.8) / legendKeysArr.length;
  const barSpacing = (groupWidth * 0.2) / (legendKeysArr.length + 1);
  // 그룹 별
  props.data.forEach((group, groupIdx) => {
    // 막대 별
    Object.keys(group).forEach((bar, barIdx) => {
      const value = group[bar] || 0; // 값이 없을 경우 0 처리
      const barHeight = (value / maxValue) * (chartHeight - 10);

      // 막대 그리기 시작할 x, y 좌표 계산
      const x =
        chartPadding +
        groupIdx * groupWidth +
        (barIdx + 1) * barSpacing +
        barIdx * barWidth;
      const y = chartHeight - barHeight;

      // 막대 그리기
      ctx.fillStyle = props.colors[barIdx % props.colors.length];
      ctx.fillRect(x, y, barWidth, barHeight);

      // 막대 위에 값 표시
      ctx.fillStyle = "#333"; // 글씨 색상
      ctx.font = "12px Arial";
      ctx.textAlign = "center";
      ctx.fillText(value.toString(), x + barWidth / 2, y - 5);

      // 직접 추가해본 테두리
      ctx.fillStyle = "#000";
      ctx.lineWidth = 2;
      ctx.strokeRect(x, y, barWidth, barHeight);
    });

    // 그룹 이름 (X축 레이블) 표시 (첫 번째 항목에서만)
    ctx.fillStyle = "#333";
    ctx.font = "12px Arial";
    ctx.textAlign = "center";
    ctx.fillText(
      `항목 ${groupIdx + 1}`,
      chartPadding + groupIdx * groupWidth + groupWidth / 2,
      chartHeight + 20
    );
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
  const chartPadding = 40;

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
  drawAxis(chartPadding, chartWidth, chartHeight);

  // Draw the scale : y축 라벨 숫자
  drawScale(chartPadding, chartHeight, maxValue);

  // Draw the bars  : 실제 그래프 그리기
  drawBars(chartPadding, chartWidth, chartHeight, maxValue);
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
