<template>
  <v-container class="fill-height">
    <v-row align="center">
      <v-col class="d-flex flex-column">
        <v-file-input label="Image input" variant="underlined" show-size counter prepend-icon="mdi-camera"
          @change="storeFile"></v-file-input>
        <v-text-field :disabled="!toggleImgConversion" class="w-50 align-self-center" clearable variant="underlined"
          label="Secret Message" v-model="secretMessage"></v-text-field>
      </v-col>
      <v-col align="center" class="d-flex flex-column">
        <v-btn color="gray" density="default" height="50" variant="text" :loading=converting @click="convertFile"
          :ripple="false">
          convert
        </v-btn>
        <div>
          <v-btn-toggle>
            <v-btn color="gray" density="default" height="50" variant="text" @click="switchConversion"
              :disabled=toggleImgConversion>
              img
            </v-btn>
            <v-btn color="gray" density="default" height="50" variant="text" @click="switchConversion"
              :disabled=!toggleImgConversion>
              text
            </v-btn>
          </v-btn-toggle>
        </div>
      </v-col>
      <v-col align="center" class="d-flex flex-column">
        <canvas v-show="false" ref="canvasRef"></canvas>
        <v-img v-if="toggleImgConversion" :height="300" aspect-ratio="16/9" cover :src=image class="rounded-lg"></v-img>
        <div v-else: ref="divRef"> {{ secretResult }}</div>
        <v-btn color="gray" density="default" variant="text" v-if="imageLoaded && toggleImgConversion"
          @click="downloadImage">Download Image</v-btn>
      </v-col>
    </v-row>
  </v-container>
</template>

<script setup>
import { ref, nextTick } from 'vue'

const secretMessage = ref('')
const secretResult = ref('')
const image = ref(null)
const imageLoaded = ref(false)
const converting = ref(false)
const storedFile = ref(null)
const toggleImgConversion = ref(false)
const canvasRef = ref(null)

const storeFile = (event) => {
  storedFile.value = event.target.files[0];
};

const convertFile = async () => {
  if (!storedFile.value) return;
  const reader = new FileReader();
  converting.value = true

  reader.onload = async (e) => {
    await nextTick();
    const canvas = canvasRef.value;
    const ctx = canvas.getContext('2d');
    const img = new Image();
    img.onload = () => {
      canvas.width = img.width;
      canvas.height = img.height;
      ctx.drawImage(img, 0, 0);
      if (toggleImgConversion.value && secretMessage.value) {
        encodeMessage(ctx, img.width, img.height, secretMessage.value);
        image.value = canvas.toDataURL();
        secretResult.value = ''
        imageLoaded.value = true
      } else {
        secretResult.value = decodeMessage(ctx, img.width, img.height);
        imageLoaded.value = false
      }
      converting.value = false

    };
    img.src = e.target.result;
  };
  reader.readAsDataURL(storedFile.value);
};


function switchConversion() {
  toggleImgConversion.value = !(toggleImgConversion.value)
  if (toggleImgConversion.value === false) {
    storeFile.value = null
    image.value = null
    secretMessage.value = ''
    secretResult.value = ''
    imageLoaded.value = false
  }
}

function encodeMessage(ctx, width, height, message) {
  const binaryMessage = message.split('').map(char => char.charCodeAt(0).toString(2).padStart(8, '0')).join('') + '00000000';
  let dataIndex = 0;
  for (let x = 0; x < width; x++) {
    for (let y = 0; y < height; y++) {
      if (dataIndex < binaryMessage.length) {
        const pixel = ctx.getImageData(x, y, 1, 1);
        const data = pixel.data;
        data[0] = (data[0] & ~1) | parseInt(binaryMessage[dataIndex], 2);
        ctx.putImageData(pixel, x, y);
        dataIndex++;
      } else {
        return;
      }
    }
  }
}

function decodeMessage(ctx, width, height) {
  let binaryMessage = '';
  for (let x = 0; x < width; x++) {
    for (let y = 0; y < height; y++) {
      const pixel = ctx.getImageData(x, y, 1, 1);
      const data = pixel.data;
      const lsb = data[0] & 1;
      binaryMessage += lsb.toString();
    }
  }

  let message = '';
  let found = false;
  for (let i = 0; i < binaryMessage.length; i += 8) {
    const byte = binaryMessage.slice(i, i + 8);
    if (byte === '00000000') {
      found = true
      break;
    }

    const asciiCode = parseInt(byte, 2);
    message += String.fromCharCode(asciiCode);
  }

  if (!found) {
    return "There is no message encoded"
  }

  return message;
}

const downloadImage = () => {
  const downloadLink = document.createElement('a');

  const fileName = "encoded_img.png";

  downloadLink.href = image.value;

  downloadLink.download = fileName;

  document.body.appendChild(downloadLink);

  downloadLink.click();

  document.body.removeChild(downloadLink);
};



</script>

