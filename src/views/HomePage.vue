<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>Barcode Scanner</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content class="ion-padding">
      <!-- Loading Spinner -->
      <ion-loading :is-open="isLoading" message="Loading" />

      <!-- Kopieren-Bestätigung -->
      <ion-toast
        :is-open="copyToast"
        message="In Zwischenablage kopiert"
        duration="2000"
        @did-dismiss="copyToast = false"
      />

      <!-- Buttons zum Scanne -->
      <ion-button expand="block" @click="scanFromCamera" :disabled="isLoading">
        <ion-icon slot="start" :icon="cameraOutline" />
        Kamera-Scan
      </ion-button>
      <!-- Buttons zum Auswählen eines Bildes aus der Galerie -->
      <ion-button expand="block" @click="scanFromGallery" :disabled="isLoading">
        <ion-icon slot="start" :icon="imagesOutline" />
        Bild auswählen
      </ion-button>

      <!-- Liste -->
      <transition-group name="barcode" tag="ion-list">
        <ion-item
          class="barcode-item"
          v-for="(barcode, index) in barcodes"
          :key="barcode.id || index"
        >
          <ion-label>
            <h2>
              <template v-if="barcode.valueType === 'URL'">
                {{ extractDomain(barcode.displayValue || barcode.rawValue) }}
              </template>
              <template v-else>
                {{ barcode.displayValue || barcode.rawValue }}
              </template>
            </h2>
            <p v-if="barcode.displayValue && barcode.valueType === 'URL'">
              {{ barcode.displayValue }}
            </p>
            <p>Format: {{ barcode.format }}</p>
            <p>Typ: {{ barcode.valueType }}</p>
          </ion-label>

          <ion-buttons slot="end">
            <!-- Button zum Teilen -->
            <ion-button fill="clear" @click="shareBarcode(barcode)">
              <ion-icon :icon="shareSocialOutline" />
            </ion-button>
            <!-- Button zum Kopieren -->
            <ion-button fill="clear" @click="copyBarcode(barcode)">
              <ion-icon :icon="copyOutline" />
            </ion-button>
            <!-- Link im Browser öffnen -->
            <ion-button
              v-if="barcode.valueType === 'URL'"
              fill="clear"
              @click="openInBrowser(barcode)"
            >
              <ion-icon :icon="globeOutline" />
            </ion-button>
            <!-- Anrufen -->
            <ion-button
              v-if="barcode.valueType === 'PHONE'"
              fill="clear"
              @click="callPhone(barcode)"
            >
              <ion-icon :icon="callOutline" />
            </ion-button>
            <!-- Button zum löschen -->
            <ion-button fill="clear" @click="deleteBarcode(index)">
              <ion-icon :icon="trashOutline" color="danger" />
            </ion-button>
          </ion-buttons>
        </ion-item>
      </transition-group>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import {
  IonPage,
  IonHeader,
  IonToolbar,
  IonTitle,
  IonContent,
  IonList,
  IonItem,
  IonLabel,
  IonButtons,
  IonButton,
  IonIcon,
  IonLoading,
  IonToast,
} from '@ionic/vue'

// Ion Icons
import {
  cameraOutline,
  imagesOutline,
  shareSocialOutline,
  copyOutline,
  globeOutline,
  callOutline,
  trashOutline,
} from 'ionicons/icons'

import { BarcodeScanner } from '@capacitor-mlkit/barcode-scanning'
import { FilePicker } from '@capawesome/capacitor-file-picker'
import { Share } from '@capacitor/share'
import { Clipboard } from '@capacitor/clipboard'
import { Browser } from '@capacitor/browser'
import { Preferences } from '@capacitor/preferences'
import { ref, onMounted } from 'vue'

// Generieren einer ID für Animations-Key
let nextId = 1
const barcodes = ref<any[]>([])
const isLoading = ref(false)
const copyToast = ref(false)

const loadBarcodes = async () => {
  const { value } = await Preferences.get({ key: 'barcodes' })
  if (value) {
    const saved = JSON.parse(value)
    barcodes.value = saved.map((b: any) => ({ ...b, id: nextId++ }))
  }
}

const saveBarcodes = async () => {
  await Preferences.set({ key: 'barcodes', value: JSON.stringify(barcodes.value) })
}

onMounted(loadBarcodes)

const ensureCameraPermission = async () => {
  const status = await BarcodeScanner.checkPermissions()
  if (status.camera !== 'granted') {
    throw new Error('Kamera-Berechtigung verweigert')
  }
}

const scanFromCamera = async () => {
  try {
    await ensureCameraPermission()
    isLoading.value = true
    const result = await BarcodeScanner.scan()
    if (result?.barcodes?.length) {
      const items = result.barcodes.map((b: any) => ({ ...b, id: nextId++ }))
      barcodes.value.push(...items)
      await saveBarcodes()
    } else {
      window.alert('Kein Barcode gefunden')
    }
  } catch (e: any) {
    console.error(e)
    window.alert(e.message || 'Scan fehlgeschlagen')
  } finally {
    isLoading.value = false
  }
}

const scanFromGallery = async () => {
  try {
    isLoading.value = true
    const result = await FilePicker.pickImages()
    if (!result.files.length) {
      window.alert('Kein Bild ausgewählt')
    } else {
      const file = result.files[0]
      if (!file.path) throw new Error('Ungültiger Dateipfad')
      const imageScan = await BarcodeScanner.readBarcodesFromImage({ path: file.path })
      if (imageScan?.barcodes?.length) {
        const items = imageScan.barcodes.map((b: any) => ({ ...b, id: nextId++ }))
        barcodes.value.push(...items)
        await saveBarcodes()
      } else {
        window.alert('Kein Barcode gefunden')
      }
    }
  } catch (e: any) {
    console.error(e)
    window.alert(e.message || 'Scan fehlgeschlagen')
  } finally {
    isLoading.value = false
  }
}

const deleteBarcode = async (index: number) => {
  barcodes.value.splice(index, 1)
  await saveBarcodes()
}

const shareBarcode = async (barcode: any) => {
  const text = barcode.displayValue ?? barcode.rawValue
  await Share.share({ text })
}

const copyBarcode = async (barcode: any) => {
  const text = barcode.displayValue ?? barcode.rawValue
  await Clipboard.write({ string: text })
  copyToast.value = true
}

const openInBrowser = async (barcode: any) => {
  const url = barcode.displayValue ?? barcode.rawValue
  await Browser.open({ url })
}

const callPhone = (barcode: any) => {
  const number = barcode.displayValue ?? barcode.rawValue
  window.open(`tel:${number}`, '_system')
}

const extractDomain = (url: string) => {
  try {
    return new URL(url).hostname
  } catch {
    return url
  }
}
</script>

<style scoped>
.barcode-item {
  position: relative;
}
.barcode-item ion-buttons {
  position: absolute;
  right: 16px;
  bottom: 8px;
}
ion-button {
  margin-top: 10px;
}

/* Animation für Einfügen/Entfernen */
.barcode-enter-active,
.barcode-leave-active {
  transition: transform 0.3s ease, opacity 0.3s ease;
  transform-origin: center center;
}
.barcode-enter-from {
  opacity: 0;
  transform: scale(0.5);
}
.barcode-enter-to {
  opacity: 1;
  transform: scale(1);
}
.barcode-leave-from {
  opacity: 1;
  transform: scale(1);
}
.barcode-leave-to {
  opacity: 0;
  transform: scale(0.5);
}
</style>
