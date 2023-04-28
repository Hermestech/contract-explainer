<template>
  <div class="main-container">
    <div class="content-container">
            <h1 class="title">Explicador de contratos🤖📄 </h1>
      <p class="disclaimer">
        <strong>Aviso:</strong> Esta herramienta proporciona una guía rápida para comprender el contenido de los contratos, pero no constituye asesoría legal. Si necesitas asesoramiento legal, consulta a un abogado.
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
      <div v-if="loading" class="spinner"></div>
      <div class="image-grid">
        <image-preview
        v-for="(image, index) in images"
        :key="index"
        :src="image.src"
        :extractedText="image.extractedText"
        />
      </div>
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
import ImagePreview from '../components/ImagePreview.vue';
import Tesseract from 'tesseract.js'
import { Configuration, OpenAIApi } from 'openai';

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
  this.allExtractedText = '';
  for (let i = 0; i < this.images.length; i++) {
    try {
      this.allExtractedText += `Imagen ${i + 1}:\n`;
      const result = await Tesseract.recognize(this.images[i].src, "spa");
      const cleanedText = this.removeUnnecessarySpaces(result.data.text);
      this.allExtractedText += cleanedText;
      this.allExtractedText += '\n\n';
    } catch (error) {
      console.error("Error al extraer texto:", error);
      this.allExtractedText += 'Error al extraer texto.\n\n';
    }
  }
  // Llamar a sendToChatGPT fuera del bucle for, después de extraer el texto de todas las imágenes
  this.sendToChatGPT(this.allExtractedText);
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
removeUnnecessarySpaces(text) {
  const lines = text.split('\n');
  const cleanedLines = lines.map(line => {
    return line.trim().replace(/\s+/g, ' ');
  });
  return cleanedLines.join('\n');
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
    async sendToChatGPT(text) {
      try {
        const apiKey = process.env.CHATGPT_API_KEY;
        const configuration = new Configuration({ apiKey });
        const openai = new OpenAIApi(configuration);

        const basePrompt = "Actua como si fueras un abogado con mucha experiencia en derecho Mexicano. Enlista los puntos más relevantes del documento, recuerda devolverlos en formato de lista para tener mayor legibilidad. Explica cada punto como si tuviera 10 años,también explícame cuáles serían mis obligaciones si firmara ese contrato y cuáles serían las obligaciones de la otra parte. Dime si encuentras algo que pudiera resultarme muy perjudicial si firmara. Si encuentras conceptos dificiles de comprender para mi que no soy abogado, explicame qué significan"

        const completion = await openai.createCompletion({
          model: "text-davinci-003",
          prompt: basePrompt + text,
          max_tokens: 1200,
          temperature: 0.3,
        })

        this.chatGPTOutput = this.parseString(completion.data.choices[0].text)

      } catch (error) { 
        console.error("Error al enviar texto a ChatGPT:", error);
        this.chatGPTOutput = 'Error al enviar texto a ChatGPT.'
      }
    },
    parseString(input) {
    const regex = /\d+\.\s/g;
    const splitString = input.split(regex);

    return splitString.filter((item) => item !== '');
    },
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
