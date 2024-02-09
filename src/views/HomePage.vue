<script setup lang="ts">
import {onMounted, ref, watch} from "vue";
import { IonContent, IonHeader, IonPage, IonTitle, IonToolbar, IonButton, IonIcon } from '@ionic/vue';
import { CameraPreview, CameraPreviewOptions } from '@capacitor-community/camera-preview';
import { camera, close} from 'ionicons/icons';

// Internal variables
const IMAGE_QUALITY = 85;
const photoSelector = '#walk-page:not(.ion-page-hidden) #directions-page:not(.ion-page-hidden) #photo .image';

// Refs
const previewStarted = ref(false)
const canTakePhoto = ref(true)
const cameraText = ref('')
const cameraTextClass = ref('')
const imageSrcData = ref('')
const imageLoadingHeight = ref(0);


const setNoCameraAvailable = (reason:string) => {
  canTakePhoto.value = false;
  cameraText.value = reason
  cameraTextClass.value = 'danger'
}

const unsetNoCameraAvailable = () => {
  canTakePhoto.value = true;
  cameraText.value = ''
  cameraTextClass.value = ''
}

const onTakePhoto = async (imageSrc: string) => {
  imageSrcData.value = imageSrc;
}


const onPressTakePhoto = async () => {
  try{
    const base64PictureData = await CameraPreview.capture({
      quality: IMAGE_QUALITY
    });
    const imageSrcData = 'data:image/jpeg;base64,' + base64PictureData.value;
    await onTakePhoto(imageSrcData)
    await stopCameraPreview();
  }catch (e:any) {
    const errorMessage = e.message;
    setNoCameraAvailable(errorMessage);
  }
}

const startCameraPreview = () => {
  requestAnimationFrame(() => {
    _startCameraPreview();
  })
}

const _startCameraPreview = async () => {
  if(previewStarted.value){
    return;
  }

  if(!canTakePhoto.value){
    console.warn('Camera preview not available');
    return;
  }

  // Get the viewport offset and dimensions of the camera preview
  const cameraPreview = document.getElementById('camera-preview');
  const cameraPreviewRect = cameraPreview?.getBoundingClientRect();

  const cameraPreviewOptions:CameraPreviewOptions = {
    parent: 'camera-preview',
    x: Math.ceil(cameraPreviewRect?.x || 0),
    y: Math.ceil(cameraPreviewRect?.y || 0) + 1,
    width: Math.ceil(cameraPreviewRect?.width || 0) - 2,
    height: Math.ceil(cameraPreviewRect?.height || 0) - 2,
    position: 'rear',
    enableZoom: true,
    toBack: false,
    storeToFile: false,
    lockAndroidOrientation: true,
    disableAudio: true,
  }

  try{
    unsetNoCameraAvailable();
    await CameraPreview.start(cameraPreviewOptions);
    document.documentElement.classList.add('camera-preview');
    previewStarted.value = true;
  }catch (e:any) {
    const errorMessage = e.message;
    if(errorMessage.match("camera already started")){
      // Try again
      console.debug(`Camera already started. Trying again...`)
      await stopCameraPreview();
      await _startCameraPreview();
    }else{
      setNoCameraAvailable(errorMessage);
    }
  }
}

const stopCameraPreview = async () => {
  try{
    await CameraPreview.stop();
    document.documentElement.classList.remove('camera-preview');
    previewStarted.value = false;
  }catch (e) {
    const errorMessage = e as string;
    setNoCameraAvailable(errorMessage);
    console.error(`Error stopping camera preview: ${errorMessage}`);
  }
}

const resolvePhotoAspectRatio =  () => {
  const photo = document.querySelector(photoSelector) as HTMLImageElement;
  const width = photo.offsetWidth;
  const height = photo.offsetHeight;
  const aspectRatio = width / height;

  // get the parent element of the photo
  const parent = photo.parentElement as HTMLElement;

  // reset
  parent.classList.remove('landscape')
  parent.classList.remove('portrait');
  parent.style.maxHeight = 'none';

  if(aspectRatio > 1){
    parent.classList.add('landscape');
  }else{
    parent.classList.add('portrait');
    parent.style.maxHeight = width + 'px';
  }
}

const waitForPhotoToRender = () => {
  const photo = document.querySelector(photoSelector) as HTMLElement;
  if(photo){
    const height = (photo as HTMLElement).offsetHeight;
    if(height > 0) {
      // if height is same as last time, image has finished rendering
      if(height === imageLoadingHeight.value){
        resolvePhotoAspectRatio();
        return;
      }else{
        imageLoadingHeight.value = height;
      }
    }
  }
  requestAnimationFrame(waitForPhotoToRender);
}

watch(imageSrcData, (value:string) => {
  if(value){
    imageLoadingHeight.value = 0;
    waitForPhotoToRender();
  }
})

const onRejectPhoto = async () => {
  imageSrcData.value = '';
  startCameraPreview();
}

onMounted(() => {
  startCameraPreview();
})
</script>

<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>Capacitor Camera Preview Test</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
      <ion-header collapse="condense">
        <ion-toolbar>
          <ion-title size="large">Capacitor Camera Preview Test</ion-title>
        </ion-toolbar>
      </ion-header>

      <div class="content-inner">

        <div id="camera" v-if="!imageSrcData">
          <div id="camera-preview" class="image"></div>

          <p class="text" v-if="cameraText" v-html="cameraText" :class="cameraTextClass"></p>
          <div class="buttons">
            <ion-button :disabled="!canTakePhoto" @click="onPressTakePhoto()" class="w-50">
              <ion-icon slot="end" :icon="camera" size="large" />
              <span>Take photo</span>
            </ion-button>
          </div>
        </div>

        <div class="photo-layout" v-else>
          <div id="photo">
            <div class="image-container">
              <img :src="imageSrcData" class="image" />
            </div>

            <div class="photo buttons">
              <ion-button @click="onRejectPhoto()" color="danger">
                <ion-icon slot="icon-only" :icon="close" />
              </ion-button>
            </div>
          </div>
        </div>

      </div>
    </ion-content>
  </ion-page>
</template>



<style scoped>
.buttons{
  text-align: center;
}
</style>
