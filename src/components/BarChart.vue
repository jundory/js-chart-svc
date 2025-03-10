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

<script>
import { ref, onMounted, watch } from "vue";

export default {
  name: "BarChart",
  props: {
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
  },
  setup(props) {
    const chartCanvas = ref(null);
    let ctx = null;
    const test = ref(null);

    const drawChart = () => {
      if (!chartCanvas.value) return;

      // Get the canvas context
      ctx = chartCanvas.value.getContext("2d");

      // Clear the canvas
      ctx.clearRect(0, 0, props.width, props.height);

      // Set chart dimensions
      const chartWidth = props.width;
      const chartHeight = props.height - 40; // Leave room for labels
      const padding = 40;

      // Calculate the max value for scaling
      let maxValue = 0;
      props.data.forEach((item) => {
        Object.keys(item).forEach((key) => {
          if (key.startsWith("legend_") && item[key] > maxValue) {
            maxValue = item[key];
          }
        });
      });

      // Add 10% to max value for better visualization
      maxValue = maxValue * 1.1;

      // Draw the axis : x,y 축 그리기
      drawAxis(padding, chartWidth, chartHeight);

      // Draw the scale : y축 라벨 숫자
      drawScale(padding, chartHeight, maxValue);

      // Draw the bars  : 실제 그래프 그리기
      drawBars(padding, chartWidth, chartHeight, maxValue);
    };

    const drawAxis = (padding, chartWidth, chartHeight) => {
      // Draw y-axis
      ctx.beginPath();
      ctx.moveTo(padding, 10);
      ctx.lineTo(padding, chartHeight);
      ctx.lineTo(chartWidth - 10, chartHeight);
      ctx.strokeStyle = "#333";
      ctx.lineWidth = 2;
      ctx.stroke();
    };

    const drawScale = (padding, chartHeight, maxValue) => {
      // Draw horizontal lines for scale
      const scaleSteps = 5;
      ctx.lineWidth = 0.5;
      ctx.strokeStyle = "#999";
      ctx.fillStyle = "#333";
      ctx.font = "12px Arial";
      ctx.textAlign = "right";

      for (let i = 0; i <= scaleSteps; i++) {
        const y = chartHeight - (i * (chartHeight - 10)) / scaleSteps;
        const value = Math.round((i * maxValue) / scaleSteps);

        // Draw scale line
        ctx.beginPath();
        ctx.moveTo(padding, y);
        ctx.lineTo(padding - 5, y);
        ctx.stroke();

        // Draw scale text
        ctx.fillText(value.toString(), padding - 10, y + 4);
      }
    };

    const drawBars = (padding, chartWidth, chartHeight, maxValue) => {
      const dataLength = props.data.length;
      const legendKeys = Object.keys(props.data[0]).filter((key) =>
        key.startsWith("legend_")
      );
      const numLegends = legendKeys.length;

      // Calculate the width for each data group
      const groupWidth = (chartWidth - padding - 20) / dataLength;
      const barWidth = (groupWidth * 0.8) / numLegends;
      const barSpacing = (groupWidth * 0.2) / (numLegends + 1);

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

    // Initial draw and watch for data changes
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
