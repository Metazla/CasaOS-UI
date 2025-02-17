<script>
import debounce from 'lodash/debounce'
import axios from 'axios'
import { ValidationObserver, ValidationProvider } from 'vee-validate'
import '@/plugins/vee-validate'
import VueSlider from 'vue-slider-component'
import 'vue-slider-component/theme/default.css'
import YAML from 'yaml'
import lowerFirst from 'lodash/lowerFirst'
import isNil from 'lodash/isNil'
import { isNumber, isString } from 'lodash/lang'
import cloneDeep from 'lodash/cloneDeep'
import merge from 'lodash/merge'
import { nanoid } from 'nanoid'
import find from 'lodash/find'
import isArray from 'lodash/isArray'
import isObject from 'lodash/isObject'
import InputGroup from '../forms/InputGroup.vue'
import EnvInputGroup from '../forms/EnvInputGroup.vue'
import CommandsInput from '../forms/CommandsInput.vue'
import Ports from '../forms/Ports.vue'
import { ice_i18n } from '@/mixins/base/common-i18n'
import VolumesInputGroup from '@/components/forms/VolumesInputGroup.vue'

const data = [
  'AUDIT_CONTROL',
  'AUDIT_READ',
  'BLOCK_SUSPEND',
  'BPF',
  'CHECKPOINT_RESTORE',
  'DAC_READ_SEARCH',
  'IPC_LOCK',
  'IPC_OWNER',
  'LEASE',
  'LINUX_IMMUTABLE',
  'MAC_ADMIN',
  'MAC_OVERRIDE',
  'NET_ADMIN',
  'NET_BROADCAST',
  'PERFMON',
  'SYS_ADMIN',
  'SYS_BOOT',
  'SYS_MODULE',
  'SYS_NICE',
  'SYS_PACCT',
  'SYS_PTRACE',
  'SYS_RAWIO',
  'SYS_RESOURCE',
  'SYS_TIME',
  'SYS_TTY_CONFIG',
  'SYSLOG',
  'WAKE_ALARM',
]

