<script setup lang="ts">
import {onMounted, ref, watch} from "vue";
import {IonContent, IonHeader, IonPage, IonTitle, IonToolbar, IonButton, IonIcon, IonRange, IonToggle} from '@ionic/vue';
import {CameraPreview, CameraPreviewOptions} from '@capacitor-community/camera-preview';
import {addCircleOutline, camera, close, refresh, removeCircleOutline} from 'ionicons/icons';
import {Capacitor} from "@capacitor/core";

// Internal variables
const IMAGE_QUALITY = 85;
const photoSelector = '#photo .image';
const ZOOM_STEPS = 10;
const MIN_ZOOM = 1;
const MAX_ZOOM_LIMIT = 5;

// Refs
const previewStarted = ref(false)
const canTakePhoto = ref(true)
const cameraText = ref('')
const cameraTextClass = ref('')
const imgSrc = ref('')
const imageLoadingHeight = ref(0);
const maxZoom = ref(0)
const currentZoom = ref(0)
const zoomRangeRef = ref()
const zoomStep = ref();
const storeToFile = ref(false);


const setNoCameraAvailable = (reason: string) => {
  canTakePhoto.value = false;
  cameraText.value = reason
  cameraTextClass.value = 'danger'
}

const unsetNoCameraAvailable = () => {
  canTakePhoto.value = true;
  cameraText.value = ''
  cameraTextClass.value = ''
}

const onPressFlip = async () => {
  try {
    await CameraPreview.flip();
  } catch (e: any) {
    const errorMessage = e.message;
    console.error(`Error flipping camera: ${errorMessage}`);
  }
}

const onTakePhoto = async (newSrc: string) => {
  imgSrc.value = newSrc;
}


