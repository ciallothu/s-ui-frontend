<template>
  <v-row>
    <v-col v-for="group in groups" :key="group.title" cols="12" md="6">
      <v-card height="100%"><v-card-title>{{ group.title }}</v-card-title><v-list>
        <v-list-item v-for="item in group.items" :key="item.title" :prepend-icon="item.icon" :title="item.title" :subtitle="item.subtitle" @click="item.action"><template #append><v-icon icon="mdi-chevron-right" /></template></v-list-item>
      </v-list></v-card>
    </v-col>
  </v-row>
	<input ref="databaseInput" type="file" accept=".db,application/x-sqlite3" hidden @change="restoreDatabase" />
	<v-dialog v-model="resultVisible" max-width="720"><v-card :title="resultTitle"><v-card-text><pre class="tool-result">{{ resultText }}</pre></v-card-text><v-card-actions><v-spacer /><v-btn color="primary" @click="resultVisible = false">{{ $t('actions.close') }}</v-btn></v-card-actions></v-card></v-dialog>
</template>
<script lang="ts" setup>
import { computed, ref } from 'vue'
import HttpUtils from '@/plugins/httputil'
import { i18n } from '@/locales'
const t = (key: string, params?: any) => i18n.global.t(key, params)
const resultVisible = ref(false), resultTitle = ref(t('tools.result')), resultText = ref('')
const databaseInput = ref<HTMLInputElement | null>(null)
const download = (path: string) => window.open(path, '_blank', 'noopener')
const restart = async (target: 'restartSb' | 'restartApp') => { if (window.confirm(t('tools.serviceUnavailableConfirm'))) await HttpUtils.post(`api/${target}`, {}) }
const showResult = (title: string, value: any) => { resultTitle.value = title; resultText.value = typeof value === 'string' ? value : JSON.stringify(value, null, 2); resultVisible.value = true }
const promptTool = async (endpoint: string, label: string) => { const value = window.prompt(label); if (!value) return; const response = await HttpUtils.post(`api/${endpoint}`, { link: value }); if (response.success) showResult(label, response.obj) }
const keypair = async () => { const type = window.prompt(t('tools.keyType'), 'reality'); if (!type) return; const options = window.prompt(t('tools.keyOptions'), '') ?? ''; const response = await HttpUtils.get('api/keypairs', { k: type, o: options }); if (response.success) showResult(t('tools.keyPair', { type }), response.obj) }
const restoreDatabase = async (event: Event) => { const input = event.target as HTMLInputElement; const file = input.files?.[0]; if (!file || !window.confirm(t('tools.restoreDatabaseConfirm'))) { input.value = ''; return }; const form = new FormData(); form.append('db', file); await HttpUtils.post('api/importdb', form); input.value = '' }
const groups = computed(() => [
  { title: t('tools.backupRestore'), items: [
    { title: t('tools.downloadDatabase'), subtitle: t('tools.fullBackup'), icon: 'mdi-database-export-outline', action: () => download('api/getdb') },
	{ title: t('tools.restoreDatabase'), subtitle: t('tools.restoreBackupSubtitle'), icon: 'mdi-database-import-outline', action: () => databaseInput.value?.click() },
    { title: t('tools.downloadSingboxConfig'), subtitle: t('tools.generatedRuntimeJson'), icon: 'mdi-code-json', action: () => download('api/singbox-config') },
  ]},
  { title: t('tools.conversionKeys'), items: [
    { title: t('tools.shareLinkConverter'), subtitle: t('tools.shareLinkSubtitle'), icon: 'mdi-swap-horizontal', action: () => promptTool('linkConvert', t('tools.shareLink')) },
    { title: t('tools.subscriptionConverter'), subtitle: t('tools.subscriptionSubtitle'), icon: 'mdi-playlist-plus', action: () => promptTool('subConvert', t('tools.subscriptionUrl')) },
	{ title: t('tools.generateKeyPairs'), subtitle: t('tools.keyPairsSubtitle'), icon: 'mdi-key-star', action: keypair },
  ]},
  { title: t('tools.serviceActions'), items: [
    { title: t('tools.restartSingbox'), subtitle: t('tools.restartSingboxSubtitle'), icon: 'mdi-restart', action: () => restart('restartSb') },
    { title: t('tools.restartPanel'), subtitle: t('tools.restartPanelSubtitle'), icon: 'mdi-power', action: () => restart('restartApp') },
  ]},
])
</script>
<style scoped>.tool-result { white-space: pre-wrap; overflow-wrap: anywhere; user-select: all; font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace; }</style>
