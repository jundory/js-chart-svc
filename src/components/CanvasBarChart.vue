<template>
  <div class="chart-container">
    <h3 v-if="chartTitle" class="chart-title">{{ chartTitle }}</h3>
    <div class="chart">
      <canvas
        ref="chartCanvas"
        @mousemove="handleTooltipMove"
        @mouseleave="hideTooltip"
        :width="width"
        :height="height"
      ></canvas>
    </div>
    <div class="legend" v-if="props.legends">
      <div
        v-for="(label, index) in props.legends"
        :key="label"
        class="legend-item"
      >
        <div
          class="color-box"
          :style="{ backgroundColor: props.colors[index] }"
        ></div>
        <div class="legend-label">{{ legends[index] }}</div>
      </div>
    </div>
    {{ tooltip }}

    <div
      v-if="tooltip.visible"
      class="tooltip"
      :style="{ top: tooltip.y + 'px', left: tooltip.x + 'px' }"
    >
      <div class="tooltip__label">{{ tooltip.label }}</div>
      <br />
      <div class="tooltip__detail">
        <div
          class="color-box"
          :style="{ backgroundColor: props.colors[tooltip.index] }"
        ></div>
        {{ tooltip.labelInfo }} :
        {{ tooltip.text }}
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, watch, computed, defineProps } from "vue";

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
  },
  legends: {
    type: Array,
  },
  maxBarsLength: {
    type: Number,
    default: 3,
  },
  colors: {
    type: Array,
    default: () => [
      "#4E79A7", // 블루
      "#F28E2B", // 오렌지
      "#E15759", // 레드
      "#76B7B2", // 청록색
      "#59A14F", // 녹색
      "#EDC949", // 노란색
      "#B07AA1", // 보라색
    ],
  },
  fontOptions: {
    type: Object,
    default: () => ({
      fontFamily: "Arial",
      fontSize: 12,
      fontColor: "#000",
      fontContrastColor: "#fff",
      fontWeight: "normal",
    }),
  },
});

// 차트 기본 좌표
const offset = reactive({
  x: 0,
  y: 0,
});
// 차트의 크기와 위치 정보 화면 좌표 기준으로 계산
const chartRect = computed(() => {
  return chartCanvas.value.getBoundingClientRect();
});

/**
 * @description 드래그
 */
const isDragging = ref(false);
const lastMouse = reactive({
  x: 0,
  y: 0,
});

const handleMouseUp = () => {
  isDragging.value = false;
};
const handleMouseClick = (event) => {
  isDragging.value = true;
  lastMouse.x = event.clientX;
  lastMouse.y = event.clientY;
};
const handleMouseMove = (event) => {
  if (scaleFactor.value <= 1 || !isDragging.value) return;
  const xGap = event.clientX - lastMouse.x;
  const yGap = event.clientY - lastMouse.y;

  const chartWidth = chartRect.value.width; // 차트의 실제 너비
  const chartHeight = chartRect.value.height; // 차트의 실제 높이

  // 🔥 중심 보정을 위해 이동 가능한 범위 계산
  const scaledWidth = chartWidth * scaleFactor.value;
  const scaledHeight = chartHeight * scaleFactor.value;

  // 이동 가능한 최소 & 최대 값 계산
  const minOffsetX = Math.min(0, chartWidth - scaledWidth); // 왼쪽 경계
  const maxOffsetX = Math.max(0, scaledWidth - chartWidth); // 오른쪽 경계
  const minOffsetY = Math.min(0, chartHeight - scaledHeight); // 상단 경계
  const maxOffsetY = Math.max(0, scaledHeight - chartHeight); // 하단 경계

  // 드래그 후 새로운 offset을 계산하면서 이동 범위 제한
  offset.x = Math.max(minOffsetX, Math.min(maxOffsetX, offset.x + xGap));
  offset.y = Math.max(minOffsetY, Math.min(maxOffsetY, offset.y + yGap));
  // console.log(" 드래그 적용 new 좌표", offset.x, offset.y);

  lastMouse.x = event.clientX;
  lastMouse.y = event.clientY;
  drawChart();
};

/**
 * @description 줌 인앤아웃
 */
const scaleFactor = ref(1); // 줌 크기(배율) 기본 1 설정
const newScale = ref(null); // 새롭게 적용될 줌 크기

