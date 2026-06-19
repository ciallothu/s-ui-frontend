<template>
  <v-card class="mt-4" :loading="loading">
    <v-card-title class="d-flex align-center ga-2">
      <v-icon icon="mdi-shield-account-outline" />
      {{ $t('security.title') }}
      <v-spacer />
      <v-btn icon="mdi-refresh" variant="text" @click="load" />
    </v-card-title>
    <v-divider />
    <v-list lines="two">
      <v-list-item prepend-icon="mdi-two-factor-authentication" :title="$t('security.totp')" :subtitle="security.totpEnabled ? $t('security.enabled') : $t('security.disabled')">
        <template #append>
          <v-btn v-if="security.totpEnabled" color="warning" variant="tonal" @click="disableDialog = true">{{ $t('actions.disable') }}</v-btn>
          <v-btn v-else color="primary" variant="tonal" @click="beginTotp">{{ $t('actions.enable') }}</v-btn>
        </template>
      </v-list-item>
      <v-divider />
      <v-list-item prepend-icon="mdi-passkey" :title="$t('security.passkeys')" :subtitle="methods.passkey ? $t('security.passkeyEnabled') : $t('security.passkeyDisabledHint')">
        <template #append>
          <v-btn color="primary" variant="tonal" :disabled="!methods.passkey || !passkeySupported" @click="addPasskey">{{ $t('security.addPasskey') }}</v-btn>
        </template>
      </v-list-item>
      <v-list-item v-for="passkey in passkeys" :key="passkey.id" class="ps-12" :title="passkey.name || $t('security.passkey')" :subtitle="formatDate(passkey.createdAt)">
        <template #append>
          <v-btn icon="mdi-pencil-outline" variant="text" @click="openRename(passkey)" />
          <v-btn icon="mdi-delete-outline" color="error" variant="text" @click="deletePasskey(passkey.id)" />
        </template>
      </v-list-item>
      <v-divider />
      <v-list-item prepend-icon="mdi-badge-account-horizontal-outline" :title="$t('security.oidc')" :subtitle="methods.oidc ? $t('security.oidcEnabled') : $t('security.disabled')" />
    </v-list>
  </v-card>

  <v-dialog v-model="totpDialog" max-width="520">
    <v-card :title="$t('security.enableTotp')">
      <v-card-text class="text-center">
        <p class="mb-4">{{ $t('security.scanTotp') }}</p>
        <QrcodeVue v-if="totp.uri" :value="totp.uri" :size="220" level="M" />
        <v-text-field class="mt-4" v-model="totpCode" :label="$t('security.code6')" autocomplete="one-time-code" />
        <v-expansion-panels variant="accordion">
          <v-expansion-panel :title="$t('security.manualKey')" :text="totp.secret" />
        </v-expansion-panels>
      </v-card-text>
      <v-card-actions><v-spacer /><v-btn @click="totpDialog = false">{{ $t('no') }}</v-btn><v-btn color="primary" @click="enableTotp">{{ $t('actions.enable') }}</v-btn></v-card-actions>
    </v-card>
  </v-dialog>

  <v-dialog v-model="recoveryDialog" max-width="520" persistent>
    <v-card :title="$t('security.saveRecoveryCodes')">
      <v-card-text><p class="mb-3">{{ $t('security.recoveryHint') }}</p><pre class="recovery-codes">{{ recoveryCodes.join('\n') }}</pre></v-card-text>
      <v-card-actions><v-spacer /><v-btn color="primary" @click="recoveryDialog = false">{{ $t('security.savedRecoveryCodes') }}</v-btn></v-card-actions>
    </v-card>
  </v-dialog>

  <v-dialog v-model="disableDialog" max-width="460">
    <v-card :title="$t('security.disableTotp')">
      <v-card-text>
        <v-text-field v-model="password" :label="$t('security.currentPassword')" type="password" />
        <v-text-field v-model="disableCode" :label="$t('security.totpOrRecovery')" />
      </v-card-text>
      <v-card-actions><v-spacer /><v-btn @click="disableDialog = false">{{ $t('no') }}</v-btn><v-btn color="warning" @click="disableTotp">{{ $t('actions.disable') }}</v-btn></v-card-actions>
    </v-card>
  </v-dialog>

  <v-dialog v-model="renameDialog" max-width="420">
    <v-card :title="$t('security.renamePasskey')">
      <v-card-text>
        <v-text-field v-model="renameName" :label="$t('security.passkeyName')" autofocus />
      </v-card-text>
      <v-card-actions><v-spacer /><v-btn @click="renameDialog = false">{{ $t('actions.close') }}</v-btn><v-btn color="primary" @click="renamePasskey">{{ $t('actions.save') }}</v-btn></v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script lang="ts" setup>
