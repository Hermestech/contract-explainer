<template>
  <div>
    <input type="file" @change="loadImages" multiple />
    <button @click="toggleCamera" :disabled="cameraActive">
        {{ cameraActive ? 'Cámara activada' : 'Activar cámara' }}
    </button>
    <div v-if="cameraActive" class="camera-container">
        <video ref="video" autoplay playsinline></video>
        <div class="guide"></div>
    </div>
    <image-preview 
        v-for="(image, index) in images" 
        :key="index" 
        :src="image.src" 
        :extractedText="image.extractedText"
    />
    <button @click="takePicture" :disabled="!cameraActive || takingPhoto">
        {{ takingPhoto ? 'Tomando foto...' : 'Tomar foto' }}
    </button>
    <button v-if="photoSrc" @click="resetPhoto">Tomar otra foto</button>
    <div v-if="photoSrc" class="photo-preview">
        <img :src="photoSrc" alt="Vista previa de la foto" />
    </div>
    <button @click="extractTextFromAllImages">Extraer texto</button>
    <pre v-if="allExtractedText">{{ allExtractedText }}</pre>
  </div>
</template>
<script>
import ImagePreview from '../components/ImagePreview.vue';
import Tesseract from 'tesseract.js'

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
  this.allExtractedText = '';
  for (let i = 0; i < this.images.length; i++) {
    try {
      this.allExtractedText += `Imagen ${i + 1}:\n`;
      const result = await Tesseract.recognize(this.images[i].src, "spa");
      this.allExtractedText += result.data.text;
      this.allExtractedText += '\n\n';
    } catch (error) {
      console.error("Error al extraer texto:", error);
      this.allExtractedText += 'Error al extraer texto.\n\n';
    }
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

  // Cierra la cámara después de tomar una foto
  this.toggleCamera();

  this.takingPhoto = false;
      },
resetPhoto() {
    this.photoSrc = null;
    this.toggleCamera();
},
}
}
</script>

<style scoped>
.camera-container {
  position: relative;
  width: 100%;
  max-width: 640px;
  height: 480px;
  overflow: hidden;
  margin: 1rem 0;
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
.guide {
  position: absolute;
  top: 10%;
  left: 10%;
  width: 80%;
  height: 80%;
  border: 2px dashed rgba(0, 255, 0, 0.5); /* Cambia el color a verde */
  pointer-events: none;
}
</style>