const handleZoom = (event) => {
  event.preventDefault();

  const minScale = 1;
  const maxScale = 2;
  const zoomIntensity = 0.2; // 줌 강도 조절
  const mouseX = event.clientX - chartRect.value.left; // 마우스 절대 위치에서 rect를 빼서 캔버스 내부 좌표로 변환
  const mouseY = event.clientY - chartRect.value.top;

  const prevScale = scaleFactor.value;
  newScale.value =
    scaleFactor.value - (event.deltaY > 0 ? zoomIntensity : -zoomIntensity); // 휠 방향 확대 & 축소

  // 줌 한계 설정
  if (newScale.value < minScale) newScale.value = minScale;
  if (newScale.value > maxScale) newScale.value = maxScale;

  if (event.deltaY < 0) {
    // 줌인 시 중심을 마우스 위치로 설정 ***기본값은 (0,0))
    offset.x =
      mouseX - (mouseX - offset.x) * (newScale.value / scaleFactor.value);
    // (현재 마우스 위치와 현재 기준점 사이 거리) * (줌 후 새로운 비율 / 줌 이전 기존 배율)
    offset.y =
      mouseY - (mouseY - offset.y) * (newScale.value / scaleFactor.value);
  } else {
    // 줌 아웃 시 차트 절대 중심 기준
    const defaultOffset = { x: 0, y: 0 };
    const chartCenterX = defaultOffset.x + chartRect.value.width / 2; // 최초 차트 중심 X
    const chartCenterY = defaultOffset.y + chartRect.value.height / 2; // 최초 차트 중심 Y

    offset.x =
      chartCenterX -
      (chartCenterX - defaultOffset.x) * (newScale.value / prevScale);
    offset.y =
      chartCenterY -
      (chartCenterY - defaultOffset.y) * (newScale.value / prevScale);
  }
  // console.log(" 줌 적용 new 좌표", offset.x, offset.y);

  scaleFactor.value = newScale.value;
  drawChart(); // 다시 그리기
};

/**
 * @description 툴팁 기능
 */
const hideTooltip = () => {
  tooltip.visible = false;
};
const tooltip = reactive({
  visible: false,
  index: null,
  labelInfo: "",
  label: "",
  text: "",
  x: 0,
  y: 0,
});
const handleTooltipMove = (event) => {
  if (!ctx.value) return;
  /**
   * 현재 마우스 커서 좌표값 구하는 공식
   * 1. 현 커서 위치에서 rect를 빼서, 캔버스 내부 좌표로 변환
   * 2. 드래그로 이동되어 바뀐(xGap, yGap이 더해진) offset값을 빼서, 차트의 새로운 위치 재계산
   * 3. scaleFactor(확대 비율)로 나눠서, 줌인&아웃으로 바뀐 차트의 비율에 맞게 마우스 위치도 변환
   */
  const mouseX =
    (event.clientX - chartRect.value.left - offset.x) / scaleFactor.value;
  const mouseY =
    (event.clientY - chartRect.value.top - offset.y) / scaleFactor.value;

  tooltip.visible = false; // 초기화
  chartDataArr.value.forEach((group) => {
    Object.keys(group).forEach((data, index) => {
      const { labelInfo, label, value, x, y, width, height } = group[data];
      if (
        mouseX >= x &&
        mouseX <= x + width &&
        mouseY >= y &&
        mouseY <= y + height
      ) {
        tooltip.index = index; //컬러박스용
        tooltip.visible = true;
        tooltip.labelInfo = labelInfo;
        tooltip.label = label;
        tooltip.text = `${value}`;
        tooltip.x = event.clientX;
        tooltip.y = event.clientY - 20;
      }
    });
  });
};

