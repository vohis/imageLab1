<template>
    <div>
      <h1>Загрузка изображения</h1>
      <input type="file" @change="onFileChange" accept="image/*" />
      <input type="text" v-model="imageUrl" placeholder="Введите URL изображения" @change="loadImageFromUrl" />
      
      <canvas
        ref="canvas"
        @click="getColor"
        @mousemove="updateHoverInfo"
        style="border: 1px solid black; margin-top: 10px; width: 400px; height: auto;"
      ></canvas>
  
      <div v-if="imageLoaded" class="info-panel">
        <p>Координаты: {{ mouseX }}, {{ mouseY }}</p>
        <p>Цвет: 
          <span
            class="color-square"
            :style="{ backgroundColor: `rgba(${color.r}, ${color.g}, ${color.b}, ${color.a / 255})` }"
          ></span>
          rgba({{ color.r }}, {{ color.g }}, {{ color.b }}, {{ (color.a / 255).toFixed(2) }})
        </p>
        <p>Размер: {{ imageWidth }} x {{ imageHeight }} px</p>
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
        canvasScale: 0.5, // Масштаб для отображения изображения
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
        this.loadImage(this.imageUrl);
      },
      loadImage(src) {
        this.image = new Image();
        this.image.src = src;
        this.image.crossOrigin = "Anonymous"; // Добавлено для поддержки изображений с других доменов
        this.image.onload = () => {
          this.imageLoaded = true;
          this.imageWidth = this.image.width;
          this.imageHeight = this.image.height;
          this.renderImage();
        };
        this.image.onerror = () => {
          alert("Не удалось загрузить изображение. Проверьте URL.");
        };
      },
      renderImage() {
        const canvas = this.$refs.canvas;
        const ctx = canvas.getContext("2d");
        canvas.width = this.image.width * this.canvasScale; // Установим ширину по масштабу
        canvas.height = this.image.height * this.canvasScale; // Установим высоту по масштабу
        ctx.drawImage(this.image, 0, 0, canvas.width, canvas.height);
      },
      getColor(event) {
        const canvas = this.$refs.canvas;
        const ctx = canvas.getContext("2d");
        const rect = canvas.getBoundingClientRect();
        const x = Math.round((event.clientX - rect.left) / this.canvasScale); // Корректируем координаты
        const y = Math.round((event.clientY - rect.top) / this.canvasScale); // Корректируем координаты
  
        // Извлекаем цвет пикселя
        const pixelData = ctx.getImageData(x, y, 1, 1).data;
        this.color = {
          r: pixelData[0],
          g: pixelData[1],
          b: pixelData[2],
          a: pixelData[3],
        };
      },
      updateHoverInfo(event) {
        const canvas = this.$refs.canvas;
        const rect = canvas.getBoundingClientRect();
        this.mouseX = Math.round((event.clientX - rect.left) / this.canvasScale); // Корректируем координаты
        this.mouseY = Math.round((event.clientY - rect.top) / this.canvasScale); // Корректируем координаты
      },
    },
  };
  </script>
  
  <style scoped>
  .info-panel {
    margin-top: 10px;
  }
  .color-square {
    display: inline-block;
    width: 20px;
    height: 20px;
    margin-left: 10px;
    border: 1px solid #000;
  }
  </style>
  