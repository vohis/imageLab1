<template>
  <div class="container">
    <!-- Квадратное окно для холста с изображением -->
    <div class="canvas-container" @mousedown="startPanning" @mousemove="panImage" @mouseup="stopPanning" @mouseleave="stopPanning">
      <canvas
        ref="canvas"
        class="image-canvas"
        @click="getColor"
        @mousemove="updateHoverInfo"
        :style="{ transform: `scale(${canvasScale / 100}) translate(${panX}px, ${panY}px)`, transformOrigin: '0 0' }"
      ></canvas>
    </div>

    <!-- Панель управления -->
    <div class="bottom-panel">
      <!-- Первая строка: Загрузка изображения -->
      <div class="file-inputs">
        <input type="file" @change="onFileChange" accept="image/*" />
        <input type="text" v-model="imageUrl" placeholder="Введите URL изображения" @keydown.enter="loadImageFromUrl" />
        <button @click="loadImageFromUrl">Загрузить</button>
      </div>

      <!-- Вторая строка: Контролы для изображения -->
      <div v-if="imageLoaded" class="controls">
        <label>Масштаб:</label>
        <input type="range" min="10" max="300" v-model="canvasScale" @input="renderImage" />
        <span>{{ canvasScale }}%</span>

        <button :class="{ active: isHandActive }" @click="toggleHandTool">✋ Рука</button>
        <!-- Кнопка для активации инструмента пипетки -->
<button :class="{ active: isEyedropperActive }" @click="toggleEyedropperTool">🖍️ Пипетка</button>

    <!-- Информационная панель для пипетки -->
      <div v-if="isEyedropperActive" class="eyedropper-info-panel">
        <h4>Информация о цветах</h4>
        <button @click="closeEyedropperInfo">Закрыть</button>
      <div>
        <p>Первый цвет:
      <span class="tooltip" @mouseenter="showTooltip('rgba')" @mouseleave="hideTooltip">
        <span class="color-square" :style="{ backgroundColor: `rgba(${color1.r}, ${color1.g}, ${color1.b}, ${color1.a / 255})` }"></span>
        rgba({{ color1.r }}, {{ color1.g }}, {{ color1.b }}, {{ (color1.a / 255).toFixed(2) }})
        <span class="tooltip-text">R: 0-255, G: 0-255, B: 0-255, A: 0-255</span>
      </span>
    </p>
    <p>Второй цвет:
      <span class="tooltip" @mouseenter="showTooltip('rgba')" @mouseleave="hideTooltip">
        <span class="color-square" :style="{ backgroundColor: `rgba(${color2.r}, ${color2.g}, ${color2.b}, ${color2.a / 255})` }"></span>
        rgba({{ color2.r }}, {{ color2.g }}, {{ color2.b }}, {{ (color2.a / 255).toFixed(2) }})
        <span class="tooltip-text">R: 0-255, G: 0-255, B: 0-255, A: 0-255</span>
      </span>
    </p>
      </div>
       <!-- Блок для контраста -->
  <div>
    <h5>Контраст: <span :class="{ insufficient: contrastRatio < 4.5 }">{{ contrastRatio.toFixed(2) }}:1</span></h5>
  </div>
