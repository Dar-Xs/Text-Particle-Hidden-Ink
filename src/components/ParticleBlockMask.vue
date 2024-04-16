<script setup lang="ts">
import { onMounted, ref } from "vue";

// 配置
const maskColor = "#387ef7";
// const textColor = "#FFFFFF";
const diffusionRadius = 250; //px
const maxSpeed = 30; //px/s
const ttl = 500; //ms
const radius = 1.4; //px
const scale = 2;

const randInt = (lowerBound: number, upperBound: number) => {
  return lowerBound + Math.floor(Math.random() * (upperBound - lowerBound));
};
const randSpeed = (): vector2 => {
  const speedVal = randInt(0, maxSpeed);
  const angle = Math.random() * 2 * Math.PI;
  const speed: vector2 = [
    speedVal * Math.sin(angle),
    speedVal * Math.cos(angle),
  ];

  return speed;
};
const distance = (a: vector2, b: vector2): number => {
  const dx = a[0] - b[0];
  const dy = a[1] - b[1];
  return Math.sqrt(dx * dx + dy * dy);
};
const delta = (a: vector2, b: vector2): vector2 => {
  const dx = a[0] - b[0];
  const dy = a[1] - b[1];
  return [dx, dy];
};

let particles: ParticleData[] = [];
const initParticles = () => {
  particles = [];
  const density = 0.012;
  const padding = 20; //px
  //ParticleData需要传入，initParticles用于演示
  if (!canvasRef.value) {
    console.error("calling initParticles() with `canvasRef.value` undefine");
    return;
  }
  lastUpdateTime.value = Date.now();
  const width = canvasRef.value.width - padding * 2;
  const height = canvasRef.value.height - padding * 2;
  const count = Math.floor(width * height * density);
  for (let i = 0; i < count; i++) {
    const startPosition: vector2 = [
      randInt(padding, width + padding),
      randInt(padding, height + padding),
    ];
    const speed: vector2 = randSpeed();
    const tl = randInt(0, ttl);
    const position: vector2 = [
      startPosition[0] + (speed[0] * tl) / 1000,
      startPosition[1] + (speed[1] * tl) / 1000,
    ];
    const newParticle: ParticleData = {
      startPosition,
      position,
      speed,
      ttl: ttl - tl,
    };
    particles.push(newParticle);
  }
};

const canvasRef = ref<HTMLCanvasElement>();
const ctx = ref<CanvasRenderingContext2D>();

type vector2 = [number, number];
type ParticleData = {
  startPosition: vector2;
  position: vector2;
  speed: vector2;
  ttl: number;
};
const lastUpdateTime = ref(Date.now());

let mouseInside = false;
const mousePosition: vector2 = [0, 0];
const straceTTL = 5000; //ms
type MouseStrace = {
  position: vector2;
  ttl: number;
};
let strace: MouseStrace[] = [];
const updateStrace = (dt: number) => {
  strace = strace
    .filter((s) => s.ttl > dt)
    .map((s) => ({ ...s, ttl: s.ttl - dt }));
};
const mouseMove = (payload: MouseEvent) => {
  mouseInside = true;
  const { offsetX, offsetY } = payload;
  const current: vector2 = [offsetX * scale, offsetY * scale];
  if (mousePosition[0] == current[0] && mousePosition[1] == current[1]) {
    return;
  }
  mousePosition[0] = current[0];
  mousePosition[1] = current[1];
  strace.push({
    position: current,
    ttl: straceTTL,
  });
};
const mouseLeave = () => {
  mouseInside = false;
};

const dis2acc = (dis: number): number => {
  return Math.sqrt(diffusionRadius - dis) * 100;
};
const updateParticles = (dt: number) => {
  particles.forEach((particle) => {
    if (mouseInside) {
      const dis = distance(particle.position, mousePosition);
      if (dis < diffusionRadius) {
        const dr = delta(particle.position, mousePosition);
        particle.speed[0] += ((dr[0] / dis) * dis2acc(dis) * dt) / 1000;
        particle.speed[1] += ((dr[1] / dis) * dis2acc(dis) * dt) / 1000;
      }
    }
    if (particle.ttl <= dt) {
      particle.position[0] = particle.startPosition[0];
      particle.position[1] = particle.startPosition[1];
      particle.speed = randSpeed();
      particle.ttl = (particle.ttl + ttl - (dt % ttl)) % ttl;
      return;
    }
    particle.position[0] += (particle.speed[0] * dt) / 1000;
    particle.position[1] += (particle.speed[1] * dt) / 1000;
    particle.ttl -= dt;
  });
};