const drawBgLines = (chartPadding, chartWidth, chartHeight, maxValue) => {
  const { tickSize, stepCount } = calculateScale(maxValue, 5);

  ctx.value.strokeStyle = "#ddd"; // 선 색상 (연한 회색)
  ctx.value.lineWidth = 1; // 선 두께
  // 가로선 (y축 기준)
  for (let i = 1; i <= stepCount; i++) {
    // const y = chartPadding + chartHeight * (1 - i / 10);
    const y = chartHeight - (i * (chartHeight - 10)) / stepCount;

    ctx.value.beginPath();
    ctx.value.moveTo(chartPadding, y); //여기부터
    ctx.value.lineTo(chartWidth - 10, y); //여기까지
    ctx.value.stroke();
  }

  // 세로선 (x축 기준)
  const groupWidth = (chartWidth - chartPadding - 20) / props.data.length; // 막대 그룹 간격
  for (let i = 1; i <= props.data.length; i++) {
    const x = chartPadding + i * groupWidth;
    ctx.value.beginPath();
    ctx.value.moveTo(x, 10);
    ctx.value.lineTo(x, chartHeight);
    ctx.value.stroke();
  }
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
  const { tickSize, stepCount } = calculateScale(maxValue, 5);
  const { fontFamily, fontSize, fontColor, fontWeight } = props.fontOptions;

  ctx.value.lineWidth = 0.5; // 눈금선 두께
  ctx.value.strokeStyle = "#999"; // 눈금선 색상
  ctx.value.fillStyle = fontColor; // 텍스트 색상
  ctx.value.font = `${fontWeight} ${fontSize}px ${fontFamily}`; // 글꼴
  ctx.value.textAlign = "right"; // 정렬

  for (let i = 0; i <= stepCount; i++) {
    // Y축 눈금 위치 계산 (***위쪽이 0, 아래쪽이 최대값)
    //  chartHeight = y축 0점, '-10'으로 라벨이 캔버스 위로 이탈 방지
    const y = chartHeight - (i * (chartHeight - 10)) / stepCount;
    //  눈금 값 계산
    const value = i * tickSize;

    // 눈금 선 그리기
    ctx.value.beginPath();
    ctx.value.moveTo(chartPadding, y); // (축 기준 눈금 시작점, 기울기)
    // ctx.value.lineTo(chartPadding - 5, y); //  눈금 끝점 (왼쪽으로 5px 이동 후 그림, 기울기)
    ctx.value.stroke(); // 선 그리기

    // Y축 레이블 숫자 표시
    ctx.value.fillText(value.toString(), chartPadding - 5, y + 4); //text만 출력 가능, 눈금보다 10px 왼쪽, 글자 중앙 정렬(미세 조정)
  }
};

/**
 * 범례 드로잉 및 x축 레이블 표시
 * @param chartPadding 캔버스 여백
 * @param chartWidth 캔버스 너비
 * @param chartHeight 캔버스 높이
 * @param maxValue 차트 데이터 최대값
 */
const chartDataArr = ref([]);
const drawBars = (chartPadding, chartWidth, chartHeight, maxValue) => {
  const { fontFamily, fontSize, fontColor, fontContrastColor, fontWeight } =
    props.fontOptions;
  const legendKeysArr = Object.keys(props.data[0].values); // return ['sales', 'profit', 'expenses', 'growth_rate']
  // 막대 너비 및 간격 계산
  const groupWidth = (chartWidth - chartPadding - 20) / props.data.length;
  const barWidth = (groupWidth * 0.8) / legendKeysArr.length;
  const barSpacing = (groupWidth * 0.2) / (legendKeysArr.length + 1);
  // 그룹 별
  props.data.forEach((group, groupIdx) => {
    const groupObj = {};
    // 막대 별
    Object.keys(group.values).forEach((bar, barIdx) => {
      const value = group.values[bar] || 0; // 값이 없을 경우 0 처리
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
      const isTopBar = barHeight > chartHeight - 20;
      const labelY = isTopBar ? y + 15 : Math.max(y - 5, 15); // 최소 y 좌표를 15로 설정
      isTopBar
        ? (ctx.value.fillStyle = fontContrastColor)
        : (ctx.value.fillStyle = fontColor); // 글씨 색상
      ctx.value.font = `${fontWeight} ${fontSize}px ${fontFamily}`;
      ctx.value.textAlign = "center";
      ctx.value.fillText(value.toString(), x + barWidth / 2, labelY);

      //막대 별 rect 추가
      groupObj[bar] = {
        value,
        x,
        y,
        labelInfo: props.legends ? props.legends[barIdx] : null,
        label: props.data[groupIdx].category,
        width: barWidth,
        height: barHeight,
      };
      chartDataArr.value.push(groupObj);
    });

    // 그룹 이름 (X축 레이블) 표시 (첫 번째 항목에서만)
    ctx.value.fillStyle = "#333";
    ctx.value.font = "12px Arial";
    ctx.value.textAlign = "center";
    ctx.value.fillText(
      props.data[groupIdx].category,
      chartPadding + groupIdx * groupWidth + groupWidth / 2,
      chartHeight + 20
    );
  });
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

  // 줌인아웃
  ctx.value.save(); //이전(초기) 상태 저장
  ctx.value.translate(offset.x, offset.y); // 줌 기준점 이동
  ctx.value.scale(scaleFactor.value, scaleFactor.value); // 줌 크기 적용

  // 캔버스 크기 설정
  const chartWidth = props.width;
  const chartHeight = props.height - 40; // 라벨 표시 공간 확보
  const chartPadding = 40;

  // 데이터 중 최댓값 찾기
  const maxValue = calculateMaxValue(
    Math.max(
      ...props.data.map((group) => Math.max(...Object.values(group.values)))
    )
  ); //막대가 차트 높이를 벗어나지 않게 (barHeight 비율은 maxValue와 반비례)

  // Draw the axis : x,y 축 그리기
  drawAxis(chartPadding, chartWidth, chartHeight);
  // Draw the scale : y축 라벨 숫자
  drawScale(chartPadding, chartHeight, maxValue);
  //
  drawBgLines(chartPadding, chartWidth, chartHeight, maxValue);
  // Draw the bars  : 실제 그래프 그리기
  drawBars(chartPadding, chartWidth, chartHeight, maxValue);
  ctx.value.restore(); // save()로 저장한 이전(초기) 상태로 복구
};

