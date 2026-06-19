<template>
  <v-card>
    <v-tabs v-model="tab" align-tabs="center">
      <v-tab value="settings">{{ $t('settingsTools.settings') }}</v-tab>
      <v-tab value="tools">{{ $t('settingsTools.tools') }}</v-tab>
    </v-tabs>
    <v-window v-model="tab">
      <v-window-item value="settings"><Settings /></v-window-item>
      <v-window-item value="tools"><Tools /></v-window-item>
    </v-window>
  </v-card>
</template>

<script lang="ts" setup>
import { ref } from 'vue'
import Settings from './Settings.vue'
import Tools from './Tools.vue'

const tab = ref('settings')
</script>
