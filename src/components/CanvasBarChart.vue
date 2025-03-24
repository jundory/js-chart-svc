<template>
  <div class="chart-container">
    <h3 v-if="chartTitle" class="chart-title">{{ chartTitle }}</h3>
    <div class="chart">
      <canvas
        ref="chartCanvas"
        @mousemove="handleMouseMove"
        @mouseleave="hideTooltip"
        :width="width"
        :height="height"
      ></canvas>
    </div>
    <div class="legend" v-if="showLegend">
      <div v-for="(color, index) in colors" :key="index" class="legend-item">
        <div class="color-box" :style="{ backgroundColor: color }"></div>
        <div class="legend-label">{{ legendLabels[index] }}</div>
      </div>
    </div>
    <div
      v-if="tooltip.visible"
      class="tooltip"
      :style="{ top: tooltip.y + 110 + 'px', left: tooltip.x + 'px' }"
    >
      <div>{{ tooltip.label }}</div>
      <br />
      {{ tooltip.labelInfo }}:
      {{ tooltip.text }}
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

/**
 * @description 줌인아웃
 */
const scaleFactor = ref(1); // 줌 스케일 기본 크기 1 설정
const newScale = ref();
const minScale = 0.5;
const maxScale = 2;
const offsetX = ref(0); // 줌 기준점
const offsetY = ref(0);

const handleZoom = (event) => {
  event.preventDefault();

  const zoomIntensity = 0.1; // 줌 강도 조절
  const rect = chartCanvas.value.getBoundingClientRect(); //캔버스의 위치와 크기 정보를 화면 좌표 기준으로 저장
  const mouseX = event.clientX - rect.left; // rect를 빼서 캔버스 내부 좌표로 변환
  const mouseY = event.clientY - rect.top;

  newScale.value =
    scaleFactor.value - (event.deltaY > 0 ? zoomIntensity : -zoomIntensity); // 확대 & 축소

  scaleFactor.value = newScale.value;
  drawChart(); // 다시 그리기
};

/**
 * @description 툴팁 기능
 */
const hideTooltip = () => {
  tooltip.value.visible = false;
};
const tooltip = ref({
  visible: false,
  labelInfo: "",
  label: "",
  text: "",
  x: 0,
  y: 0,
});
const handleMouseMove = (event) => {
  if (!ctx.value) return;
  const rect = chartCanvas.value.getBoundingClientRect(); //캔버스의 위치와 크기 정보를 화면 좌표 기준으로 저장
  const mouseX = event.clientX - rect.left; // rect를 빼서 캔버스 내부 좌표로 변환
  const mouseY = event.clientY - rect.top;

  tooltip.value.visible = false; // 초기화

  chartDataWithRects.value.forEach((group) => {
    Object.keys(group).forEach((data) => {
      const { labelInfo, label, value, x, y, width, height } = group[data];
      if (
        mouseX >= x &&
        mouseX <= x + width &&
        mouseY >= y &&
        mouseY <= y + height
      ) {
        tooltip.value = {
          visible: true,
          labelInfo,
          label: label,
          text: `${value}`,
          x: event.clientX,
          y: event.clientY - 20,
        };
      }
    });
  });
};

/**
 * 차트의 x,y 축 드로잉
 * @param chartPadding 캔버스 여백
 * @param chartWidth 캔버스 너비
 * @param chartHeight 캔버스 높이
 */
const drawAxis = (chartPadding, chartWidth, chartHeight) => {
  ctx.value.beginPath(); // 기존 경로 초기화 후 새로 선 그리기
  //Y축
  ctx.value.moveTo(chartPadding, 10); //최상단 시작점
  ctx.value.lineTo(chartPadding, chartHeight); //(chartPadding, chartHeight)까지 선 그리기
  //X축
  ctx.value.lineTo(chartWidth - 10, chartHeight); //(chartPadding, chartHeight)에서 chartWidth-10까지 선 그리기 (-10은 오른쪽 여백)
  ctx.value.strokeStyle = "#333";
  ctx.value.lineWidth = 2;
  ctx.value.stroke(); //실제 선을 그림
};

/**
 * y축의 눈금 및 레이블 표시
 * @param chartPadding 캔버스 여백
 * @param chartHeight 캔버스 높이
 * @param maxValue 차트 데이터 최대값
 */