</div>

        <button @click="openResizeModal">Изменить размер</button>
        <button @click="saveImage">Сохранить</button>
      </div>

      <!-- Информация об изображении -->
      <div v-if="imageLoaded" class="info-panel">
        <p>Координаты: {{ mouseX }}, {{ mouseY }}</p>
        <p>Цвет:
          <span class="color-square" :style="{ backgroundColor: `rgba(${color.r}, ${color.g}, ${color.b}, ${color.a / 255})` }"></span>
          rgba({{ color.r }}, {{ color.g }}, {{ color.b }}, {{ (color.a / 255).toFixed(2) }})
        </p>
        <p>Размер: {{ imageWidth }} x {{ imageHeight }} px</p>
      </div>
    </div>

    <!-- Модальное окно изменения размера изображения -->
    <div v-if="isResizeModalVisible" class="modal-overlay">
      <div class="modal-window">
        <h3>Изменить размер изображения</h3>

        <!-- Информация о пикселях -->
        <p>Исходное количество пикселей: {{ originalPixels.toFixed(2) }} Мпикс</p>
        <p>Новое количество пикселей: {{ newPixels.toFixed(2) }} Мпикс</p>

        <!-- Выбор единиц измерения -->
        <label for="unit">Единица измерения:</label>
        <select v-model="resizeUnit">
          <option value="percent">Проценты</option>
          <option value="pixels">Пиксели</option>
        </select>

        <!-- Поля для ввода ширины и высоты -->
        <label for="resizeWidth">Ширина:</label>
        <input type="number" v-model.number="newWidth" @input="updateHeight" :disabled="resizeUnit === 'percent'" />
        <label for="resizeHeight">Высота:</label>
        <input type="number" v-model.number="newHeight" :disabled="maintainAspectRatio || resizeUnit === 'percent'" />

        <!-- Чекбокс для сохранения пропорций -->
        <label>
          <input type="checkbox" v-model="maintainAspectRatio" />
          Сохранять пропорции
        </label>

        <!-- Алгоритм интерполяции с тултипом -->
        <label for="algorithm">Алгоритм интерполяции:</label>
        <select v-model="interpolationAlgorithm" title="Ближайший сосед: выбирает ближайший пиксель, не добавляя сглаживание.">
          <option value="nearest-neighbor">Ближайший сосед</option>
        </select>

        <!-- Проверка пользовательского ввода -->
        <p v-if="inputError" class="error-message">{{ inputError }}</p>

        <div class="button-group">
          <button @click="applyResize">Применить</button>
          <button @click="closeResizeModal">Отмена</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      imageUrl: "",
      image: null,
      imageLoaded: false,
      mouseX: 0,
      mouseY: 0,
      color: { r: 0, g: 0, b: 0, a: 0 },
      imageWidth: 0,
      imageHeight: 0,
      canvasScale: 100,
      isResizeModalVisible: false,
      newWidth: 0,
      newHeight: 0,
      originalPixels: 0,  
      newPixels: 0,
      resizeUnit: 'pixels',
      interpolationAlgorithm: 'nearest-neighbor',
      maintainAspectRatio: true,
      aspectRatio: 1,
      isPanning: false,
      isHandActive: false,
      panX: 0,
      panY: 0,
      startX: 0,
      startY: 0,
      inputError: null,  
      isEyedropperActive: false,  
    color1: { r: 0, g: 0, b: 0, a: 0 },  
    color2: { r: 0, g: 0, b: 0, a: 0 },  
    isTooltipVisible: false,
    tooltipText: '',
    contrastRatio: 0,  
    };
  },
  methods: {
    onFileChange(event) {
      const file = event.target.files[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = (e) => {
          this.loadImage(e.target.result);
        };
        reader.readAsDataURL(file);
      }
    },
    loadImageFromUrl() {
      if (!this.imageUrl) {
        alert("Пожалуйста, введите корректный URL изображения.");
        return;
      }
      this.loadImage(this.imageUrl);
    },
    loadImage(src) {
      const img = new Image();
      img.src = src;
      img.crossOrigin = "Anonymous";
      img.onload = () => {
        this.image = img;
        this.imageLoaded = true;
        this.imageWidth = this.image.width;
        this.imageHeight = this.image.height;
        this.newWidth = this.imageWidth;
        this.newHeight = this.imageHeight;
        this.aspectRatio = this.imageWidth / this.imageHeight;
        this.originalPixels = (this.imageWidth * this.imageHeight) / 1e6;
        this.newPixels = this.originalPixels;
        this.renderImage();
      };
      img.onerror = (error) => {
        console.error("Ошибка загрузки изображения: ", error);
        alert("Не удалось загрузить изображение. Проверьте URL.");
        this.imageLoaded = false;
      };
    },
    renderImage() {
      if (!this.image) return;
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");
      ctx.clearRect(0, 0, canvas.width, canvas.height);

      const scaleFactor = this.canvasScale / 100;
      canvas.width = this.imageWidth * scaleFactor;
      canvas.height = this.imageHeight * scaleFactor;

      ctx.drawImage(this.image, 0, 0, canvas.width, canvas.height);
    },
    openResizeModal() {
      this.isResizeModalVisible = true;
    },
    closeResizeModal() {
      this.isResizeModalVisible = false;
    },
    updateHeight() {
      if (this.maintainAspectRatio) {
        this.newHeight = Math.round(this.newWidth / this.aspectRatio);
      }
      this.calculateNewPixels();
    },
    calculateNewPixels() {
      this.newPixels = (this.newWidth * this.newHeight) / 1e6;
    },
    applyResize() {
      if (this.newWidth <= 0 || this.newHeight <= 0) {
        this.inputError = "Укажите допустимые значения ширины и высоты.";
        return;
      }
      this.resizeImage();
      this.closeResizeModal();
    },
    resizeImage() {
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");
      const imageData = ctx.getImageData(0, 0, this.imageWidth, this.imageHeight);
      const newImageData = ctx.createImageData(this.newWidth, this.newHeight);

      for (let y = 0; y < this.newHeight; y++) {
        for (let x = 0; x < this.newWidth; x++) {
          const nearestX = Math.floor((x / this.newWidth) * this.imageWidth);
          const nearestY = Math.floor((y / this.newHeight) * this.imageHeight);

          const oldIndex = (nearestY * this.imageWidth + nearestX) * 4;
          const newIndex = (y * this.newWidth + x) * 4;

          newImageData.data[newIndex] = imageData.data[oldIndex];
          newImageData.data[newIndex + 1] = imageData.data[oldIndex + 1];
          newImageData.data[newIndex + 2] = imageData.data[oldIndex + 2];
          newImageData.data[newIndex + 3] = imageData.data[oldIndex + 3];
        }
      }

      canvas.width = this.newWidth;
      canvas.height = this.newHeight;
      ctx.putImageData(newImageData, 0, 0);

      this.imageWidth = this.newWidth;
      this.imageHeight = this.newHeight;
    },
    saveImage() {
      const canvas = this.$refs.canvas;
      const link = document.createElement('a');
      link.href = canvas.toDataURL('image/png');
      link.download = 'image.png';
      link.click();
    },
    toggleHandTool() {
      this.isHandActive = !this.isHandActive;
    },
    startPanning(event) {
      if (!this.isHandActive) return;
      this.isPanning = true;
      this.startX = event.clientX - this.panX;
      this.startY = event.clientY - this.panY;
    },
    panImage(event) {
      if (!this.isPanning) return;
      this.panX = event.clientX - this.startX;
      this.panY = event.clientY - this.startY;
    },
    stopPanning() {
      this.isPanning = false;
    },
    updateHoverInfo(event) {
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");
      const rect = canvas.getBoundingClientRect();
      const x = event.clientX - rect.left;
      const y = event.clientY - rect.top;
      this.mouseX = Math.floor(x);
      this.mouseY = Math.floor(y);

      const pixelData = ctx.getImageData(this.mouseX, this.mouseY, 1, 1).data;
      this.color = {
        r: pixelData[0],
        g: pixelData[1],
        b: pixelData[2],
        a: pixelData[3],
      };
    },
    getColor(event) {
      this.updateHoverInfo(event);
    },
    toggleEyedropperTool() {
    this.isEyedropperActive = !this.isEyedropperActive;
    if (this.isEyedropperActive) {
      this.color1 = { r: 0, g: 0, b: 0, a: 0 }; 
      this.color2 = { r: 0, g: 0, b: 0, a: 0 };
    }
  },
  closeEyedropperInfo() {
    this.isEyedropperActive = false;
  },
  getColor(event) {
    this.updateHoverInfo(event);
    if (this.isEyedropperActive) {
      if (event.altKey || event.ctrlKey || event.shiftKey) {
        this.color2 = { ...this.color }; 
      } else {
        this.color1 = { ...this.color }
      }
    }
  },
  calculateContrast(color1, color2) {
    const luminance = (color) => {
      const r = color.r / 255;
      const g = color.g / 255;
      const b = color.b / 255;

      const adjustedR = r <= 0.03928 ? r / 12.92 : Math.pow((r + 0.055) / 1.055, 2.4);
      const adjustedG = g <= 0.03928 ? g / 12.92 : Math.pow((g + 0.055) / 1.055, 2.4);
      const adjustedB = b <= 0.03928 ? b / 12.92 : Math.pow((b + 0.055) / 1.055, 2.4);

      return 0.2126 * adjustedR + 0.7152 * adjustedG + 0.0722 * adjustedB;
    };

    const lum1 = luminance(color1);
    const lum2 = luminance(color2);

    const lighter = Math.max(lum1, lum2);
    const darker = Math.min(lum1, lum2);

    return (lighter + 0.05) / (darker + 0.05);
  },

  updateContrast() {
    this.contrastRatio = this.calculateContrast(this.color1, this.color2);
  },

  getColor(event) {
    this.updateHoverInfo(event);
    if (this.isEyedropperActive) {
      if (event.altKey || event.ctrlKey || event.shiftKey) {
        this.color2 = { ...this.color };
        this.updateContrast(); 
      } else {
        this.color1 = { ...this.color }; 
        this.updateContrast();
      }
    }
  },

  showTooltip(type) {
    this.isTooltipVisible = true;
    this.tooltipText = type === 'rgba' ? 'R: 0-255, G: 0-255, B: 0-255, A: 0-255' : '';
  },

  hideTooltip() {
    this.isTooltipVisible = false;
  },
},

};
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  padding: 50px;
  box-sizing: border-box;
}