const onPressTakePhoto = async () => {
  try {
    const result = await CameraPreview.capture({
      quality: IMAGE_QUALITY
    });
    let imgSrc;
    if(storeToFile.value){
      imgSrc = Capacitor.convertFileSrc(result.value);
    }else{
      imgSrc = 'data:image/jpeg;base64,' + result.value;
    }

    await onTakePhoto(imgSrc)
    await stopCameraPreview();
  } catch (e: any) {
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
  if (previewStarted.value) {
    return;
  }

  if (!canTakePhoto.value) {
    console.warn('Camera preview not available');
    return;
  }

  // Get the viewport offset and dimensions of the camera preview
  const cameraPreview = document.getElementById('camera-preview');
  const cameraPreviewRect = cameraPreview?.getBoundingClientRect();

  const cameraPreviewOptions: CameraPreviewOptions = {
    parent: 'camera-preview',
    x: Math.ceil(cameraPreviewRect?.x || 0),
    y: Math.ceil(cameraPreviewRect?.y || 0) + 1,
    width: Math.ceil(cameraPreviewRect?.width || 0) - 2,
    height: Math.ceil(cameraPreviewRect?.height || 0) - 2,
    position: 'rear',
    enableZoom: true,
    toBack: false,
    storeToFile: storeToFile.value,
    lockAndroidOrientation: true,
    disableExifHeaderStripping: false,
    maxZoomLimit: MAX_ZOOM_LIMIT,
  }

  try {
    unsetNoCameraAvailable();

    // make sure it's stopped before starting
    try{await CameraPreview.stop();} catch(e){
      null; // ignore
    }

    await CameraPreview.start(cameraPreviewOptions);

    try{
      const _maxZoom = await CameraPreview.getMaxZoom(),
          deviceMaxZoom = _maxZoom.value;
      console.log('Device max zoom:', deviceMaxZoom)
      maxZoom.value = Math.min(deviceMaxZoom, MAX_ZOOM_LIMIT);

      zoomStep.value = maxZoom.value / ZOOM_STEPS;

      const _currentZoom = await CameraPreview.getZoom();
      currentZoom.value = _currentZoom.value;
    }catch (e: any) {
      console.error(`Error getting camera zoom: ${e.message}`);
    }

    document.documentElement.classList.add('camera-preview');
    previewStarted.value = true;
  } catch (e: any) {
    const errorMessage = e.message;
    if (errorMessage.match("camera already started")) {
      // Try again
      console.debug(`Camera already started. Trying again...`)
      await stopCameraPreview();
      await _startCameraPreview();
    } else {
      setNoCameraAvailable(errorMessage);
    }
  }
}

const stopCameraPreview = async () => {
  try {
    await CameraPreview.stop();
    document.documentElement.classList.remove('camera-preview');
    previewStarted.value = false;
  } catch (e) {
    const errorMessage = e as string;
    setNoCameraAvailable(errorMessage);
    console.error(`Error stopping camera preview: ${errorMessage}`);
  }
}

const resolvePhotoAspectRatio = () => {
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

  if (aspectRatio > 1) {
    parent.classList.add('landscape');
  } else {
    parent.classList.add('portrait');
    parent.style.maxHeight = width + 'px';
  }
}

const waitForPhotoToRender = () => {
  const photo = document.querySelector(photoSelector) as HTMLElement;
  if (photo) {
    const height = (photo as HTMLElement).offsetHeight;
    if (height > 0) {
      // if height is same as last time, image has finished rendering
      if (height === imageLoadingHeight.value) {
        resolvePhotoAspectRatio();
        return;
      } else {
        imageLoadingHeight.value = height;
      }
    }
  }
  requestAnimationFrame(waitForPhotoToRender);
}

const onChangeZoom = (zoom:number) => {
  try {
    CameraPreview.setZoom({zoom});
  }catch (e: any) {
    console.error(`Error setting camera zoom: ${e.message}`);
  }
}

const onTapZoomOut = () => {
  let zoom = Math.max(MIN_ZOOM, currentZoom.value - zoomStep.value);
  zoom = Math.round(zoom);
  currentZoom.value = zoom;
}

const onTapZoomIn = () => {
  let zoom = Math.min(maxZoom.value, currentZoom.value + zoomStep.value);
  zoom = Math.round(zoom);
  currentZoom.value = zoom;
}

watch(imgSrc, (value: string) => {
  if (value) {
    imageLoadingHeight.value = 0;
    waitForPhotoToRender();
  }
})

watch(currentZoom, (value: number, oldValue:number) => {
  if(value === oldValue) return;
  console.log('Zoom changed to', value);
  onChangeZoom(value);
})

watch(maxZoom, (value: number) => {
  console.log('Max zoom changed to', value);
})

watch (zoomStep, (value: number) => {
  console.log('Zoom step changed to', value);
})

const onRejectPhoto = async () => {
  imgSrc.value = '';
  startCameraPreview();
}

const onZoomLevelChanged = (event: any) => {
  const zoom = event.level;
  if(zoom != undefined){
    debouncedZoomLevelChange(zoom);
  }

}

const debounce = (func: (...args: any[]) => void, wait: number) => {
  let timeout: any;
  return function(this: any, ...args: any[]) {
    clearTimeout(timeout);
    // eslint-disable-next-line prefer-spread
    timeout = setTimeout(() => func.apply(null, args), wait);
  }
}

const zoomLevelChanged = (zoom: number) => {
  console.log(`zoom changed to: ${zoom}`);
  currentZoom.value = zoom;
}
const debouncedZoomLevelChange = debounce(zoomLevelChanged, 100);

const onChangeStoreToFile = (value: boolean) => {
  storeToFile.value = value;
  imgSrc.value = '';
}

onMounted(async () => {
  (window as any).addEventListener('CameraPreview.zoomLevelChanged', onZoomLevelChanged);
  startCameraPreview();

  try{
    const cameraCharacteristics = await CameraPreview.getCameraCharacteristics();
    console.dir(cameraCharacteristics);
  }catch (e: any) {
    console.error(`Error getting camera characteristics: ${e.message}`);
  }
})
</script>

<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>Capacitor Camera Preview Test</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content>
      <ion-header collapse="condense">
        <ion-toolbar>
          <ion-title size="large">Capacitor Camera Preview Test</ion-title>
        </ion-toolbar>
      </ion-header>

      <div class="content-inner">

        <div id="camera" v-if="!imgSrc">
          <div id="camera-preview" class="image"></div>

          <p class="text" v-if="cameraText" v-html="cameraText" :class="cameraTextClass"></p>
          <div class="buttons">
            <ion-button :disabled="!canTakePhoto" @click="onPressTakePhoto()">
              <ion-icon slot="end" :icon="camera" size="large"/>
              <span>Take photo</span>
            </ion-button>
            <ion-button :disabled="!canTakePhoto" @click="onPressFlip()">
              <ion-icon slot="end" :icon="refresh" size="large"/>
              <span>Flip camera</span>
            </ion-button>
          </div>

          <ion-range
              v-model="currentZoom"
              ref="zoomRangeRef"
              aria-label="Camera zoom level"
              :max="maxZoom"
              :min="MIN_ZOOM"
          >
            <ion-icon @click="onTapZoomOut()" slot="start" :icon="removeCircleOutline" color="primary"></ion-icon>
            <ion-icon @click="onTapZoomIn()" slot="end" :icon="addCircleOutline" color="primary"></ion-icon>
          </ion-range>

          <div class="store-to-file">
            <ion-toggle
                :enable-on-off-labels="true"
                @ionChange="onChangeStoreToFile($event.target.checked)"
                :checked="storeToFile">
              Store to file
            </ion-toggle>
          </div>

        </div>

        <div class="photo-layout" v-else>
          <div id="photo">
            <div class="image-container">
              <img :src="imgSrc" class="image"/>
            </div>

            <div class="photo buttons">
              <ion-button @click="onRejectPhoto()" color="danger">
                <ion-icon slot="icon-only" :icon="close"/>
              </ion-button>
            </div>
          </div>
        </div>

      </div>
    </ion-content>
  </ion-page>
</template>


<style scoped lang="scss">
.buttons {
  text-align: center;
}

ion-content {
  --background: transparent;

  &, #content {
    overflow: hidden;
  }
}