export default {
  name: 'ComposeConfig',
  components: {
    ValidationObserver,
    ValidationProvider,
    Ports,
    InputGroup,
    VolumesInputGroup,
    EnvInputGroup,
    CommandsInput,
    VueSlider,
  },
  filters: {
    duplexDisplay(val) {
      if (!val) {
        return 256
      }
      return (isNumber(val) && val) || (val && val.replace(/m/i, ''))
    },
  },
  props: {
    state: String,
    totalMemory: {
      type: Number,
      required: true,
    },
    networks: {
      type: Array,
      required: true,
    },
    capArray: {
      type: Array,
      required: true,
    },
    dockerComposeCommands: {
      type: String,
    },
    errInfo: {
      type: Object,
      default() {
        return {
          ports_in_use: { udp: [], tcp: [] },
        }
      },
    },
  },
  emits: ['updateDockerComposeServiceName', 'updateDockerComposeCommands', 'updateMainName'],
  data() {
    return {
      current_service: '',
      baseUrl: '',
      portSelected: null,
      serviceStableVersion: '',
      configData: {
        'services': {
          main_app: {
            image: '',
            mem_reservation: '0',
            network_mode: 'bridge',
            cpu_shares: 10,
            restart: 'unless-stopped',
            volumes: [],
            depends_on: [],
            ports: [],
            environment: [],
            devices: [],
            command: [],
            container_name: '',
            deploy: {
              resources: {
                limits: {
                  memory: '256',
                },
              },
            },
          },
        },
        'x-casaos': {
          hostname: '',
          scheme: 'http',
          index: '/',
          port_map: '',

          author: 'self',
          category: 'self',
          icon: '',
          title: {
            custom: '',
          },
        },
      },
      // error info
      ports_in_use: { udp: [], tcp: [] },

      memory_min: 256,
      // other level_config
      volumes: [],
    }
  },
  computed: {
    firstAppName() {
      return Object.keys(this.configData.services)[0]
    },
    appIcon() {
      return this.configData['x-casaos'].icon
    },
    appendNetworks() {
      return this.networks.map((item) => {
        if (item.driver === 'bridge') {
          if (find(item.networks, ['name', this.firstAppName])) {
            return item
          }
          else {
            return {
              driver: 'bridge',
              networks: [
                {
                  driver: 'bridge',
                  id: nanoid(),
                  name: this.firstAppName,
                },
                ...item.networks,
              ],
            }
          }
        }
        else {
          return item
        }
      })
    },
    markData() {
      const data = []
      let markData = 256
      let i = 0
      while (markData < this.totalMemory) {
        markData = 256 * 2 ** i
        if (markData < this.totalMemory) {
          data.push(markData)
        }
        else {
          data.push(this.totalMemory)
        }
        i++
      }
      return data
    },
  },
  watch: {
    'current_service': {
      handler(val) {
        this.$emit('updateDockerComposeServiceName', val)
        if (this.configData.name) {
          this.$openAPI.appManagement.appStore
            .composeAppServiceStableTag(this.configData.name, this.current_service)
            .then((res) => {
              this.serviceStableVersion = res.data.data.tag
            })
            .catch((e) => {
              this.serviceStableVersion = ''
            })
        }
        else {
          this.serviceStableVersion = ''
        }
      },
      immediate: true,
    },
    // Watch if configData changes
    'configData': {
      handler(val, _newVal) {
        if (this.state === 'install') {
          localStorage.setItem('app_data', JSON.stringify(val))
        }
        this.outputConfigDataCommands(val)
      },
      deep: true,
    },
    // The parent component passes in data
    'dockerComposeCommands': {
      handler(val) {
        if (val != null) {
          this.parseComposeYaml(val)
        }
      },
      immediate: true,
    },
    'configData.name': {
      handler(name = '') {
        this.$emit('updateMainName', name)
      },
      immediate: true,
    },
    'errInfo.ports_in_use': {
      handler(val) {
        this.ports_in_use = val
      },
      immediate: true,
    },
  },

  created() {
    // Set Front-end base url
    this.baseUrl = `${document.domain}`
    // update Service Name.
    this.$emit('updateDockerComposeServiceName', this.firstAppName)
  },

  methods: {
    ice_i18n,
    // ****** migration !!! start !!!
    /**
     * @description: Get remote synchronization information
     * @param {*} function
     * @return {*} void
     */
    getAsyncData: debounce(function (name) {
      if (!name.length) {
        this.data = []
        return
      }
      this.isFetching = true
      axios
        .get(
          `https://hub.docker.com/api/content/v1/products/search?source=community&q=${name}&page=1&page_size=4`,
        )
        .then(({ data }) => {
          this.data = []
          data.summaries.forEach(item => this.data.push(item.name))
        })
        .catch((error) => {
          this.data = []
          throw error
        })
        .finally(() => {
          this.isFetching = false
        })
    }, 500),

    /**
     * @description: Change App icon when image changed
     * @param {string} image
     * @return {*} void
     */
    changeIcon(image) {
      // set this.configData['x-casaos'].icon
      this.configData['x-casaos'].icon = this.getIconFromImage(image)
    },

    /**
     * @description: Get App icon form image
     * @param {*} image
     * @return {*}
     */
    getIconFromImage(image) {
      if (image === '') {
        return ''
      }
      else {
        const appIcon = image.split(':')[0].split('/').pop()
        return `https://icon.casaos.io/main/all/${appIcon}.png`
      }
    },

    /**
     * @description:
     * @param {*} text
     * @return {*}
     */
    getFilteredTags(text) {
      return data.filter((option) => {
        return option.toString().includes(text.toUpperCase())
      })
    },

    async checkStepItem(ref) {
      const isValid = await ref.validate()
      return isValid
    },

    /**
     * @description: Validate form async
     * @return {boolean}
     */
    async checkStep() {
      const promises = []
      for (const servicesKey in this.configData.services) {
        promises.push(this.$refs[`${servicesKey}valida`][0].validate())
      }
      return await Promise.all(promises)
    },

    /**
     * @description: Parse Import Docker Compose Commands
     * @return {boolean}
     */
    async parseComposeYaml(val) {
      try {
        const yaml = YAML.parse(val)
        if (!yaml) {
          return
        }
        // 其他配置
        this.volumes = yaml.volumes || {}

        // set main app name
        this.configData.name = yaml?.name || ''
        this.configData.services = {}
        // 删除掉原默认主应用。
        this.$delete(this.configData.services, 'main_app')
        // this.current_service = yaml["x-casaos"].main;
        this.current_service = Object.keys(yaml.services)[0]
        // 解析 services，并将其赋值到 configData.services中。

        // set top level x-casaos data
        this.configData['x-casaos'] = merge(this.configData['x-casaos'], yaml['x-casaos'])

        for (const serviceKey in yaml.services) {
          this.$set(this.configData.services, serviceKey, this.parseComposeItem(yaml.services[serviceKey]))
        }
      }
      catch (error) {
        console.error(error)
      }
    },

    /*
     * formate for render
     * */
    parseComposeItem(composeServicesItemInput) {
      const composeServicesItem = {}
      // Image
      composeServicesItem.image = composeServicesItemInput.image
      // Envs
      if (composeServicesItemInput.environment) {
        const envArray = Array.isArray(composeServicesItemInput.environment)
          ? composeServicesItemInput.environment
          : Object.entries(composeServicesItemInput.environment)
        composeServicesItem.environment = envArray.map((item) => {
          const ii = typeof item === 'object' ? Array.from(item) : item.split('=')
          return {
            host: ii[1].replace(/"/g, ''),
            container: ii[0],
          }
        })
      }
      else {
        composeServicesItem.environment = []
      }

      // Ports
      /*
      - "3000"
      - "3000-3005"
      - "8000:8000"
      - "9090-9091:8080-8081"
      - "49100:22"
      - "127.0.0.1:8001:8001"
      - "127.0.0.1:5000-5010:5000-5010"
      - "127.0.0.1:5000-5010:5000-5010/udp"
      - "6060:6060/udp"
      test_data : ["3000-3005", "8000:8000", "9090-9091:8080-8081", "49100:22", "127.0.0.1:8001:8001", "127.0.0.1:5000-5010:5000-5010","6060:6060/udp"]
      */
      composeServicesItem.ports = this.makeArray(composeServicesItemInput.ports).map((item) => {
        if (isString(item)) {
          const regex
            // eslint-disable-next-line regexp/no-misleading-capturing-group
            = /(^(?<host>(\d{1,3}\.){1,3}\d{1,3}):?)?(?<published>(\d{1,5})(-(\d{1,5}))?)(:(?<target>(\d{1,5})(-(\d{1,5}))?))?(\/(?<protocol>.*)$)?/
          const match = item.match(regex).groups
          const host_ip = match.host
          const target = Number(match.target?.split('-')?.[0])
          const published = match.published
          const protocol = match.protocol || 'tcp'

          return {
            host_ip,
            target,
            published,
            protocol,
          }
        }
        else if (isObject(item)) {
          item.published = item.published.toString()
          return item
        }
        else {
          return {
            host_ip: '',
            target: `${item}`,
            published: `${item}`,
            protocol: 'tcp',
          }
        }
      })
      isNil(composeServicesItem.ports) && this.$set(composeServicesItem, 'ports', [])

      // Volume
      // https://yeasy.gitbook.io/docker_practice/compose/compose_file#volumes
      composeServicesItem.volumes = this.makeArray(composeServicesItemInput.volumes).map((item) => {
        if (isString(item)) {
          // 1\ replace variable in string for example: ${VOLUME_PATH}:/data
          // this.volumes 可能为空。
          Object.keys(this.volumes || {}).map((key) => {
            item = item.replace(key, this.volumes[key] || '')
          })
          // 2\ split string
          const ii = item.split(':')
          if (ii.length > 1) {
            return {
              type: 'bind',
              target: ii[1],
              source: this.volumeAutoCheck(ii[1], ii[0], lowerFirst(this.configData['x-casaos'].title.en_us)),
            }
          }
          else {
            return {
              type: 'bind',
              target: ii[0],
              source: this.volumeAutoCheck(ii[0], '', lowerFirst(this.configData['x-casaos'].title.en_us)),
            }
          }
        }
        else if (item) {
          // 1\ replace value in object for example: {type: 'bind', source: '${VOLUME_PATH}', target: '/data'}
          Object.keys(this.volumes || {}).map((key) => {
            item.source = item?.source.replace(key, this?.volumes[key] || '')
            // item.target = item?.target
          })

          return item
        }
      })
      isNil(composeServicesItem.volumes) && this.$set(composeServicesItem, 'volumes', [])

      // Devices
      composeServicesItem.devices = this.makeArray(composeServicesItemInput.devices).map((item) => {
        const ii = item.split(':')
        return {
          container: ii[1],
          host: ii[0],
        }
      })
      isNil(composeServicesItem.devices) && this.$set(composeServicesItem, 'devices', [])

      // Network_mode
      const network_mode = composeServicesItemInput?.network_mode
      const networks = composeServicesItemInput?.networks
      if (networks) {
        composeServicesItem.networks = isArray(networks) ? networks : Object.keys(networks)
      }
      else if (network_mode === 'bridge' || network_mode === undefined) {
        composeServicesItem.network_mode = 'bridge'
      }
      else if (network_mode === 'host') {
        composeServicesItem.network_mode = 'host'
      }
      else if (network_mode === 'physical') {
        composeServicesItem.network_mode = 'macvlan'
      }
      else {
        composeServicesItem.network_mode = network_mode
      }

      // hostname
      // configData.host_name = parsedInput.hostname != undefined ? parsedInput.hostname : ""

      // privileged
      // relation issue: https://github.com/IceWhaleTech/CasaOS/issues/1264
      // if privileged is undefined or false, set it to false.
      composeServicesItem.privileged = composeServicesItemInput.privileged === true

      // cap-add
      if (composeServicesItemInput.cap_add !== undefined) {
        composeServicesItem.cap_add = composeServicesItemInput.cap_add
      }
      else {
        composeServicesItem.cap_add = []
      }
      // Restart
      if (composeServicesItemInput.restart !== undefined) {
        composeServicesItem.restart = composeServicesItemInput.restart
      }
      composeServicesItem.restart
        = composeServicesItem.restart === 'no' || !composeServicesItem.restart
          ? 'unless-stopped'
          : composeServicesItem.restart

      // command
      composeServicesItem.command = this.makeArray(composeServicesItemInput.command)

      // container_name
      composeServicesItem.container_name = composeServicesItemInput?.container_name || ''
      composeServicesItem.hostname = composeServicesItemInput?.container_name || ''
      // this.$set(composeServicesItem, "container_name", composeServicesItemInput?.container_name);

      if (
        composeServicesItemInput.cpu_shares === 0
        || composeServicesItemInput.cpu_shares > 99
        || isNil(composeServicesItemInput.cpu_shares)
      ) {
        this.$set(composeServicesItem, 'cpu_shares', 90)
      }
      else {
        this.$set(composeServicesItem, 'cpu_shares', composeServicesItemInput.cpu_shares)
      }

      // 判断是否存在
      const memory = composeServicesItemInput?.deploy?.resources?.limits?.memory
      let newMemory = 0
      if (memory) {
        // 存在的情况下，检测是否有单位
        // 检测没有单位的情况
        if (isNumber(memory - 0) && memory > 0) {
          newMemory = memory / 1024 / 1024
        }
        else if (/M$/i.test(memory)) {
          newMemory = memory.replace(/M/i, '')
        }
        else if (/G$/i.test(memory)) {
          newMemory = memory.replace(/G/i, '') * 1024
        }
      }
      const ob = merge(composeServicesItemInput?.deploy, {
        resources: { limits: { memory: newMemory || this.totalMemory } },
      })
      this.$set(composeServicesItem, 'deploy', ob)

      return composeServicesItem
    },

    volumeAutoCheck(containerPath, hostPath, appName) {
      let finalHostPath = hostPath
      const rootDir = '/DATA'
      const checkArray = [
        {
          keywords: ['config'],
          value: `/AppData/${appName}${containerPath}`,
        },
        {
          keywords: ['tvshows', 'TV', 'tv'],
          value: `/Media/TV Shows`,
        },
        {
          keywords: ['movies', 'Movie', 'movie'],
          value: `/Media/Movies`,
        },
        {
          keywords: ['Music', 'music'],
          value: `/Media/Music`,
        },
        {
          keywords: ['download'],
          value: `/Downloads`,
        },
        {
          keywords: ['pictures', 'photo'],
          value: `/Gallery`,
        },
        {
          keywords: ['media'],
          value: `/Media`,
        },
      ]

      checkArray.forEach((item) => {
        if (
          item.keywords.some((keywordsItem) => {
            return containerPath.includes(keywordsItem)
          })
        ) {
          finalHostPath = rootDir + item.value
        }
      })

      return finalHostPath
    },

    makeArray(foo) {
      const newArray = typeof foo == 'string' ? [foo] : foo
      return newArray === undefined ? [] : newArray
    },

    // ****** migration !!! end !!!

    /*
     * follow this.configData
     * var : this.configData
     * */
    outputConfigDataCommands(val) {
      // configData tans to docker-compose.yml
      let ConfigData = cloneDeep(val)
      for (const servicesKey in val.services) {
        // 数据源
        const service = val.services[servicesKey]
        // 输出结果
        const outputService = ConfigData.services[servicesKey]
        // memory
        outputService.deploy.resources.limits.memory = `${service.deploy.resources.limits.memory}M`
        outputService.devices = service.devices
          .filter((device) => {
            if (device.container || device.host) {
              return true
            }
            return false
          })
          .map((device) => {
            return `${device.host}:${device.container}`
          })
        outputService.environment = service.environment
          .filter((env) => {
            if (env.container || env.host) {
              return true
            }
            return false
          })
          .map((env) => {
            return `${env.container}=${env.host}`
          })
      }
      if (this.dockerComposeCommands) {
        const yaml = YAML.parse(this.dockerComposeCommands)
        Object.keys(yaml.services).forEach((key) => {
          yaml.services[key].ports = []
          yaml.services[key].volumes = []
          yaml.services[key].devices = []
          yaml.services[key].cap_add = []
          yaml.services[key].command = []
          delete yaml.services[key]?.network_mode
          delete yaml.services[key]?.networks
        })

        ConfigData = merge(yaml, ConfigData)
      }
      this.$emit('updateDockerComposeCommands', YAML.stringify(ConfigData))
    },

    showPorts(service) {
      if (service.networks) {
        return true
      }
      if (!service?.network_mode) {
        return true
      }
      // 存在
      if (
        service.network_mode.toLowerCase().includes('macvlan')
        || service.network_mode.includes('host')
      ) {
        return false
      }
      else {
        return true
      }
    },
    showHostPort(service) {
      if (!service.network_mode) {
        return true
      }
      // 存在
      if (service.network_mode.includes('host')) {
        return false
      }
      else {
        return true
      }
    },
    bridgePorts(services) {
      const result = []
      for (const key in services) {
        const service = services[key]
        service.ports.forEach((item) => {
          if (isNumber(item)) {
            result.push(`${item}`)
          }
          else {
            const TEMPORARY_PORT_INFORMATION = item.published?.split(':')
            const published
              = TEMPORARY_PORT_INFORMATION?.length > 1
                ? TEMPORARY_PORT_INFORMATION[1]
                : TEMPORARY_PORT_INFORMATION[0]
            const publishedRange = published?.split('-')
            if (publishedRange?.length > 1) {
              const start = Number.parseInt(publishedRange[0])
              const end = Number.parseInt(publishedRange[1])
              for (let i = start; i <= end; i++) {
                result.push(`${i}`)
              }
            }
            else {
              result.push(`${Number.parseInt(publishedRange[0])}`)
            }
          }
        })
      }
      return result
    },
    // unused
    filteredBeidgePort(service) {
      return this.bridgePorts(service).filter((port) => {
        return port.host.includes(service.port_map)
      })
    },
    // networks or network_mode
    patchNetworkValue(value, service) {
      if (value === 'host' || value === 'bridge') {
        this.$delete(service, 'networks')
        this.$set(service, 'network_mode', value)
      }
      else {
        this.$delete(service, 'network_mode')
        this.$set(service, 'networks', [value])
        const tempNetworks = merge(this.configData?.networks || {}, { [value]: { name: value } })
        this.$set(this.configData, 'networks', tempNetworks)
      }
    },

    getFirstField(image) {
      return image?.split(':')[0]
    },

    getLateField(image) {
      return image?.split(':')[1]
    },
  },
}
</script>

<template>
  <section style="height: calc(100vh - 12.8125rem)">
    <b-tabs class="has-text-full-03" style="height: 100%" :value="firstAppName">
      <b-tab-item v-for="(service, key) in configData.services" :key="key" :label="key" :value="key" @click="current_service = key">
        <ValidationObserver :ref="`${key}valida`">
          <b-field grouped>
            <ValidationProvider v-slot="{ errors, valid }" class="is-flex-grow-1 mr-3" name="Image0" rules="required">
              <b-field :label="`${$t('Docker Image')} *`" :message="$t(errors)" :type="{ 'is-danger': errors[0], 'is-success': valid }" class="mb-3">
                <b-input
                  :key="service.image" :readonly="state === 'update' || serviceStableVersion !== ''" :value="getFirstField(service.image)" :placeholder="$t('e.g.,hello-world:latest')" @input="(V) => changeIcon(V)" @blur="
                    (E) => {
                      return (service.image = service.image.split(':')[1]
                        ? `${E.target._value}:${service.image.split(':')[1]}`
                        : E.target._value);
                    }
                  "
                />
              </b-field>
            </ValidationProvider>

            <b-dropdown aria-role="menu" trap-focus>
              <template #trigger>
                <ValidationProvider v-slot="{ errors, valid }" name="Image1" rules="required">
                  <b-field
                    :label="$t('Tag')"
                    :message="$t(errors) || '123'"
                    :type="{ 'is-danger': errors[0], 'is-success': valid }"
                  >
                    <b-input
                      icon-pack="casa"
                      icon-right="down-outline"
                      class="is-flex-grow-1"
                      :value="getLateField(service.image)"
                      @input="
                        (V) => {
                          service.image = `${service.image.split(':')[0]}:${V}`;
                        }
                      "
                    />
                  </b-field>
                </ValidationProvider>
              </template>
              <b-dropdown-item
                key="latest"
                @click="
                  () => {
                    service.image = `${service.image.split(':')[0]}:latest`;
                  }
                "
              >
                latest
              </b-dropdown-item>
              <b-dropdown-item
                v-show="serviceStableVersion !== '' && firstAppName === key"
                key="stable"
                @click="
                  () => {
                    service.image = `${service.image.split(':')[0]}:${serviceStableVersion}`;
                  }
                "
              >
                stable({{ serviceStableVersion }})
              </b-dropdown-item>
            </b-dropdown>
          </b-field>

          <ValidationProvider v-slot="{ errors, valid }" name="composeAppName" rules="required">
            <b-field :label="`${$t('App Name')} *`" :message="$t(errors)" :type="{ 'is-danger': errors[0], 'is-success': valid }">
              <b-input :placeholder="$t('e.g.,Your App Name')" :value="ice_i18n(configData['x-casaos'].title)" @blur="(E) => (configData['x-casaos'].title.custom = E.target._value)" />
            </b-field>
          </ValidationProvider>

          <b-field v-if="key === firstAppName" :label="$t('Icon URL')">
            <p class="control">
              <span class="button is-static container-icon">
                <b-image :key="appIcon" :src="appIcon" :src-fallback="require('@/assets/img/app/default.svg')" class="is-32x32" ratio="1by1" />
              </span>
            </p>
            <b-input v-model="configData['x-casaos'].icon" :placeholder="$t('Your custom icon URL')" expanded />
          </b-field>

          <b-field v-if="key === firstAppName" label="Web UI">
            <b-select v-model="configData['x-casaos'].scheme">
              <option value="http">
                http://
              </option>
              <option value="https">
                https://
              </option>
            </b-select>
            <b-input v-model="configData['x-casaos'].hostname" :placeholder="baseUrl" expanded />
            <b-autocomplete v-model="configData['x-casaos'].port_map" :data="bridgePorts(configData.services)" :open-on-focus="true" :placeholder="$t('Port')" class="has-colon" field="hostname" @select="(option) => (portSelected = option)" />
            <b-input v-model="configData['x-casaos'].index" :placeholder="`/index.html ${$t('[Optional]')}`" expanded />
          </b-field>

          <b-field :label="$t('Network')">
            <b-select :value="service.network_mode || service?.networks?.[0]" expanded placeholder="Select" @input="(v) => patchNetworkValue(v, service)">
              <optgroup v-for="net in appendNetworks" :key="net.driver" :label="net.driver">
                <option v-for="(option, index) in net.networks" :key="option.name + index" :value="option.name">
                  {{ option.name }}
                </option>
              </optgroup>
            </b-select>
          </b-field>

          <Ports v-if="showPorts(service)" v-model="service.ports" :ports_in_use="ports_in_use" :show-host-post="showHostPort(service)" />

          <VolumesInputGroup v-model="service.volumes" :label="$t('Volumes')" :message="$t('No volumes now, click “+” to add one.')" type="volume" />
          <EnvInputGroup v-model="service.environment" :label="$t('Environment Variables')" :message="$t('No environment variables now, click “+” to add one.')" />
          <InputGroup :devices="service.devices" :label="$t('Devices')" :message="$t('No devices now, click “+” to add one.')" type="device" />
          <CommandsInput v-model="service.command" :label="$t('Container Command')" :message="$t('No commands now, click “+” to add one.')" />

          <b-field :label="$t('Privileged')">
            <b-switch v-model="service.privileged" />
          </b-field>

          <b-field :label="$t('Memory Limit')" class="mb-5">
            <VueSlider :max="totalMemory" :min="memory_min" class="mx-2" :marks="true" :data="markData" :value="service.deploy.resources.limits.memory | duplexDisplay" @change="(v) => (service.deploy.resources.limits.memory = v)" />
          </b-field>

          <b-field :label="$t('CPU Shares')">
            <b-select v-model="service.cpu_shares" :placeholder="$t('Select')" expanded>
              <option :value="10">
                {{ $t("Low") }}
              </option>
              <option :value="50">
                {{ $t("Medium") }}
              </option>
              <option :value="90">
                {{ $t("High") }}
              </option>
            </b-select>
          </b-field>

          <b-field :label="$t('Restart Policy')">
            <b-select v-model="service.restart" :placeholder="$t('Select')" expanded>
              <option value="on-failure">
                on-failure
              </option>
              <option value="always">
                always
              </option>
              <option value="unless-stopped">
                unless-stopped
              </option>
            </b-select>
          </b-field>

          <b-field :label="$t('Container Capabilities (cap-add)')">
            <b-taginput ref="taginput" v-model="service.cap_add" :allow-new="false" :data="capArray" :open-on-focus="false" autocomplete @typing="getFilteredTags">
              <template slot-scope="props">
                {{ props.option }}
              </template>
              <template #empty>
                There are no items
              </template>
              <template #portSelected="props">
                <b-tag v-for="(tag, index) in props.tags" :key="index" :tabstop="false" closable @close="$refs.taginput.removeTag(index, $event)">
                  {{ tag }}
                </b-tag>
              </template>
            </b-taginput>
          </b-field>

          <ValidationProvider v-slot="{ errors, valid }" name="Name" rules="ContainerName">
            <b-field :label="$t('Container Name')" :message="$t(errors)" :type="{ 'is-danger': errors[0], 'is-success': valid && service.container_name }">
              <b-input v-model="service.container_name" :placeholder="$t('Name of app container')" value="" />
            </b-field>
          </ValidationProvider>
        </ValidationObserver>
      </b-tab-item>
    </b-tabs>
  </section>
</template>

<style lang="scss" scoped>
.b-tabs {
  margin-bottom: 0;

  ::v-deep .tabs {
    padding: 0 1.5rem;
    border-bottom: 1px solid hsla(208, 16%, 94%, 1);

    ul {
      border: 0;

      li {
        font-size: 0.875rem;

        a {
          padding-top: 0.75rem;
          padding-bottom: 5px;
        }
      }
    }
  }

  /* 设置滚动条样式 */
  ::v-deep .tab-content {
    padding-right: 1.5rem;
    padding-left: 1.5rem;
    height: calc(100% - 2.75rem);
    overflow-y: overlay;
    overflow-x: hidden;

    .container-icon {
      height: 2.5rem;
    }
  }
}

::v-deep .vue-slider-mark-step {
  width: 100%;
  height: 100%;
  border-radius: 50%;
  -webkit-box-shadow: 0 0 0 2px #ccc;
  box-shadow: 0 0 0 2px #ccc;
  background-color: #fff;
}

::v-deep .vue-slider-mark-label {
  font-size: 0.75rem;
}

::v-deep .vue-slider-dot-tooltip-text {
  font-size: 0.75rem;
}
</style>
