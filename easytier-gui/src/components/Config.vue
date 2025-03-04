<script setup lang="ts">
import InputGroup from 'primevue/inputgroup'
import InputGroupAddon from 'primevue/inputgroupaddon'
import { getOsHostname } from '~/composables/network'
import { NetworkingMethod } from '~/types/network'
const { t } = useI18n()

const props = defineProps<{
  configInvalid?: boolean
  instanceId?: string
}>()

defineEmits(['runNetwork'])

const networking_methods = ref([
  { value: NetworkingMethod.PublicServer, label: t('public_server') },
  { value: NetworkingMethod.Manual, label: t('manual') },
  { value: NetworkingMethod.Standalone, label: t('standalone') },
])

const networkStore = useNetworkStore()
const curNetwork = computed(() => {
  if (props.instanceId) {
    // console.log('instanceId', props.instanceId)
    const c = networkStore.networkList.find(n => n.instance_id === props.instanceId)
    if (c !== undefined)
      return c
  }

  return networkStore.curNetwork
})

const presetPublicServers = [
  'tcp://easytier.public.kkrainbow.top:11010',
]

function validateHostname() {
  if (curNetwork.value.hostname) {
    // eslint no-useless-escape
    let name = curNetwork.value.hostname!.replaceAll(/[^\u4E00-\u9FA5a-zA-Z0-9\-]*/g, '')
    if (name.length > 32)
      name = name.substring(0, 32)

    if (curNetwork.value.hostname !== name)
      curNetwork.value.hostname = name
  }
}

const osHostname = ref<string>('')

onMounted(async () => {
  osHostname.value = await getOsHostname()
})
</script>