const drawParticles = (ctx: CanvasRenderingContext2D) => {
  ctx.globalCompositeOperation = "source-over";
  ctx.fillStyle = maskColor;
  ctx.fillRect(0, 0, canvasRef.value!.width, canvasRef.value!.height);
  particles.forEach((particle) => {
    ctx.fillStyle = `rgba(255,255,255,${
      (Math.min(particle.ttl, ttl - particle.ttl) * 2) / ttl
    })`;
    ctx.beginPath();
    ctx.arc(
      particle.position[0],
      particle.position[1],
      radius,
      0,
      Math.PI * 2,
      true
    );
    ctx.fill();
  });

  const drawStrace = (position: vector2, percent = 1) => {
    const gradient = ctx.createRadialGradient(
      position[0],
      position[1],
      radius * 25 * percent,
      position[0],
      position[1],
      radius * 60
    );
    gradient.addColorStop(0, `rgba(0, 0, 0, ${percent})`);
    gradient.addColorStop(0.5, `rgba(0, 0, 0, ${0.866 * percent})`);
    gradient.addColorStop(1, "rgba(0, 0, 0, 0)");
    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, canvasRef.value!.width, canvasRef.value!.height);
  };
  ctx.globalCompositeOperation = "destination-out";
  strace.forEach((s) => {
    drawStrace(s.position, Math.min(s.ttl / (straceTTL - 3000), 1));
  });
  if (mouseInside) {
    drawStrace(mousePosition);
  }
};
const refresh = () => {
  if (!canvasRef.value || !ctx.value) {
    console.error("calling refresh() with `canvasRef.value` undefine");
    return;
  }

  const now = Date.now();
  const dt = now - lastUpdateTime.value;
  lastUpdateTime.value = now;
  updateParticles(dt);
  updateStrace(dt);
  ctx.value.clearRect(0, 0, canvasRef.value.width, canvasRef.value.height);
  drawParticles(ctx.value);
  requestAnimationFrame(refresh);
};

onMounted(() => {
  if (canvasRef.value) {
    ctx.value = canvasRef.value.getContext("2d") || undefined;
    canvasRef.value.width = canvasRef.value.offsetWidth * scale;
    canvasRef.value.height = canvasRef.value.offsetHeight * scale;
  }
  if (particles.length === 0) initParticles();
  if (adjustHandleRef.value) {
    document.addEventListener("mousedown", (e) => {
      if (e.target !== adjustHandleRef.value) return;
      document.addEventListener("mousemove", resize);
      const removeEventListener = () => {
        document.removeEventListener("mousemove", resize);
        document.removeEventListener("mouseup", removeEventListener);
      };
      document.addEventListener("mouseup", removeEventListener);
    });
  }
  refresh();
});

const style = ref(400);
const adjustHandleRef = ref<HTMLDivElement>();
const resize = (event: MouseEvent) => {
  if (event.button !== 0) return;
  if (canvasRef.value) {
    canvasRef.value.width = canvasRef.value.offsetWidth * scale;
    canvasRef.value.height = canvasRef.value.offsetHeight * scale;
    initParticles();
  }

  style.value += event.movementX * 2;
};
</script>

<template>
  <div class="adjust-card" :style="`width:${style}px`">
    <div class="message">
      Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod
      tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim
      veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea
      commodo consequat. Duis aute irure dolor in reprehenderit in voluptate
      velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat
      cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id
      est laborum.
      <div class="tail">
        <div class="cut"></div>
      </div>
      <canvas
        ref="canvasRef"
        width="600"
        height="300"
        class="block-mask"
        @mousemove="mouseMove"
        @mouseleave="mouseLeave"
      ></canvas>
    </div>
    <div ref="adjustHandleRef" class="adjust-handle"></div>
  </div>
</template>

<style scoped>
.adjust-card {
  margin: 0 auto;
  /* height: 300px; */
  display: flex;
  flex-direction: row;
  align-items: center;
  padding: 12px;
  margin-bottom: 12px;
  border-radius: 20px;
  background-color: rgba(121, 121, 121, 0.3);
}
.adjust-handle {
  cursor: ew-resize;
  flex-shrink: 0;
  height: 50px;
  width: 5px;
  margin-left: 12px;
  background-color: rgba(117, 117, 117, 0.7);
  border-radius: 6px;
}
.block-mask {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  border-radius: 12px;
}
.message {
  /* overflow: hidden; */
  color: white;
  position: relative;
  flex-grow: 0;
  text-align: start;
  padding: 8px 14px;
  margin: 0;
  border-radius: 12px;
  background-color: #387ef7;
}
.tail {
  position: absolute;
  bottom: 0;
  right: 0;
  overflow: hidden;
  height: 16px;
  width: 18px;
  margin-right: -12px;
  border-bottom-left-radius: 50%;
  background-color: #387ef7;
}
.tail .cut {
  height: 22px;
  width: 16px;
  margin-top: -5px;
  margin-left: 6px;
  border-bottom-left-radius: 50%;
  background-color: #3e3e3e;
}

@media (prefers-color-scheme: light) {
  .tail .cut {
    background-color: #d7d7d7;
  }
}
</style>
