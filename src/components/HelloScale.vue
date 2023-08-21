<script lang="ts" setup>
import { ref } from 'vue';
import { pausableWatch, useBluetooth } from '@vueuse/core'
// import type { BluetoothRemoteGATTCharacteristic } from 'web-bluetooth'


/*
*  
*  
* 
* 
*/
// const SCALE_MAC = '03:B3:EC:8D:F1:21'
const SERVICE_UUID =  0xFFB0 //'0000fb0-0000-1000-8000-00805f9b34fb' //'0xFFB0'
const NOTFIY_UUID =  0xFFB2 // 
const WRITE_UUID =  0xFFB1 // Write 0x01 to characteristic to start notifications
const scaleUnit = ref<number | undefined>(0)

const {
  isSupported,
  isConnected,
  device,
  requestDevice,
  server,
  error,
} = useBluetooth({
  acceptAllDevices: true,
  optionalServices: [
    SERVICE_UUID,
  ],
})


const isGettingBatteryLevels = ref(false)

async function getScale() {
  isGettingBatteryLevels.value = true

  const scaleService = await server.value?.getPrimaryService(SERVICE_UUID)
  console.log('Service', scaleService);
  console.log('Service Characteristics', scaleService?.getCharacteristics());


  // Subscribe to notifications
  const scaleCharacteristic = await scaleService?.getCharacteristic(
    NOTFIY_UUID,
  )
  await scaleCharacteristic?.startNotifications();
  
  const writeCharacteristic = await scaleService?.getCharacteristic(
    WRITE_UUID,
  )
  let arrayBuffer = new Uint8Array([0x01]);
  await writeCharacteristic.writeValue(arrayBuffer);

  //const descriptor = await  writeCharacteristic?.getDescriptor(WRITE_UUID);
  //let arrayBuffer = new Uint8Array([0x01]);

  //await descriptor?.writeValue(arrayBuffer).then(() => console.log('then')).catch((e) => console.log(e));
  // 0,...,7
  // 256*buffer[3] + buffer[4] = number
  // if (buffer[5]) -number else  
  // batteryPercent.value = event.target.value.getUint8(0)
  // Listen to when characteristic value changes on `characteristicvaluechanged` event:
  scaleCharacteristic?.addEventListener('characteristicvaluechanged', (event) => {
    //
    if (event?.target?.value?.getUint8(5) ?? false) {
      scaleUnit.value =  -1 * (event?.target?.value?.getUint8(3) ?? 0)  * 256 + -1 * (event?.target?.value?.getUint8(4) ?? 0);
    } else {
      scaleUnit.value = (event?.target?.value?.getUint8(3) ?? 0)  * 256 + (event?.target?.value?.getUint8(4) ?? 0);
    }
    console.log(scaleUnit.value)
  })

}

const { stop } = pausableWatch(isConnected, (newIsConnected) => {
  if (!newIsConnected || !server.value || isGettingBatteryLevels.value)
    return
  // Attempt to get the battery levels of the device:
  getScale()
  // We only want to run this on the initial connection, as we will use an event listener to handle updates:
  stop()
})
</script>

<template>
  <div class="grid grid-cols-1 gap-x-4 gap-y-4">
    <div>{{ isSupported ? 'Bluetooth Web API Supported' : 'Your browser does not support the Bluetooth Web API' }}</div>

    <div v-if="isSupported">
      <button @click="requestDevice()">
        Request Bluetooth Device
      </button>
    </div>
    <div>
      {{  scaleUnit }}
    </div>

    <div v-if="device">
      <p>Device Name: {{ device.name }}</p>
    </div>

    <div v-if="isConnected" class="bg-green-500 text-white p-3 rounded-md">
      <p>Connected</p>
    </div>

    <div v-if="!isConnected" class="bg-orange-800 text-white p-3 rounded-md">
      <p>Not Connected</p>
    </div>

    <div v-if="error">
      <div>Errors:</div>
      <pre>
      <code class="block p-5 whitespace-pre">{{ error }}</code>
    </pre>
    </div>
  </div>
</template>
