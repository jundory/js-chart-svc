<template>
  <div ref="chartContainer">
    <h3>{{ chartTitle }}</h3>
    <div ref="legend">
      <ul>
        <li
          v-for="(label, index) in legendLabels"
          :key="index"
          :style="{ color: colors[index] }"
        >
          {{ label }}
        </li>
      </ul>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, defineProps } from "vue";
import * as THREE from "three";

const props = defineProps({
  data: {
    type: Array,
    required: true,
  },
  width: {
    type: Number,
    default: 800,
  },
  height: {
    type: Number,
    default: 500,
  },
  chartTitle: {
    type: String,
    default: "WebGL 3D 차트",
  },
  legendLabels: {
    type: Array,
    required: true,
  },
  colors: {
    type: Array,
    required: true,
  },
});
const chartContainer = ref(null);

// Three.js Scene, Camera, Renderer
const create3DChart = () => {
  const container = chartContainer.value;

  // Scene, Camera, Renderer 설정
  const scene = new THREE.Scene(); // 기본 3D 장면 위치 생성 <- WebGL은 직접 3D 객체 배치
  const camera = new THREE.PerspectiveCamera( // 카메라 세팅 (시야각, 종횡비, min거리, max거리)
    75,
    props.width / props.height,
    0.1,
    1000
  );
  console.log("props.height", props.height);
  const renderer = new THREE.WebGLRenderer(); //물체가 그려질 렌더링 객체 설정
  renderer.setSize(props.width, props.height); //렌더링 크기 설정 자동 처리
  container.appendChild(renderer.domElement);

  // Light 추가
  const light = new THREE.AmbientLight(0xffffff); // 빛 설정 <- 계산 필요x
  scene.add(light);

  // 막대 그래프를 위한 데이터
  const barWidth = 1; // 각 막대의 너비
  const gap = 1; // 막대 간의 간격
  const maxHeight = Math.max(
    ...props.data.map((item) => Math.max(...Object.values(item)))
  );

  // 3D 막대 생성
  props.data.forEach((item, index) => {
    const x =
      index * (barWidth + gap) - (props.data.length * (barWidth + gap)) / 2;

    Object.entries(item).forEach(([key, value], i) => {
      const color = props.colors[i] || "#000000";
      const geometry = new THREE.BoxGeometry( //3D 도형 자동 생성 (vertex 등 불필요)
        barWidth,
        (value / maxHeight) * 5,
        barWidth
      ); // Y축 크기 조정
      const material = new THREE.MeshLambertMaterial({ color }); // 3D 객체 표면 설정
      const bar = new THREE.Mesh(geometry, material);

      bar.position.set(x, value / 2, (i - 0.5) * barWidth); // Y축 위치, X축, Z축 위치 설정
      scene.add(bar);
    });
  });

  // 카메라 위치 조정
  camera.position.z = 10;
  // 카메라 위치 확인
  // if (
  //   camera.position.x === 0 &&
  //   camera.position.y === 0 &&
  //   camera.position.z > 0
  // ) {
  //   console.log("카메라가 정면을 보고 있음", camera.position, camera.lotation);
  // } else {
  //   console.log("카메라 위치를 조정해야 함");
  // }

  // 애니메이션 함수
  function animate() {
    requestAnimationFrame(animate);
    renderer.render(scene, camera);
  }
  animate();
};

onMounted(() => {
  create3DChart();
});
</script>

<style scoped>
#chartContainer {
  position: relative;
}
h3 {
  text-align: center;
}
#legend {
  margin-top: 10px;
  text-align: center;
}
ul {
  list-style: none;
  padding: 0;
}
li {
  display: inline-block;
  margin-right: 20px;
}
</style>