onMounted(() => {
  drawChart();
  //wheel in&out
  chartCanvas.value.addEventListener("wheel", handleZoom);
  //mouse drag
  chartCanvas.value.addEventListener("mousedown", handleMouseClick);
  chartCanvas.value.addEventListener("mousemove", handleMouseMove);
  chartCanvas.value.addEventListener("mouseup", handleMouseUp);
  chartCanvas.value.addEventListener("mouseleave", handleMouseUp);
});

watch(
  () => [props.data, props.legends],
  () => {
    drawChart();
  },
  { deep: true }
);

// 눈금 최대값 계산
const calculateMaxValue = (maxValue) => {
  if (maxValue < 50) return 50; // 최소값 50 보장

  // 50 단위로 반올림하되, maxValue를 초과하지 않도록 조정
  const roundedMax = Math.ceil(maxValue / 50) * 50;

  // 만약 반올림된 값이 maxValue보다 50 이상 크다면, maxValue에 가장 가까운 50 단위로 설정
  return roundedMax - maxValue >= 50 ? roundedMax - 50 : roundedMax;
};

// 눈금 개수&간격 계산
const calculateScale = (maxValue, scaleStep) => {
  // 최댓값을 scaleSteps 개수로 나누고, 가장 가까운 10의 배수로 반올림
  const rawTickSize = maxValue / scaleStep;
  const magnitude = 10 ** Math.floor(Math.log10(rawTickSize)); // 10의 승수 구하기
  // 데이터의 최대값(maxValue)이 커지면 자연스럽게 간격(tickSize)도 커짐
  let tickSize = Math.ceil(rawTickSize / magnitude) * magnitude; // 10^n 단위로 반올림 (애매한 숫자x)
  const stepCount = Math.max(2, Math.ceil(maxValue / tickSize)); // 최소 2개 이상의 눈금 보장
  return { tickSize, stepCount };
};
// 최대값에 맞춰 눈금 개수 계산
// const calculateScale = (maxValue) => {
//   const tickSize = calculateTickSize(maxValue);
//   const stepCount = Math.ceil(maxValue / tickSize); // 총 눈금 개수 계산
//   return {
//     tickSize,
//     stepCount,
//   };
// };
// const calculateTickSize = (maxValue) => {
//   const numDigits = Math.floor(Math.log10(maxValue)); // 자릿수 계산
//   let tickSize = numDigits === 1 ? 20 : numDigits === 2 ? 200 : 2000; // 간격 계산
//   if (maxValue < tickSize) {
//     tickSize /= 2; // 최대값 < 간격 일 시, 절반으로 간격 조정
//   }
//   return tickSize;
// };
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
  /* background: rgba(0, 0, 0, 0.7); */
  /* color: white; */
  color: rgba(0, 0, 0, 0.7);
  background: #fff;
  padding: 5px 10px;
  border: thick double #32a1ce;
  border-radius: 3px;
  font-size: 14px;
  pointer-events: none; /* 마우스 이벤트 방해 안 받도록 */
  transform: translate(-50%, -100%);
  white-space: nowrap;
}

.tooltip__label {
  font-weight: 600;
}

.tooltip__detail {
  display: flex;
  align-items: center;
}
</style>