ion-title {
  color: white !important;
}

.content-inner {
  --clamped-p-font-size: 1rem;
  --preview-padding: var(--clamped-p-font-size);
  --min-preview-size: calc(100vw - 2 * var(--preview-padding));
  --max-preview-size: 40vh;
  --preview-size: min(var(--min-preview-size), var(--max-preview-size));

  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: space-between;

  #camera {
    #camera-preview.image {
      height: var(--preview-size);
      width: var(--preview-size);
      margin-left: auto;
      margin-right: auto;
      z-index: 1;
      text-align: center;
    }

    p {
      background: rgba(255, 255, 255, 0.75);
      padding: var(--clamped-p-font-size);
    }

    .text {
      text-align: center;
    }

    .text {
      &.danger {
        color: var(--ion-color-danger);
      }
    }
    ion-range{
      padding: 20px;
    }
    .store-to-file{
      width: 100%;
      display: flex;
      justify-content: center;
    }
  }

  #photo{
    margin-bottom: var(--clamped-p-font-size);
    padding: 0 var(--clamped-p-font-size) var(--clamped-p-font-size) var(--clamped-p-font-size);

    .image{
      width: var(--preview-size);
      margin-left: auto;
      margin-right: auto;
    }

    .text{
      text-align: center;
    }
  }

  .buttons {
    display: flex;
    flex-direction: column;
    justify-content: space-around;
    align-items: center;
    margin-top: var(--clamped-p-font-size);
  }

  .image-container {
    height: auto;
    object-fit: cover;
    object-position: center center;
    overflow: hidden;
    display: flex;
    align-items: center;
  }
}

</style>