<template>
  <div class="flex flex-column h-full">
    <div class="flex flex-column">
      <div class="w-10/12 self-center ">
        <Message severity="warn">
          {{ t('dhcp_experimental_warning') }}
        </Message>
      </div>
      <div class="w-10/12 self-center ">
        <Panel :header="t('basic_settings')">
          <div class="flex flex-column gap-y-2">
            <div class="flex flex-row gap-x-9 flex-wrap">
              <div class="flex flex-column gap-2 basis-5/12 grow">
                <div class="flex align-items-center" for="virtual_ip">
                  <label class="mr-2"> {{ t('virtual_ipv4') }} </label>
                  <Checkbox v-model="curNetwork.dhcp" input-id="virtual_ip_auto" :binary="true" />

                  <label for="virtual_ip_auto" class="ml-2">
                    {{ t('virtual_ipv4_dhcp') }}
                  </label>
                </div>
                <InputGroup>
                  <InputText
                    id="virtual_ip" v-model="curNetwork.virtual_ipv4" :disabled="curNetwork.dhcp"
                    aria-describedby="virtual_ipv4-help"
                  />
                  <InputGroupAddon>
                    <span>/24</span>
                  </InputGroupAddon>
                </InputGroup>
              </div>
            </div>

            <div class="flex flex-row gap-x-9 flex-wrap">
              <div class="flex flex-column gap-2 basis-5/12 grow">
                <label for="network_name">{{ t('network_name') }}</label>
                <InputText id="network_name" v-model="curNetwork.network_name" aria-describedby="network_name-help" />
              </div>
              <div class="flex flex-column gap-2 basis-5/12 grow">
                <label for="network_secret">{{ t('network_secret') }}</label>
                <InputText
                  id="network_secret" v-model="curNetwork.network_secret"
                  aria-describedby=" network_secret-help"
                />
              </div>
            </div>

            <div class="flex flex-row gap-x-9 flex-wrap">
              <div class="flex flex-column gap-2 basis-5/12 grow">
                <label for="nm">{{ t('networking_method') }}</label>
                <div class="items-center flex flex-row p-fluid gap-x-1">
                  <Dropdown
                    v-model="curNetwork.networking_method" :options="networking_methods" option-label="label"
                    option-value="value" placeholder="Select Method" class=""
                  />
                  <Chips
                    v-if="curNetwork.networking_method === NetworkingMethod.Manual" id="chips"
                    v-model="curNetwork.peer_urls" :placeholder="t('chips_placeholder', ['tcp://8.8.8.8:11010'])"
                    separator=" " class="grow"
                  />

                  <Dropdown
                    v-if="curNetwork.networking_method === NetworkingMethod.PublicServer"
                    v-model="curNetwork.public_server_url" :editable="true" class="grow"
                    :options="presetPublicServers"
                  />
                </div>
              </div>
            </div>
          </div>
        </Panel>

        <Divider />

        <Panel :header="t('advanced_settings')" toggleable collapsed>
          <div class="flex flex-column gap-y-2">
            <div class="flex flex-row gap-x-9 flex-wrap">
              <div class="flex flex-column gap-2 basis-5/12 grow">
                <label for="hostname">{{ t('hostname') }}</label>
                <InputText
                  id="hostname" v-model="curNetwork.hostname" aria-describedby="hostname-help" :format="true"
                  :placeholder="t('hostname_placeholder', [osHostname])" @blur="validateHostname"
                />
              </div>
            </div>

            <div class="flex flex-row gap-x-9 flex-wrap w-full">
              <div class="flex flex-column gap-2 grow p-fluid">
                <label for="username">{{ t('proxy_cidrs') }}</label>
                <Chips
                  id="chips" v-model="curNetwork.proxy_cidrs"
                  :placeholder="t('chips_placeholder', ['10.0.0.0/24'])" separator=" " class="w-full"
                />
              </div>
            </div>

            <div class="flex flex-row gap-x-9 flex-wrap ">
              <div class="flex flex-column gap-2 grow">
                <label for="username">VPN Portal</label>
                <div class="items-center flex flex-row gap-x-4">
                  <ToggleButton
                    v-model="curNetwork.enable_vpn_portal" on-icon="pi pi-check" off-icon="pi pi-times"
                    :on-label="t('off_text')" :off-label="t('on_text')"
                  />
                  <div v-if="curNetwork.enable_vpn_portal" class="grow">
                    <InputGroup>
                      <InputText
                        v-model="curNetwork.vpn_portal_client_network_addr"
                        :placeholder="t('vpn_portal_client_network')"
                      />
                      <InputGroupAddon>
                        <span>/{{ curNetwork.vpn_portal_client_network_len }}</span>
                      </InputGroupAddon>
                    </InputGroup>
                  </div>
                  <InputNumber
                    v-if="curNetwork.enable_vpn_portal" v-model="curNetwork.vpn_portal_listen_port"
                    :placeholder="t('vpn_portal_listen_port')" class="" :format="false" :min="0" :max="65535"
                  />
                </div>
              </div>
            </div>

            <div class="flex flex-row gap-x-9 flex-wrap">
              <div class="flex flex-column gap-2 grow p-fluid">
                <label for="listener_urls">{{ t('listener_urls') }}</label>
                <Chips
                  id="listener_urls" v-model="curNetwork.listener_urls"
                  :placeholder="t('chips_placeholder', ['tcp://1.1.1.1:11010'])" separator=" " class="w-full"
                />
              </div>
            </div>

            <div class="flex flex-row gap-x-9 flex-wrap">
              <div class="flex flex-column gap-2 basis-5/12 grow">
                <label for="rpc_port">{{ t('rpc_port') }}</label>
                <InputNumber
                  id="rpc_port" v-model="curNetwork.rpc_port" aria-describedby="username-help"
                  :format="false" :min="0" :max="65535"
                />
              </div>
            </div>
          </div>
        </Panel>

        <div class="flex pt-4 justify-content-center">
          <Button
            :label="t('run_network')" icon="pi pi-arrow-right" icon-pos="right" :disabled="configInvalid"
            @click="$emit('runNetwork', curNetwork)"
          />
        </div>
      </div>
    </div>
  </div>
</template>
