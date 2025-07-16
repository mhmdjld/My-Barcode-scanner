<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>Barcode Scanner</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true" class="ion-padding">
      <ion-header collapse="condense">
        <ion-toolbar>
          <ion-title size="large">Barcode Scanner</ion-title>
        </ion-toolbar>
      </ion-header>

      <ion-button expand="block" @click="scanBarcode">
        Barcode scannen
      </ion-button>

      <ion-card v-if="scanResult">
        <ion-card-header>
          <ion-card-title>Scan-Ergebnis</ion-card-title>
        </ion-card-header>
        <ion-card-content>
          <p><strong>Wert:</strong> {{ scanResult.rawValue }}</p>
          <p><strong>Typ:</strong> {{ scanResult.valueType }}</p>
          <p><strong>Format:</strong> {{ scanResult.format }}</p>
        </ion-card-content>
      </ion-card>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import {
  IonButton,
  IonCard,
  IonCardHeader,
  IonCardTitle,
  IonCardContent,
  IonContent,
  IonHeader,
  IonPage,
  IonTitle,
  IonToolbar,
} from '@ionic/vue'

import { BarcodeScanner } from '@capacitor-mlkit/barcode-scanning'
import { ref } from 'vue'

const scanResult = ref<any | null>(null)

const scanBarcode = async () => {
  try {
    const result = await BarcodeScanner.scan()
    if (result?.barcodes?.length > 0) {
      scanResult.value = result.barcodes[0] // Erstes Ergebnis speichern
    }
  } catch (error) {
    console.error('Scan-Fehler:', error)
  }
}
</script>

<style scoped>
#container {
  text-align: center;
  position: absolute;
  left: 0;
  right: 0;
  top: 50%;
  transform: translateY(-50%);
}

#container strong {
  font-size: 20px;
  line-height: 26px;
}

#container p {
  font-size: 16px;
  line-height: 22px;
  color: #8c8c8c;
  margin: 0;
}

#container a {
  text-decoration: none;
}
</style>