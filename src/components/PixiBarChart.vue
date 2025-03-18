<template>
  <div>
    <h3>{{ chartTitle }}</h3>
    <div ref="chartContainer"></div>
  </div>
</template>
<script setup>
import { onMounted, ref, defineProps, watch } from "vue";
import * as PIXI from "pixi.js";

const props = defineProps({
  data: { type: Array, required: true },
  width: { type: Number, default: 500 },
  height: { type: Number, default: 300 },
  chartTitle: { type: String, default: "" },
  showLegend: { type: Boolean, default: true },
  legendLabels: { type: Array, default: () => ["legend_1", "legend_2"] },
  colors: { type: Array, default: () => ["#4CAF50", "#2196F3"] },
});

function drawChart() {
  if (!app) return;

  app.stage.removeChildren();

  const barWidth = props.width / (props.data.length * 2);
  const maxHeight = Math.max(
    ...props.data.map((d) => Math.max(d.legend_1, d.legend_2))
  );

  props.data.forEach((item, groupIdx) => {
    ["legend_1", "legend_2"].forEach((key, barIdx) => {
      const value = item[key];
      const barHeight = (value / maxHeight) * props.height;
      const x = groupIdx * barWidth * 2 + barIdx * barWidth;
      const y = props.height - barHeight;

      const color = PIXI.utils.string2hex(props.colors[barIdx]);

      const bar = new PIXI.Graphics();
      bar.beginFill(color);
      bar.drawRect(x, y, barWidth - 5, barHeight);
      bar.endFill();

      app.stage.addChild(bar);
    });
  });
}

const chartContainer = ref(null);
let app;

onMounted(() => {
  app = new Application({
    width: props.width,
    height: props.height,
    backgroundColor: 0xffffff,
  });
  chartContainer.value.appendChild(app.view);
  drawChart();
});
</script>
<style>
.chart-container {
  width: 500px;
  height: 300px;
  border: 1px solid #ddd;
}
</style>
