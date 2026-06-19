<template>
  <v-card>
    <v-tabs v-model="tab" align-tabs="center" show-arrows>
      <v-tab value="basics">{{ $t('core.basics') }}</v-tab><v-tab value="dns">{{ $t('core.dns') }}</v-tab><v-tab value="route">{{ $t('core.routing') }}</v-tab><v-tab value="json">{{ $t('core.rawJson') }}</v-tab>
    </v-tabs>
    <v-window v-model="tab">
      <v-window-item value="basics"><Basics /></v-window-item>
      <v-window-item value="dns"><Dns /></v-window-item>
      <v-window-item value="route"><Rules /></v-window-item>
      <v-window-item value="json">
        <v-card-text>
          <v-alert v-if="error" type="error" variant="tonal" class="mb-3">{{ error }}</v-alert>
          <v-textarea v-model="raw" rows="24" auto-grow max-rows="36" class="raw-json" spellcheck="false" />
          <div class="d-flex justify-end ga-2"><v-btn variant="tonal" @click="format">{{ $t('core.format') }}</v-btn><v-btn color="primary" :loading="loading" @click="save">{{ $t('core.saveJson') }}</v-btn></div>
        </v-card-text>
      </v-window-item>
    </v-window>
  </v-card>
</template>

<script lang="ts" setup>
import { onMounted, ref } from 'vue'
import HttpUtils from '@/plugins/httputil'
import Basics from './Basics.vue'
import Dns from './Dns.vue'
import Rules from './Rules.vue'
import { i18n } from '@/locales'
const tab = ref('basics'), raw = ref('{}'), error = ref(''), loading = ref(false)
const load = async () => { const response = await HttpUtils.get('api/config'); if (response.success) raw.value = JSON.stringify(response.obj?.config ?? response.obj ?? {}, null, 2) }
const parse = () => { const value = JSON.parse(raw.value); if (!value || Array.isArray(value) || typeof value !== 'object') throw new Error(i18n.global.t('core.rootObjectRequired')); return value }
const format = () => { try { raw.value = JSON.stringify(parse(), null, 2); error.value = '' } catch (exception: any) { error.value = exception.message } }
const save = async () => { try { loading.value = true; const value = parse(); const response = await HttpUtils.post('api/save', { object: 'config', action: 'set', data: JSON.stringify(value) }); if (response.success) { raw.value = JSON.stringify(value, null, 2); error.value = '' } } catch (exception: any) { error.value = exception.message } finally { loading.value = false } }
onMounted(load)
</script>
<style scoped>:deep(.raw-json textarea) { font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace; font-size: 13px; line-height: 1.5; }</style>