import { computed, onMounted, ref } from 'vue'
import QrcodeVue from 'qrcode.vue'
import { push } from 'notivue'
import HttpUtils from '@/plugins/httputil'
import { registerPasskey } from '@/plugins/webauthn'
import { i18n } from '@/locales'

const loading = ref(false)
const security = ref<any>({ totpEnabled: false, passkeys: [], methods: {} })
const totp = ref<any>({})
const totpCode = ref('')
const recoveryCodes = ref<string[]>([])
const password = ref('')
const disableCode = ref('')
const totpDialog = ref(false)
const recoveryDialog = ref(false)
const disableDialog = ref(false)
const renameDialog = ref(false)
const renameId = ref<number | null>(null)
const renameName = ref('')
const methods = computed(() => security.value.methods ?? {})
const passkeys = computed(() => security.value.passkeys ?? [])
const passkeySupported = typeof window !== 'undefined' && !!window.PublicKeyCredential

const load = async () => {
  loading.value = true
  const response = await HttpUtils.get('api/security')
  if (response.success) security.value = response.obj ?? security.value
  loading.value = false
}
const beginTotp = async () => {
  const response = await HttpUtils.get('api/totp-begin')
  if (response.success) {
    totp.value = response.obj
    totpCode.value = ''
    totpDialog.value = true
  }
}
const enableTotp = async () => {
  const response = await HttpUtils.post('api/totp-enable', { code: totpCode.value })
  if (response.success) {
    recoveryCodes.value = response.obj?.recoveryCodes ?? []
    totpDialog.value = false
    recoveryDialog.value = true
    await load()
  }
}
const disableTotp = async () => {
  const response = await HttpUtils.post('api/totp-disable', { password: password.value, code: disableCode.value })
  if (response.success) {
    disableDialog.value = false
    password.value = ''
    disableCode.value = ''
    await load()
  }
}
const addPasskey = async () => {
  try {
    const name = await registerPasskey()
    push.success({ message: i18n.global.t('security.passkeyRegistered', { name }) })
    await load()
  } catch (error: any) {
    push.error({ message: error?.message ?? String(error) })
  }
}
const deletePasskey = async (id: number) => {
  if (!window.confirm(i18n.global.t('security.deletePasskey'))) return
  const response = await HttpUtils.post('api/passkey-delete', { id })
  if (response.success) await load()
}
const openRename = (passkey: any) => {
  renameId.value = passkey.id
  renameName.value = passkey.name || i18n.global.t('security.passkey')
  renameDialog.value = true
}
const renamePasskey = async () => {
  if (!renameId.value) return
  const response = await HttpUtils.post('api/passkey-rename', { id: renameId.value, name: renameName.value })
  if (response.success) {
    renameDialog.value = false
    await load()
  }
}
const formatDate = (value: number) => value ? new Date(value * 1000).toLocaleString() : i18n.global.t('security.neverUsed')

onMounted(load)
</script>

<style scoped>
.recovery-codes { padding: 16px; border-radius: 8px; background: rgb(var(--v-theme-surface-variant)); line-height: 1.7; user-select: all; }
</style>