.canvas-container {
  flex-grow: 1;
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: #f0f0f0;
  overflow: auto;
  position: relative;
  border: 2px solid #ccc;
}

.image-canvas {
  transform-origin: 0 0;
  will-change: transform;
  transition: transform 0.2s ease;
}

.bottom-panel {
  display: flex;
  flex-direction: column;
  gap: 5px;
  width: 100%;
  background-color: #333;
  color: white;
  padding: 10px;
  font-size: 12px;
}

.file-inputs,
.controls,
.info-panel {
  display: flex;
  gap: 10px;
  justify-content: space-around;
}

.color-square {
  display: inline-block;
  width: 16px;
  height: 16px;
  margin-left: 8px;
  border: 1px solid #000;
}

.active {
  background-color: #007bff;
  color: white;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.8);
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal-window {
  background-color: #222;
  padding: 20px;
  border-radius: 8px;
  width: 400px;
  text-align: center;
  color: white;
}

.button-group {
  margin-top: 20px;
}

.error-message {
  color: red;
}

.tooltip {
  position: relative;
  display: inline-block;
  cursor: pointer;
}

.tooltip-text {
  visibility: hidden;
  width: 120px;
  background-color: black;
  color: #fff;
  text-align: center;
  border-radius: 5px;
  padding: 5px;
  position: absolute;
  z-index: 1;
  bottom: 125%;
  left: 50%;
  margin-left: -60px;
  opacity: 0;
  transition: opacity 0.3s;
}

.tooltip:hover .tooltip-text {
  visibility: visible;
  opacity: 1;
}

.insufficient {
  color: red; /* Цвет для недостаточного контраста */
}

</style>
