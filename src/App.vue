<template>
  <h1>簽名測試</h1>
  <el-button plain @click="dialogVisible = true">簽名</el-button>

  <el-dialog v-model="dialogVisible" title="請簽名" @open="setCanvas" fullscreen>
    <el-radio-group v-model="signType">
      <el-radio-button label="手指" value="finger" />
      <el-radio-button label="Apple Pencil" value="pencil" />
      <el-radio-button label="滑鼠" value="mouse" />
    </el-radio-group>
    <div class="signature-pad">
      <canvas ref="drawCanvas" @mousedown.prevent="startDrawing" @mousemove.prevent="draw"
        @mouseup.prevent="stopDrawing" @mouseout.prevent="stopDrawing" @touchstart.prevent="startDrawing"
        @touchmove.prevent="draw" @touchend.prevent="stopDrawing"></canvas>
    </div>

    <template #footer>
      <div class="dialog-footer">
        <el-button @click="dialogVisible = false">關閉視窗</el-button>

        <el-button @click="Clear">清除</el-button>

        <el-button type="primary" @click="Save">存檔</el-button>
      </div>
    </template>
  </el-dialog>
</template>

<script setup>
  import { ref, nextTick, onMounted, onBeforeUnmount, } from "vue";
  import { ElMessage } from "element-plus";

  const dialogVisible = ref(false);

  const drawCanvas = ref(null);
  const ctx = ref(null);
  const isDrawing = ref(false);

  const points = ref([]); // 用來存放繪圖點
  const lastPoint = ref({ x: 0, y: 0 });



  onMounted(() => {
    window.addEventListener("resize", setCanvas);
  })

  onBeforeUnmount(() => {
    window.removeEventListener("resize", setCanvas);
  })

  const setCanvas = () => {
    nextTick(() => {
      const height = window.innerHeight - 170

      const canvas = drawCanvas.value;
      const container = canvas.parentElement;

      // 使用設備像素比來提高畫布清晰度
      const dpr = window.devicePixelRatio || 1;
      canvas.width = container.offsetWidth * dpr;
      // canvas.height = container.offsetHeight * dpr;
      canvas.height = height * dpr;

      // 縮放畫布以保持原始大小
      canvas.style.width = `${container.offsetWidth}px`;
      // canvas.style.height = `${container.offsetHeight}px`;
      canvas.style.height = `${height}px`;

      ctx.value = canvas.getContext("2d");

      // 縮放繪圖上下文
      ctx.value.scale(dpr, dpr);

      // 配置繪圖上下文以防止虛線和提高平滑度
      ctx.value.setLineDash([]);  // 清除任何可能的虛線設置
      ctx.value.strokeStyle = "#000000";
      ctx.value.lineJoin = "round";
      ctx.value.lineCap = "round";
      ctx.value.lineWidth = 2;

      // 啟用高品質抗鋸齒
      ctx.value.imageSmoothingEnabled = true;
      ctx.value.imageSmoothingQuality = 'high';
    });
  };

  const signType = ref('finger');

  const startDrawing = (e) => {
    let clientX = 0;
    let clientY = 0;
    if (signType.value === 'finger' || signType.value === 'pencil') {
      if (e.touches) {
        const touchType = e.touches[0].touchType
        if (touchType === 'direct' && signType.value === 'pencil') {
          ElMessage({ type: "error", message: '請選擇正確的裝置類型' });
          return;
        }
        if (touchType === 'stylus' && signType.value === 'finger') return;

        clientX = e.touches[0].clientX;
        clientY = e.touches[0].clientY;
      } else {
        ElMessage({ type: "error", message: '請選擇正確的裝置類型' });
        return;
      }
    } else {
      if (e.touches) {
        ElMessage({ type: "error", message: '請選擇正確的裝置類型' });
        return
      }
      clientX = e.clientX;
      clientY = e.clientY;
    }

    const rect = drawCanvas.value.getBoundingClientRect();

    if (!clientX || !clientY) return;

    const x = clientX - rect.left;
    const y = clientY - rect.top;

    isDrawing.value = true;

    // 確保畫筆開始於第一個點
    ctx.value.beginPath();
    ctx.value.moveTo(x, y);

    points.value = [{ x, y }];
    lastPoint.value = { x, y };
  };

  const stopDrawing = () => {
    if (isDrawing.value) {
      isDrawing.value = false;
      points.value = [];
      ctx.value.closePath();
    }
  };

  const draw = (e) => {
    if (!isDrawing.value || !ctx.value) return;

    const rect = drawCanvas.value.getBoundingClientRect();
    const clientX = e.clientX || (e.touches && e.touches[0].clientX);
    const clientY = e.clientY || (e.touches && e.touches[0].clientY);

    if (!clientX || !clientY) return;

    const x = clientX - rect.left;
    const y = clientY - rect.top;

    points.value.push({ x, y }); // 每次繪製時記錄新點

    // 取得中間點（控制點）
    const midPoint = {
      x: (lastPoint.value.x + x) / 2,
      y: (lastPoint.value.y + y) / 2,
    };

    // 使用二次貝塞爾曲線創建平滑的線條
    ctx.value.quadraticCurveTo(lastPoint.value.x, lastPoint.value.y, midPoint.x, midPoint.y);

    // 繪製線條
    ctx.value.stroke();

    // 更新最後一個點
    lastPoint.value = { x, y };
  };

  const Clear = () => {
    if (ctx.value) {
      ctx.value.clearRect(0, 0, drawCanvas.value.width, drawCanvas.value.height);
    }
  };

  const Save = () => {
    const base64 = drawCanvas.value.toDataURL();
    download(base64);
  };

  const download = (base64) => {
    const link = document.createElement("a");
    link.download = "signature.png";
    link.href = base64;
    link.click();
  };
</script>

<style scoped>

  .el-dialog__body {
    display: flex;
    flex-direction: column;
  }

  .signature-pad {
    width: 100%;
    height: 100%;
    background: #f5f5f5;
    margin-top: 1rem;
  }



  canvas {
    width: 100%;
    height: 100%;
    display: block;
  }
</style>