<template>
  <div class="main-container">
    <div class="content-container">
            <h1 class="title">Explicador de contratos🤖📄 </h1>
      <p class="disclaimer">
        <strong>Aviso</strong> Esta herramienta proporciona una guía rápida para comprender el contenido de los contratos, pero no constituye asesoría legal. Si necesitas asesoramiento legal, consulta a un abogado.
      </p>
      <p class="instructions">
        Esta herramienta se enfoca en contratos de uso común, como contratos de renta, contratos laborales y términos y condiciones. Para obtener los mejores resultados, sigue estas recomendaciones al tomar fotos:
        <ul>
          <li>Asegúrate de que las fotos sean nítidas y claras.</li>
          <li>Coloca el texto en primer plano y asegúrate de que esté bien iluminado.</li>
          <li>Evita sombras y reflejos sobre el texto.</li>
          <li>Utiliza un fondo con contraste respecto al texto para facilitar la lectura.</li>
        </ul>
        La herramienta se limita a un máximo de 10 imágenes.
      </p>
      <input type="file" @change="loadImages" multiple/>
      <div class="button-row">
        <button @click="toggleCamera" :disabled="cameraActive" class="mui-button">
            {{ cameraActive ? 'Cámara activada' : 'Activar cámara' }}
        </button>
        <button @click="takePicture" :disabled="!cameraActive || takingPhoto" class="mui-button">
            {{ takingPhoto ? 'Tomando foto...' : 'Tomar foto' }}
        </button>
        <button v-if="photoSrc" @click="resetPhoto" class="mui-button">Tomar otra foto</button>
        <button @click="extractTextFromAllImages" class="mui-button">
           Explicar el contrato
        </button>
      </div>
      <div v-if="cameraActive" class="camera-container">
        <video ref="video" autoplay playsinline></video>
        <div class="guide"></div>
      </div>
      <div class="image-grid">
        <image-preview
        v-for="(image, index) in images"
        :key="index"
        :src="image.src"
        :extractedText="image.extractedText"
        />
      </div>
      <div v-if="loading" class="spinner"></div>
      <div v-if="chatGPTOutput" class="chatgpt-response">
        <h3 v-t="'response'"></h3>
        <ul class="chat-list">
          <li class="chat-list" v-for="(item, index) in chatGPTOutput" :key="index">{{ item }}</li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script>
import ImagePreview from './ImagePreview.vue';

