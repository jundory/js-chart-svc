<template>
  <div class="chart-container">
    <h3 v-if="chartTitle" class="chart-title">{{ chartTitle }}</h3>
    <div class="chart">
      <canvas ref="glCanvas" :width="width" :height="height"></canvas>
      <canvas
        ref="overlayCanvas"
        :width="width"
        :height="height"
        class="overlay"
      ></canvas>
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

const props = defineProps({
  data: { type: Array, required: true },
  width: { type: Number, default: 500 },
  height: { type: Number, default: 300 },
  chartTitle: { type: String, default: "" },
  showLegend: { type: Boolean, default: true },
  legendLabels: { type: Array, default: () => ["legend_1", "legend_2"] },
  colors: { type: Array, default: () => ["#4CAF50", "#2196F3"] },
});

const glCanvas = ref(null);
const overlayCanvas = ref(null);
let gl;

const vertexShaderSource = `
    attribute vec2 a_position;
    uniform vec2 u_resolution;
    void main() {
      vec2 zeroToOne = a_position / u_resolution;
      vec2 zeroToTwo = zeroToOne * 2.0;
      vec2 clipSpace = zeroToTwo - 1.0;
      gl_Position = vec4(clipSpace * vec2(1, -1), 0, 1);
    }
  `;

const fragmentShaderSource = `
    precision mediump float;
    uniform vec4 u_color;
    void main() {
      gl_FragColor = u_color;
    }
  `;

const initWebGL = () => {
  if (!glCanvas.value) return;
  gl = glCanvas.value.getContext("webgl");
  if (!gl) {
    console.error("WebGL을 지원하지 않는 브라우저입니다.");
    return;
  }

  const vertexShader = createShader(gl.VERTEX_SHADER, vertexShaderSource);
  const fragmentShader = createShader(gl.FRAGMENT_SHADER, fragmentShaderSource);
  const program = createProgram(vertexShader, fragmentShader);
  gl.useProgram(program);

  return program;
};

const createShader = (type, source) => {
  const shader = gl.createShader(type);
  gl.shaderSource(shader, source);
  gl.compileShader(shader);
  if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
    console.error(`셰이더 컴파일 오류: ${gl.getShaderInfoLog(shader)}`);
    gl.deleteShader(shader);
    return null;
  }
  return shader;
};

const createProgram = (vertexShader, fragmentShader) => {
  const program = gl.createProgram();
  gl.attachShader(program, vertexShader);
  gl.attachShader(program, fragmentShader);
  gl.linkProgram(program);
  if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
    console.error(`프로그램 링크 오류: ${gl.getProgramInfoLog(program)}`);
    gl.deleteProgram(program);
    return null;
  }
  return program;
};

const drawBars = (program) => {
  const positionLocation = gl.getAttribLocation(program, "a_position");
  const resolutionLocation = gl.getUniformLocation(program, "u_resolution");
  const colorLocation = gl.getUniformLocation(program, "u_color");

  const positionBuffer = gl.createBuffer();
  gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);

  const chartWidth = props.width;
  const chartHeight = props.height - 40;
  const chartPadding = 40;
  const maxValue = Math.max(...props.data.flatMap(Object.values)) * 1.1;
  const groupWidth = (chartWidth - chartPadding - 20) / props.data.length;
  const barWidth = (groupWidth * 0.8) / props.legendLabels.length;
  const barSpacing = (groupWidth * 0.2) / (props.legendLabels.length + 1);

  const vertices = [];
  const colorData = [];

  props.data.forEach((group, groupIdx) => {
    Object.entries(group).forEach(([bar, value], barIdx) => {
      const barHeight = (value / maxValue) * (chartHeight - 10);
      const x =
        chartPadding +
        groupIdx * groupWidth +
        (barIdx + 1) * barSpacing +
        barIdx * barWidth;
      const y = chartHeight - barHeight;

      // 삼각형 2개로 사각형 생성
      vertices.push(x, chartHeight, x, y, x + barWidth, chartHeight);
      vertices.push(x + barWidth, chartHeight, x, y, x + barWidth, y);

      //   // 색상 데이터
      const [r, g, b] = hexToRgb(
        props.colors[(groupIdx + barIdx) % props.colors.length]
      );
      //   // 삼각형 2개 -> 꼭짓점 6개
      colorData.push(...Array(6).fill([r, g, b]));
      //   colorData.push(...[].concat(...Array(6).fill([r, g, b])));
      //   const colorIdx =
      //     (groupIdx * numBarsPerGroup + barIdx) % props.colors.length;
      //   const [r, g, b] = hexToRgb(props.colors[colorIdx]);
      //   colorData.push(...Array(6).fill([r, g, b]).flat());
    });
  });

  gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(vertices), gl.STATIC_DRAW);
  gl.viewport(0, 0, glCanvas.value.width, glCanvas.value.height);
  gl.uniform2f(resolutionLocation, glCanvas.value.width, glCanvas.value.height);
  gl.enableVertexAttribArray(positionLocation);
  gl.vertexAttribPointer(positionLocation, 2, gl.FLOAT, false, 0, 0);
  gl.clearColor(1, 1, 1, 1);
  gl.clear(gl.COLOR_BUFFER_BIT);

  //   const formattedColorData = colorData.reduce((acc, _, i, arr) => {
  //     //3개의 r,g,b 값을 하나의 배열로 묶어 저장
  //     if (i % 3 === 0) acc.push(arr.slice(i, i + 3));
  //     return acc;
  //   }, []);

  //   const formattedColorData = [];
  // for (let i = 0; i < colorData.length; i += 3) {
  //   formattedColorData.push([colorData[i], colorData[i + 1], colorData[i + 2]]);
  // }

  colorData.forEach(([r, g, b], i) => {
    gl.uniform4f(colorLocation, r, g, b, 1);
    gl.drawArrays(gl.TRIANGLES, i * 6, 6);
  });
};

const hexToRgb = (hex) => hex.match(/\w\w/g).map((c) => parseInt(c, 16) / 255);

const drawChart = () => {
  const program = initWebGL();
  if (program) drawBars(program);
};

onMounted(drawChart);
watch(() => props.data, drawChart, { deep: true });
</script>

<style scoped>
.chart-container {
  text-align: center;
}
.chart {
  position: relative;
}
.overlay {
  position: absolute;
  top: 0;
  left: 0;
  pointer-events: none;
}
.legend {
  display: flex;
  justify-content: center;
  margin-top: 10px;
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