const drawScale = (chartPadding, chartHeight, maxValue) => {
  const scaleSteps = 5; // 눈금 5개로 설정
  ctx.value.lineWidth = 0.5; // 눈금선 두께
  ctx.value.strokeStyle = "#999"; // 눈금선 색상
  ctx.value.fillStyle = "#333"; // 텍스트 색상
  ctx.value.font = "12px Arial"; // 글꼴
  ctx.value.textAlign = "right"; // 정렬

  for (let i = 0; i <= scaleSteps; i++) {
    // Y축 눈금 위치 계산 (***위쪽이 0, 아래쪽이 최대값)
    //  chartHeight = y축 0점, '-10'으로 라벨이 캔버스 위로 이탈 방지
    const y = chartHeight - (i * (chartHeight - 10)) / scaleSteps;

    //  눈금 값 계산
    const value = Math.round((i * maxValue) / scaleSteps); // 최대값을 눈금 수로 나눈 후 반올림

    // 눈금 선 그리기
    ctx.value.beginPath();
    ctx.value.moveTo(chartPadding, y); // (축 기준 눈금 시작점, 기울기)
    ctx.value.lineTo(chartPadding - 5, y); //  눈금 끝점 (왼쪽으로 5px 이동 후 그림, 기울기)
    ctx.value.stroke(); // 선 그리기

    // Y축 레이블 숫자 표시
    ctx.value.fillText(value.toString(), chartPadding - 10, y + 4); //text만 출력 가능, 눈금보다 10px 왼쪽, 글자 중앙 정렬(미세 조정)
  }
};

/**
 * 범례 드로잉 및 x축 레이블 표시
 * @param chartPadding 캔버스 여백
 * @param chartWidth 캔버스 너비
 * @param chartHeight 캔버스 높이
 * @param maxValue 차트 데이터 최대값
 */
const chartDataWithRects = ref([]);
const drawBars = (chartPadding, chartWidth, chartHeight, maxValue) => {
  const legendKeysArr = Object.keys(props.data[0]); // return ['legend_1', 'legend_2']
  // 막대 너비 및 간격 계산
  const groupWidth = (chartWidth - chartPadding - 20) / props.data.length;
  const barWidth = (groupWidth * 0.8) / legendKeysArr.length;
  const barSpacing = (groupWidth * 0.2) / (legendKeysArr.length + 1);
  // 그룹 별
  props.data.forEach((group, groupIdx) => {
    const groupObj = {};
    // 막대 별
    Object.keys(group).forEach((bar, barIdx) => {
      const value = group[bar] || 0; // 값이 없을 경우 0 처리
      const barHeight = (value / maxValue) * (chartHeight - 10);

      // 막대 그리기 시작할 x, y 좌표 계산
      const x =
        chartPadding + //차트의 왼쪽 여백 (그래프 전체가 좌측에 붙지 않도록)
        groupIdx * groupWidth + //해당 그룹이 몇 번째 위치인지 계산 (0기준 그룹별 이동 거리)
        (barIdx + 1) * barSpacing + //막대 사이 간격 (첫 번째 막대 앞에도 간격 필요)
        barIdx * barWidth; //그룹 내 개별 막대가 차지하는 넓이 (막대 위치 이동)
      const y = chartHeight - barHeight - 1;

      // 막대 그리기
      ctx.value.fillStyle = props.colors[barIdx % props.colors.length];
      ctx.value.fillRect(x, y, barWidth, barHeight);

      // 막대 위에 값 표시
      ctx.value.fillStyle = "#333"; // 글씨 색상
      ctx.value.font = "12px Arial";
      ctx.value.textAlign = "center";
      ctx.value.fillText(value.toString(), x + barWidth / 2, y - 5);

      // 직접 추가해본 테두리
      // ctx.value.fillStyle = "#000";
      // ctx.value.lineWidth = 2;
      // ctx.value.strokeRect(x, y, barWidth, barHeight);
      //막대 별 rect 추가
      groupObj[bar] = {
        value,
        x,
        y,
        labelInfo: props.legendLabels[barIdx],
        label: `항목 ${groupIdx + 1}`,
        width: barWidth,
        height: barHeight,
      };
      chartDataWithRects.value.push(groupObj);
    });

    // 그룹 이름 (X축 레이블) 표시 (첫 번째 항목에서만)
    ctx.value.fillStyle = "#333";
    ctx.value.font = "12px Arial";
    ctx.value.textAlign = "center";
    ctx.value.fillText(
      `항목 ${groupIdx + 1}`,
      chartPadding + groupIdx * groupWidth + groupWidth / 2,
      chartHeight + 20
    );
  });
  ctx.value.restore(); // 변환 상태 복구
};

/**
 * @description 차트 초기 설정 및 드로잉 시작지점
 */

const chartCanvas = ref(null);
const ctx = ref(null);
const drawChart = () => {
  if (!chartCanvas.value) return;

  // canvas 2d context 생성
  ctx.value = chartCanvas.value.getContext("2d");
  // 초기화
  ctx.value.clearRect(0, 0, props.width, props.height);

  ctx.value.save(); //이전 상태 저장
  ctx.value.translate(offsetX.value, offsetY.value); // 줌 기준점 이동
  ctx.value.scale(scaleFactor.value, scaleFactor.value); // 줌 크기 적용

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
  chartCanvas.value.addEventListener("wheel", handleZoom);
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
.tooltip {
  position: absolute;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 5px 10px;
  border-radius: 5px;
  font-size: 14px;
  pointer-events: none; /* 마우스 이벤트 방해 안 받도록 */
  transform: translate(-50%, -100%);
  white-space: nowrap;
}
</style>