export default {
    components: {
        ImagePreview
    },
  data() {
    return {
        images: [],
        allExtractedText: '',
        cameraActive: false,
        cameraStream: null,
        photoSrc: null,
      takingPhoto: false,
      chatGPTOutput: '',
        loading: false
    }
  },
  methods: {
    loadImages(event) {
     const files = event.target.files;
  if (files.length > 10) {
    alert("Solo se permiten 10 imágenes como máximo.");
    return;
  }
  this.images = [];
  Array.from(files).forEach((file) => {
    const reader = new FileReader();
    reader.onload = (e) => {
      this.images.push({ src: e.target.result, extractedText: "" });
    };
    reader.readAsDataURL(file);
    });
},
    async extractTextFromAllImages() {
      this.loading = true;
      try {
        const formData = new FormData();
        this.images.forEach((image, index) => {
        const file = this.dataURLtoFile(image.src, `image-${index}.jpg`);
        formData.append("image", file);
        });
        const response = await fetch(process.env.TRANSLATE_URL, {
          method: "POST",
          body: formData,
        });

        if (!response.ok) {
          throw new Error("Error al extraer el texto de las imágenes.");
        }
        const data = await response.json();
        this.chatGPTOutput = data.response;
      } catch (error) {
        console.error(error);
        this.chatGPTOutput = 'Error al extraer el texto.';
      } finally {
        this.loading = false;
      }
},
async toggleCamera() {
  if (!this.cameraActive) {
    try {
      const constraints = { video: { facingMode: "environment" } };
      this.cameraStream = await navigator.mediaDevices.getUserMedia(constraints);

      // Agrega un pequeño retraso antes de asignar el objeto de transmisión al elemento de video
      setTimeout(() => {
        this.$refs.video.srcObject = this.cameraStream;
      }, 100);

      this.cameraActive = true;
    } catch (error) {
      console.error("Error al activar la cámara:", error);
      alert("No se pudo activar la cámara.");
    }
  } else {
    if (this.cameraStream) {
      this.cameraStream.getTracks().forEach((track) => track.stop());
      this.cameraStream = null;
    }
    this.cameraActive = false;
  }
},
async takePicture() {
  if (!this.cameraActive || this.takingPhoto) return;

  this.takingPhoto = true;

  const canvas = document.createElement("canvas");
  canvas.width = this.$refs.video.videoWidth;
  canvas.height = this.$refs.video.videoHeight;
  const context = canvas.getContext("2d");
  context.drawImage(this.$refs.video, 0, 0, canvas.width, canvas.height);
  this.photoSrc = canvas.toDataURL("image/jpeg");

  // Agrega la imagen tomada desde la cámara al array 'images'
  this.images.push({ src: this.photoSrc, extractedText: "" });

  // Cierra la cámara después de tomar una foto
  this.toggleCamera();

  this.takingPhoto = false;
      },
resetPhoto() {
    this.photoSrc = null;
    this.toggleCamera();
    },
    dataURLtoFile(dataurl, filename) {
      const arr = dataurl.split(",");
      const mime = arr[0].match(/:(.*?);/)[1];
      const bstr = atob(arr[1]);
      let n = bstr.length;
      const u8arr = new Uint8Array(n);

      while (n--) {
        u8arr[n] = bstr.charCodeAt(n);
      }
      return new File([u8arr], filename, { type: mime });
    }
}
}
</script>

<style scoped>

.main-container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  background-image: linear-gradient(0deg, rgba(189,34,195,0.9626444327731093) 0%, rgba(74,45,253,1) 100%);
}

.content-container {
  width: 100%;
  max-width: 800px;
  min-height: 80vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  background-color: white;
  padding: 20px;
  border-radius: 10px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}


.button-row {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 10px;
  margin-top: 40px;
  margin-bottom: 20px;
}

.mui-button {
  display: inline-block;
  background-color: #3f51b5;
  color: white;
  padding: 10px 15px;
  border: none;
  border-radius: 4px;
  font-size: 16px;
  font-weight: 500;
  text-transform: uppercase;
  cursor: pointer;
  transition: background-color 0.3s;
}

.mui-button:disabled {
  background-color: #c5cae9;
  cursor: not-allowed;
}

.mui-button:hover {
  background-color: #283593;
}

video {
  width: 100%;
  height: auto;
}

.guide {
  position: absolute;
  top: 10%;
  left: 10%;
  width: 80%;
  height: 80%;
  border: 2px dashed rgba(255, 255, 255, 0.5);
  pointer-events: none;
}

.photo-preview {
  width: 100%;
  max-width: 640px;
  margin: 1rem 0;
}

.photo-preview img {
  width: 100%;
  height: auto;
}
.camera-container {
  position: relative;
  display: inline-block;
}

.guide {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  border: 10px solid rgba(0, 255, 0, 0.5);
  box-sizing: border-box;
  pointer-events: none;
}

  .spinner {
  display: inline-block;
  width: 80px;
  height: 80px;
  border: 8px solid #f3f3f3;
  border-top: 8px solid #3498db;
  border-radius: 50%;
  animation: spin 2s linear infinite;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}
  
    .chatGPTOutput {
      margin-top: 1rem;
      padding: 1rem;
      border: 1px solid #ccc;
      border-radius: 4px;
      background-color: #f3f3f3;
      font-family: monospace;
      white-space: pre-wrap;
}

.image-grid {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
}

image-preview {
  flex-basis: 100%;
  max-width: 100%;
  padding: 8px;
}

@media (min-width: 768px) {
  image-preview {
    flex-basis: calc(100% / 3);
    max-width: calc(100% / 3);
  }
}
</style>
