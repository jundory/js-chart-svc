<template>
  <div class="chart-container">
    <h3 v-if="chartTitle" class="chart-title">{{ chartTitle }}</h3>
    <div class="chart">
      <canvas ref="chartCanvas" :width="width" :height="height"></canvas>
    </div>
    <div class="legend" v-if="showLegend">
      <div v-for="(dataset, index) in data" :key="index" class="legend-item">
        <div
          class="color-box"
          :style="{ backgroundColor: colors[index % colors.length] }"
        ></div>
        <div class="legend-label">
          {{ dataset.label || `데이터셋 ${index + 1}` }}
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, watch } from "vue";

export default {
  name: "RadarChart",
  props: {
    data: {
      type: Array,
      required: true,
      // 예상 형태: [
      //   { label: '항목1', values: [80, 50, 30, 70, 60] },
      //   { label: '항목2', values: [60, 70, 80, 40, 50] }
      // ]
    },
    categories: {
      type: Array,
      required: true,
      // 예: ['카테고리1', '카테고리2', '카테고리3', '카테고리4', '카테고리5']
    },
    width: {
      type: Number,
      default: 500,
    },
    height: {
      type: Number,
      default: 500,
    },
    chartTitle: {
      type: String,
      default: "",
    },
    showLegend: {
      type: Boolean,
      default: true,
    },
    colors: {
      type: Array,
      default: () => ["#FF6384", "#36A2EB", "#FFCE56", "#4BC0C0", "#9966FF"],
    },
    maxValue: {
      type: Number,
      default: null, // null일 경우 데이터에서 최대값 찾음
    },
    stepSize: {
      type: Number,
      default: null, // null일 경우 자동 계산
    },
    steps: {
      type: Number,
      default: 5, // 그리드 동심원 개수
    },
  },
  setup(props) {
    const chartCanvas = ref(null);
    let ctx = null;

    const drawChart = () => {
      if (!chartCanvas.value) return;

      // 캔버스 컨텍스트 가져오기
      ctx = chartCanvas.value.getContext("2d");

      // 캔버스 초기화
      ctx.clearRect(0, 0, props.width, props.height);

      // 차트 크기 및 중심점 계산
      const chartSize = Math.min(props.width, props.height) - 40;
      const centerX = props.width / 2;
      const centerY = props.height / 2;
      const radius = chartSize / 2;

      // 최대값 계산 (props.maxValue가 없을 경우)
      let maxVal = props.maxValue;
      if (maxVal === null) {
        maxVal = 0;
        props.data.forEach((dataset) => {
          const datasetMax = Math.max(...dataset.values);
          maxVal = Math.max(maxVal, datasetMax);
        });
        // 최대값에 10% 더해서 여유 공간 확보
        maxVal = Math.ceil(maxVal * 1.1);
      }

      // 스텝 크기 계산
      const step = props.stepSize || Math.ceil(maxVal / props.steps);

      // 배경 그리드 그리기
      drawGrid(centerX, centerY, radius, maxVal, step);

      // 카테고리 라벨 그리기
      drawCategoryLabels(centerX, centerY, radius);

      // 데이터 다각형 그리기
      drawDataPolygons(centerX, centerY, radius, maxVal);
    };

    const drawGrid = (centerX, centerY, radius, maxVal, step) => {
      const categoryCount = props.categories.length;

      // 배경 동심원 그리기
      ctx.strokeStyle = "#DDDDDD";
      ctx.fillStyle = "#F5F5F5";

      for (let s = step; s <= maxVal; s += step) {
        // 현재 단계의 반지름 비율
        const stepRadius = (radius * s) / maxVal;

        // 동심원 그리기
        ctx.beginPath();
        for (let i = 0; i <= categoryCount; i++) {
          const angle = (Math.PI * 2 * i) / categoryCount - Math.PI / 2;
          const x = centerX + stepRadius * Math.cos(angle);
          const y = centerY + stepRadius * Math.sin(angle);

          if (i === 0) {
            ctx.moveTo(x, y);
          } else {
            ctx.lineTo(x, y);
          }
        }
        ctx.closePath();
        ctx.stroke();

        // 값 표시
        ctx.fillStyle = "#666666";
        ctx.font = "12px Arial";
        ctx.textAlign = "center";
        ctx.textBaseline = "middle";
        ctx.fillText(s.toString(), centerX, centerY - stepRadius);
      }

      // 축 그리기
      ctx.strokeStyle = "#AAAAAA";
      for (let i = 0; i < categoryCount; i++) {
        const angle = (Math.PI * 2 * i) / categoryCount - Math.PI / 2;
        ctx.beginPath();
        ctx.moveTo(centerX, centerY);
        ctx.lineTo(
          centerX + radius * Math.cos(angle),
          centerY + radius * Math.sin(angle)
        );
        ctx.stroke();
      }
    };

    const drawCategoryLabels = (centerX, centerY, radius) => {
      const categoryCount = props.categories.length;
      const labelDistance = radius * 1.1; // 라벨을 차트 바깥에 표시

      ctx.fillStyle = "#333333";
      ctx.font = "14px Arial";
      ctx.textAlign = "center";
      ctx.textBaseline = "middle";

      props.categories.forEach((category, index) => {
        const angle = (Math.PI * 2 * index) / categoryCount - Math.PI / 2;
        const x = centerX + labelDistance * Math.cos(angle);
        const y = centerY + labelDistance * Math.sin(angle);

        // 텍스트 위치 조정 (각도에 따라)
        if (angle === -Math.PI / 2) {
          // 상단
          ctx.textBaseline = "bottom";
        } else if (angle === Math.PI / 2) {
          // 하단
          ctx.textBaseline = "top";
        } else if (angle > -Math.PI / 2 && angle < Math.PI / 2) {
          // 오른쪽
          ctx.textAlign = "left";
        } else {
          // 왼쪽
          ctx.textAlign = "right";
        }

        ctx.fillText(category, x, y);
      });

      // 텍스트 정렬 재설정
      ctx.textAlign = "center";
      ctx.textBaseline = "middle";
    };

    const drawDataPolygons = (centerX, centerY, radius, maxVal) => {
      const categoryCount = props.categories.length;

      props.data.forEach((dataset, datasetIndex) => {
        const color = props.colors[datasetIndex % props.colors.length];
        const values = dataset.values;

        // 다각형 그리기
        ctx.beginPath();
        for (let i = 0; i < categoryCount; i++) {
          const angle = (Math.PI * 2 * i) / categoryCount - Math.PI / 2;
          const value = values[i];
          const distance = (radius * value) / maxVal;

          const x = centerX + distance * Math.cos(angle);
          const y = centerY + distance * Math.sin(angle);

          if (i === 0) {
            ctx.moveTo(x, y);
          } else {
            ctx.lineTo(x, y);
          }
        }
        ctx.closePath();

        // 스타일 설정 및 채우기
        ctx.fillStyle = `${color}33`; // 알파값 추가 (투명도)
        ctx.fill();
        ctx.strokeStyle = color;
        ctx.lineWidth = 2;
        ctx.stroke();

        // 데이터 포인트 그리기
        ctx.fillStyle = color;
        for (let i = 0; i < categoryCount; i++) {
          const angle = (Math.PI * 2 * i) / categoryCount - Math.PI / 2;
          const value = values[i];
          const distance = (radius * value) / maxVal;

          const x = centerX + distance * Math.cos(angle);
          const y = centerY + distance * Math.sin(angle);

          ctx.beginPath();
          ctx.arc(x, y, 4, 0, Math.PI * 2);
          ctx.fill();
        }
      });
    };

    // 초기 그리기 및 데이터 변경 감시
    onMounted(() => {
      drawChart();
    });

    watch(
      () => [...props.data],
      () => {
        drawChart();
      },
      { deep: true }
    );

    watch(
      () => [...props.categories],
      () => {
        drawChart();
      }
    );

    watch([() => props.width, () => props.height], () => {
      drawChart();
    });

    return {
      chartCanvas,
    };
  },
};
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
