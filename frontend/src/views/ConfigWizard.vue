<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, reactive, ref } from 'vue'
import {
  ArrowLeftIcon,
  ArrowRightIcon,
  CheckCircleIcon,
  ClipboardDocumentListIcon,
  Cog6ToothIcon,
  MusicalNoteIcon,
  PlusIcon,
  ServerStackIcon,
  ShieldCheckIcon,
  TrashIcon,
  ChatBubbleLeftRightIcon,
} from '@heroicons/vue/24/outline'
import UiButton from '@/components/ui/UiButton.vue'
import UiCard from '@/components/ui/UiCard.vue'
import UiListInput from '@/components/ui/UiListInput.vue'
import { useAuthStore } from '@/stores/auth'
import { logger } from '@/utils/logger'
import { onBeforeRouteLeave } from 'vue-router'

type JsonRecord = Record<string, any>
type StepId = 'general' | 'emby' | 'telegram' | 'subsonic' | 'network' | 'review'

type EmbyAccountDraft = {
  raw: JsonRecord
  url: string
  username: string
  password: string
  name: string
  enabled: boolean
  use_proxy: boolean
  allow_stream: boolean
  cf_challenge: boolean
  allow_multiple: boolean
  play_id: string
  time_start: number
  time_end: number
  interval_days: string
  time_range: string
  useragent: string
  client: string
  client_version: string
  device: string
  device_id: string
  continuous: boolean
  advanced: boolean
}

type SubsonicAccountDraft = {
  raw: JsonRecord
  url: string
  username: string
  password: string
  name: string
  enabled: boolean
  use_proxy: boolean
  time_start: number
  time_end: number
  interval_days: string
  time_range: string
  useragent: string
  client: string
  client_version: string
  device: string
  device_id: string
  advanced: boolean
}

type TelegramAccountDraft = {
  raw: JsonRecord
  phone: string
  enabled: boolean
  checkiner: boolean
  monitor: boolean
  messager: boolean
  registrar: boolean
  api_id: string
  api_hash: string
  session: string
  site_checkiner: string[]
  site_monitor: string[]
  site_messager: string[]
  site_registrar: string[]
  checkiner_override: boolean
  checkiner_interval_days: string
  checkiner_time_range: string
  checkiner_timeout: number
  checkiner_retries: number
  checkiner_concurrency: number
  checkiner_random_start: number
  registrar_override: boolean
  registrar_concurrency: number
  advanced: boolean
}

const auth = useAuthStore()

const steps: Array<{ id: StepId; label: string; hint: string; icon: typeof Cog6ToothIcon }> = [
  { id: 'general', label: '运行总览', hint: '开关与任务计划', icon: Cog6ToothIcon },
  { id: 'emby', label: 'Emby 账号', hint: '媒体服务器保活', icon: ServerStackIcon },
  { id: 'telegram', label: 'Telegram 账号', hint: '签到与群组功能', icon: ChatBubbleLeftRightIcon },
  { id: 'subsonic', label: 'Subsonic 账号', hint: '音乐服务器保活', icon: MusicalNoteIcon },
  { id: 'network', label: '网络与通知', hint: '代理、推送、站点', icon: ShieldCheckIcon },
  { id: 'review', label: '检查并保存', hint: '一次提交全部配置', icon: ClipboardDocumentListIcon },
]

const fieldClass = 'block w-full rounded-[16px] border-0 bg-white px-4 py-3 text-sm text-slate-900 ring-1 ring-inset ring-slate-200 placeholder:text-slate-400 focus:ring-2 focus:ring-primary-600'
const labelClass = 'space-y-2 text-sm text-slate-600'
const helperClass = 'text-xs leading-5 text-slate-400'
const toggleClass = 'flex items-center justify-between gap-3 rounded-[18px] border border-slate-200 bg-slate-50 px-4 py-3 text-sm text-slate-700'

const configSource = ref<JsonRecord>({})
const loading = ref(true)
const saving = ref(false)
const loadError = ref('')
const saveMessage = ref('')
const validationError = ref('')
const activeStep = ref(0)
const initialSnapshot = ref('')

const general = reactive({
  nofail: true,
  noexit: false,
  debug_cron: false,
  checkiner: {
    enabled: true,
    time_range: '<11:00AM,11:00PM>',
    interval_days: '1',
    timeout: 120,
    retries: 4,
    concurrency: 1,
    random_start: 60,
  },
  emby: { enabled: true, time_range: '<11:00AM,11:00PM>', interval_days: '<7,12>', concurrency: 1, retries: 5 },
  subsonic: { enabled: true, time_range: '<11:00AM,11:00PM>', interval_days: '<7,12>', concurrency: 1, retries: 5 },
  monitor: { enabled: true },
  messager: { enabled: true },
  registrar: { enabled: true, concurrency: 1 },
})

const notifier = reactive({
  enabled: false,
  account: '1',
  immediately: false,
  once: false,
  method: 'telegram',
  apprise_uri: '',
})

const proxy = reactive({
  enabled: false,
  scheme: 'http',
  hostname: '127.0.0.1',
  port: 7890,
  username: '',
  password: '',
})

const telegramUseProxy = ref(true)
const siteScopes = reactive({
  checkiner: [] as string[],
  monitor: [] as string[],
  messager: [] as string[],
  registrar: [] as string[],
})
const embyAccounts = ref<EmbyAccountDraft[]>([])
const telegramAccounts = ref<TelegramAccountDraft[]>([])
const subsonicAccounts = ref<SubsonicAccountDraft[]>([])

const currentStep = computed(() => steps[activeStep.value])
const isFirstStep = computed(() => activeStep.value === 0)
const isLastStep = computed(() => activeStep.value === steps.length - 1)
const accountTotal = computed(() => embyAccounts.value.length + telegramAccounts.value.length + subsonicAccounts.value.length)
const isDirty = computed(() => initialSnapshot.value !== draftSnapshot())
const notifierAccountOptions = computed(() => {
  const options = [{ value: '0', label: '不使用 Telegram 账号' }]
  telegramAccounts.value.forEach((account, index) => options.push({ value: String(index + 1), label: `${index + 1} · ${account.phone || '未填写手机号'}` }))
  if (notifier.account && !options.some(option => option.value === notifier.account)) options.push({ value: notifier.account, label: `${notifier.account} · 当前配置` })
  return options
})
const proxySummary = computed(() => {
  if (!proxy.enabled) return '代理未启用'
  return `${proxy.scheme}://${proxy.hostname || '未填写'}:${proxy.port || '未填写'}`
})

// 深拷贝接口返回的配置，确保编辑草稿不会修改原始快照。
const clone = <T,>(value: T): T => JSON.parse(JSON.stringify(value))

// 读取可选的配置段，避免旧版本配置缺少某个段时出现空值错误。
const recordOf = (value: unknown): JsonRecord => (value && typeof value === 'object' && !Array.isArray(value) ? value as JsonRecord : {})

// 将不规则的历史配置值转换成表单可编辑的数字。
const numberOf = (value: unknown, fallback: number) => {
  const parsed = Number(value)
  return Number.isFinite(parsed) ? parsed : fallback
}

// 将配置中的列表或逗号分隔文本统一成可编辑的站点列表。
const scopeOf = (value: unknown): string[] => {
  if (Array.isArray(value)) return value.map(item => String(item)).filter(Boolean)
  if (typeof value === 'string') return value.split(/[\n,]/).map(item => item.trim()).filter(Boolean)
  return []
}

// 将单值或区间播放时长转换成两个输入框。
const timeRangeOf = (value: unknown, fallback: [number, number] = [300, 600]): [number, number] => {
  if (Array.isArray(value) && value.length >= 2) return [numberOf(value[0], fallback[0]), numberOf(value[1], fallback[1])]
  if (value !== undefined && value !== null && value !== '') {
    const single = numberOf(value, fallback[0])
    return [single, single]
  }
  return fallback
}

// 检查数字输入是否满足后端 schema 的正数或非负数约束。
const numberValid = (value: unknown, minimum: number) => Number.isInteger(Number(value)) && Number(value) >= minimum

// 将账号对象变成 Emby 表单行，同时保留版本升级后可能新增的字段。
const normalizeEmbyAccount = (value: unknown): EmbyAccountDraft => {
  const raw = clone(recordOf(value))
  const [timeStart, timeEnd] = timeRangeOf(raw.time)
  return {
    raw,
    url: String(raw.url || ''),
    username: String(raw.username || ''),
    password: String(raw.password || ''),
    name: String(raw.name || ''),
    enabled: raw.enabled ?? true,
    use_proxy: raw.use_proxy ?? true,
    allow_stream: raw.allow_stream ?? false,
    cf_challenge: raw.cf_challenge ?? true,
    allow_multiple: raw.allow_multiple ?? true,
    play_id: String(raw.play_id || ''),
    time_start: timeStart,
    time_end: timeEnd,
    interval_days: String(raw.interval_days ?? ''),
    time_range: String(raw.time_range ?? ''),
    useragent: String(raw.useragent || raw.ua || ''),
    client: String(raw.client || ''),
    client_version: String(raw.client_version || ''),
    device: String(raw.device || ''),
    device_id: String(raw.device_id || ''),
    continuous: raw.continuous ?? false,
    advanced: Boolean(raw.useragent || raw.ua || raw.client || raw.client_version || raw.device || raw.device_id || raw.continuous),
  }
}

// 将账号对象变成 Subsonic 表单行，同时保留设备和兼容字段。
const normalizeSubsonicAccount = (value: unknown): SubsonicAccountDraft => {
  const raw = clone(recordOf(value))
  const [timeStart, timeEnd] = timeRangeOf(raw.time)
  return {
    raw,
    url: String(raw.url || ''),
    username: String(raw.username || ''),
    password: String(raw.password || ''),
    name: String(raw.name || ''),
    enabled: raw.enabled ?? true,
    use_proxy: raw.use_proxy ?? true,
    time_start: timeStart,
    time_end: timeEnd,
    interval_days: String(raw.interval_days ?? ''),
    time_range: String(raw.time_range ?? ''),
    useragent: String(raw.useragent || raw.ua || ''),
    client: String(raw.client || ''),
    client_version: String(raw.client_version || raw.version || ''),
    device: String(raw.device || ''),
    device_id: String(raw.device_id || ''),
    advanced: Boolean(raw.useragent || raw.ua || raw.client || raw.client_version || raw.version || raw.device || raw.device_id),
  }
}

// 将 Telegram 账号和账号级调度覆盖项转换成一行可编辑表单。
const normalizeTelegramAccount = (value: unknown): TelegramAccountDraft => {
  const raw = clone(recordOf(value))
  const checkinerConfig = recordOf(raw.checkiner_config)
  const registrarConfig = recordOf(raw.registrar_config)
  return {
    raw,
    phone: String(raw.phone || ''),
    enabled: raw.enabled ?? true,
    checkiner: raw.checkiner ?? true,
    monitor: raw.monitor ?? false,
    messager: raw.messager ?? false,
    registrar: raw.registrar ?? false,
    api_id: String(raw.api_id || ''),
    api_hash: String(raw.api_hash || ''),
    session: String(raw.session || ''),
    site_checkiner: scopeOf(recordOf(raw.site).checkiner),
    site_monitor: scopeOf(recordOf(raw.site).monitor),
    site_messager: scopeOf(recordOf(raw.site).messager),
    site_registrar: scopeOf(recordOf(raw.site).registrar),
    checkiner_override: Boolean(raw.checkiner_config),
    checkiner_interval_days: String(checkinerConfig.interval_days ?? ''),
    checkiner_time_range: String(checkinerConfig.time_range ?? ''),
    checkiner_timeout: numberOf(checkinerConfig.timeout, 120),
    checkiner_retries: numberOf(checkinerConfig.retries, 4),
    checkiner_concurrency: numberOf(checkinerConfig.concurrency, 1),
    checkiner_random_start: numberOf(checkinerConfig.random_start, 60),
    registrar_override: Boolean(raw.registrar_config),
    registrar_concurrency: numberOf(registrarConfig.concurrency, 1),
    advanced: Boolean(raw.site || raw.checkiner_config || raw.registrar_config || raw.api_id || raw.api_hash || raw.session),
  }
}

// 从已保存配置同步所有表单状态，保证打开向导就能继续编辑原配置。
const syncDraft = () => {
  const source = configSource.value
  const checkiner = recordOf(source.checkiner)
  const emby = recordOf(source.emby)
  const subsonic = recordOf(source.subsonic)
  const registrar = recordOf(source.registrar)
  const telegram = recordOf(source.telegram)
  const sourceProxy = recordOf(source.proxy)
  const sourceNotifier = recordOf(source.notifier)

  general.nofail = source.nofail ?? true
  general.noexit = source.noexit ?? false
  general.debug_cron = source.debug_cron ?? false
  Object.assign(general.checkiner, {
    enabled: checkiner.enabled ?? true,
    time_range: String(checkiner.time_range ?? '<11:00AM,11:00PM>'),
    interval_days: String(checkiner.interval_days ?? '1'),
    timeout: numberOf(checkiner.timeout, 120),
    retries: numberOf(checkiner.retries, 4),
    concurrency: numberOf(checkiner.concurrency, 1),
    random_start: numberOf(checkiner.random_start, 60),
  })
  Object.assign(general.emby, {
    enabled: emby.enabled ?? true,
    time_range: String(emby.time_range ?? '<11:00AM,11:00PM>'),
    interval_days: String(emby.interval_days ?? '<7,12>'),
    concurrency: numberOf(emby.concurrency, 1),
    retries: numberOf(emby.retries, 5),
  })
  Object.assign(general.subsonic, {
    enabled: subsonic.enabled ?? true,
    time_range: String(subsonic.time_range ?? '<11:00AM,11:00PM>'),
    interval_days: String(subsonic.interval_days ?? '<7,12>'),
    concurrency: numberOf(subsonic.concurrency, 1),
    retries: numberOf(subsonic.retries, 5),
  })
  general.monitor.enabled = recordOf(source.monitor).enabled ?? true
  general.messager.enabled = recordOf(source.messager).enabled ?? true
  Object.assign(general.registrar, {
    enabled: registrar.enabled ?? true,
    concurrency: numberOf(registrar.concurrency, 1),
  })

  Object.assign(notifier, {
    enabled: sourceNotifier.enabled ?? false,
    account: String(sourceNotifier.account ?? '1'),
    immediately: sourceNotifier.immediately ?? false,
    once: sourceNotifier.once ?? false,
    method: String(sourceNotifier.method ?? 'telegram'),
    apprise_uri: String(sourceNotifier.apprise_uri ?? ''),
  })
  Object.assign(proxy, {
    enabled: Boolean(sourceProxy.hostname && sourceProxy.port),
    scheme: String(sourceProxy.scheme ?? 'http'),
    hostname: String(sourceProxy.hostname ?? '127.0.0.1'),
    port: numberOf(sourceProxy.port, 7890),
    username: String(sourceProxy.username ?? ''),
    password: String(sourceProxy.password ?? ''),
  })
  telegramUseProxy.value = telegram.use_proxy ?? true
  Object.assign(siteScopes, {
    checkiner: scopeOf(recordOf(source.site).checkiner),
    monitor: scopeOf(recordOf(source.site).monitor),
    messager: scopeOf(recordOf(source.site).messager),
    registrar: scopeOf(recordOf(source.site).registrar),
  })
  embyAccounts.value = Array.isArray(emby.account) ? emby.account.map(normalizeEmbyAccount) : []
  telegramAccounts.value = Array.isArray(telegram.account) ? telegram.account.map(normalizeTelegramAccount) : []
  subsonicAccounts.value = Array.isArray(subsonic.account) ? subsonic.account.map(normalizeSubsonicAccount) : []
}

// 根据输入框内容生成嵌套站点配置，空配置会被省略以保持旧配置语义。
const buildSiteValue = (account: { site_checkiner: string[]; site_monitor: string[]; site_messager: string[]; site_registrar: string[] }, original: unknown) => {
  const current = recordOf(original)
  const next: JsonRecord = { ...current }
  const values = {
    checkiner: account.site_checkiner,
    monitor: account.site_monitor,
    messager: account.site_messager,
    registrar: account.site_registrar,
  }
  Object.entries(values).forEach(([key, value]) => {
    if (value.length) next[key] = [...value]
    else delete next[key]
  })
  return Object.keys(next).length ? next : undefined
}

// 将 Emby 行还原成后端 schema 使用的账号对象。
const serializeEmbyAccount = (account: EmbyAccountDraft): JsonRecord => ({
  ...account.raw,
  url: account.url.trim(),
  username: account.username.trim(),
  password: account.password,
  name: account.name.trim() || null,
  enabled: account.enabled,
  use_proxy: account.use_proxy,
  allow_stream: account.allow_stream,
  cf_challenge: account.cf_challenge,
  allow_multiple: account.allow_multiple,
  play_id: account.play_id.trim() || null,
  time: [numberOf(account.time_start, 300), numberOf(account.time_end, 600)],
  interval_days: account.interval_days.trim() || null,
  time_range: account.time_range.trim() || null,
  useragent: account.advanced ? account.useragent.trim() || null : account.raw.useragent,
  client: account.advanced ? account.client.trim() || null : account.raw.client,
  client_version: account.advanced ? account.client_version.trim() || null : account.raw.client_version,
  device: account.advanced ? account.device.trim() || null : account.raw.device,
  device_id: account.advanced ? account.device_id.trim() || null : account.raw.device_id,
  continuous: account.advanced ? account.continuous : account.raw.continuous,
})

// 将 Subsonic 行还原成后端 schema 使用的账号对象。
const serializeSubsonicAccount = (account: SubsonicAccountDraft): JsonRecord => ({
  ...account.raw,
  url: account.url.trim(),
  username: account.username.trim(),
  password: account.password,
  name: account.name.trim() || null,
  enabled: account.enabled,
  use_proxy: account.use_proxy,
  time: [numberOf(account.time_start, 300), numberOf(account.time_end, 600)],
  interval_days: account.interval_days.trim() || null,
  time_range: account.time_range.trim() || null,
  useragent: account.advanced ? account.useragent.trim() || null : account.raw.useragent,
  client: account.advanced ? account.client.trim() || null : account.raw.client,
  client_version: account.advanced ? account.client_version.trim() || null : account.raw.client_version,
  device: account.advanced ? account.device.trim() || null : account.raw.device,
  device_id: account.advanced ? account.device_id.trim() || null : account.raw.device_id,
})

// 将 Telegram 行还原成账号、站点范围和账号级调度配置。
const serializeTelegramAccount = (account: TelegramAccountDraft): JsonRecord => {
  const payload: JsonRecord = {
    ...account.raw,
    phone: account.phone.trim(),
    enabled: account.enabled,
    checkiner: account.checkiner,
    monitor: account.monitor,
    messager: account.messager,
    registrar: account.registrar,
    api_id: account.api_id.trim() || null,
    api_hash: account.api_hash.trim() || null,
    session: account.session.trim() || null,
  }
  const site = buildSiteValue(account, account.raw.site)
  if (site) payload.site = site
  else delete payload.site

  if (account.checkiner_override) {
    payload.checkiner_config = {
      ...recordOf(account.raw.checkiner_config),
      interval_days: account.checkiner_interval_days.trim() || null,
      time_range: account.checkiner_time_range.trim() || null,
      timeout: numberOf(account.checkiner_timeout, 120),
      retries: numberOf(account.checkiner_retries, 4),
      concurrency: numberOf(account.checkiner_concurrency, 1),
      random_start: numberOf(account.checkiner_random_start, 60),
    }
  } else {
    delete payload.checkiner_config
  }
  if (account.registrar_override) {
    payload.registrar_config = {
      ...recordOf(account.raw.registrar_config),
      concurrency: numberOf(account.registrar_concurrency, 1),
    }
  } else {
    delete payload.registrar_config
  }
  return payload
}

// 生成最终提交对象，只覆盖向导管理的字段并保留未知配置。
const buildPayload = (): JsonRecord => {
  const payload = clone(configSource.value)
  payload.nofail = general.nofail
  payload.noexit = general.noexit
  payload.debug_cron = general.debug_cron
  payload.checkiner = { ...recordOf(payload.checkiner), ...general.checkiner }
  payload.emby = { ...recordOf(payload.emby), ...general.emby, account: embyAccounts.value.map(serializeEmbyAccount) }
  payload.subsonic = { ...recordOf(payload.subsonic), ...general.subsonic, account: subsonicAccounts.value.map(serializeSubsonicAccount) }
  payload.monitor = { ...recordOf(payload.monitor), ...general.monitor }
  payload.messager = { ...recordOf(payload.messager), ...general.messager }
  payload.registrar = { ...recordOf(payload.registrar), ...general.registrar }
  payload.telegram = { ...recordOf(payload.telegram), use_proxy: telegramUseProxy.value, account: telegramAccounts.value.map(serializeTelegramAccount) }
  payload.notifier = { ...recordOf(payload.notifier), ...notifier }
  if (proxy.enabled) payload.proxy = { scheme: proxy.scheme, hostname: proxy.hostname.trim(), port: numberOf(proxy.port, 7890), username: proxy.username.trim() || null, password: proxy.password || null }
  else delete payload.proxy

  const hasGlobalSite = Object.values(siteScopes).some(values => values.length)
  if (hasGlobalSite || payload.site) {
    payload.site = {
      ...recordOf(payload.site),
      checkiner: [...siteScopes.checkiner],
      monitor: [...siteScopes.monitor],
      messager: [...siteScopes.messager],
      registrar: [...siteScopes.registrar],
    }
  }
  return payload
}

// 将当前草稿序列化，用于提示未保存修改并在保存后建立新快照。
const draftSnapshot = () => JSON.stringify(buildPayload())

// 检查当前步骤的必填内容，错误会留在当前页面而不会丢失其他步骤草稿。
const validateStep = (step = activeStep.value) => {
  validationError.value = ''
  if (step === 0) {
    const positive = [
      ['Telegram 超时', general.checkiner.timeout, 1],
      ['Telegram 并发', general.checkiner.concurrency, 1],
      ['Emby 并发', general.emby.concurrency, 1],
      ['Subsonic 并发', general.subsonic.concurrency, 1],
      ['抢注并发', general.registrar.concurrency, 1],
    ] as const
    const nonNegative = [
      ['Telegram 重试次数', general.checkiner.retries],
      ['随机启动时间', general.checkiner.random_start],
      ['Emby 重试次数', general.emby.retries],
      ['Subsonic 重试次数', general.subsonic.retries],
    ] as const
    const invalidPositive = positive.find(([, value, minimum]) => !numberValid(value, minimum))
    const invalidNonNegative = nonNegative.find(([, value]) => !numberValid(value, 0))
    if (invalidPositive) validationError.value = `${invalidPositive[0]}必须是有效的正整数。`
    else if (invalidNonNegative) validationError.value = `${invalidNonNegative[0]}必须是有效的非负整数。`
  }
  if (step === 1) {
    const invalid = embyAccounts.value.find(account => !account.url.trim() || !account.username.trim() || !account.password)
    if (invalid) validationError.value = '请为每个 Emby 账号填写站点、用户名和密码。'
    else if (embyAccounts.value.some(account => !numberValid(account.time_start, 1) || !numberValid(account.time_end, 1) || account.time_end < account.time_start)) validationError.value = 'Emby 播放时长必须是正整数，且最长时长不能小于最短时长。'
  }
  if (step === 2) {
    const invalid = telegramAccounts.value.find(account => !account.phone.trim())
    if (invalid) validationError.value = '请为每个 Telegram 账号填写带国家区号的手机号。'
  }
  if (step === 3) {
    const invalid = subsonicAccounts.value.find(account => !account.url.trim() || !account.username.trim() || !account.password)
    if (invalid) validationError.value = '请为每个 Subsonic 账号填写站点、用户名和密码。'
    else if (subsonicAccounts.value.some(account => !numberValid(account.time_start, 1) || !numberValid(account.time_end, 1) || account.time_end < account.time_start)) validationError.value = 'Subsonic 播放时长必须是正整数，且最长时长不能小于最短时长。'
  }
  if (step === 4 && proxy.enabled && (!proxy.hostname.trim() || !proxy.port || proxy.port < 1)) {
    validationError.value = '启用代理后，请填写有效的主机和端口。'
  }
  if (step === 4 && !validationError.value && notifier.enabled && notifier.method === 'telegram' && (!telegramAccounts.value.length || notifier.account === '0')) {
    validationError.value = '启用 Telegram 通知时，请先添加账号并选择一个通知账号。'
  }
  if (step === 4 && !validationError.value && notifier.enabled && notifier.method === 'apprise' && !notifier.apprise_uri.trim()) {
    validationError.value = '启用 Apprise 通知时，请填写 Apprise 地址。'
  }
  return !validationError.value
}

// 跳转到任意步骤，允许用户随时回看和修改之前已经填写的内容。
const jumpToStep = (index: number) => {
  activeStep.value = Math.max(0, Math.min(index, steps.length - 1))
  validationError.value = ''
  saveMessage.value = ''
}

// 前进前只校验当前步骤，避免长表单因为一个后续账号阻塞用户回看。
const nextStep = () => {
  if (!validateStep()) return
  if (!isLastStep.value) activeStep.value += 1
}

// 返回上一步时保留全部输入内容，不触发保存或重置。
const previousStep = () => {
  if (!isFirstStep.value) activeStep.value -= 1
}

// 统一校验账号和代理字段，提交按钮只会在完整配置可被后端接受时发送请求。
const validateAll = () => {
  for (let index = 0; index < steps.length - 1; index += 1) {
    if (!validateStep(index)) {
      activeStep.value = index
      return false
    }
  }
  return true
}

// 离开路由或刷新页面前保护未保存草稿，避免填写一半后误操作导致内容丢失。
const canLeave = () => !saving.value && (!isDirty.value || window.confirm('当前配置还有未保存的修改，确定离开吗？'))

// 浏览器刷新或关闭标签页时使用原生提示保护草稿。
const beforeUnload = (event: BeforeUnloadEvent) => {
  if (!isDirty.value && !saving.value) return
  event.preventDefault()
  event.returnValue = ''
}

// 新增一行 Emby 账号，用户可以直接在列表中填写而无需打开下一层页面。
const addEmbyAccount = () => {
  embyAccounts.value.push(normalizeEmbyAccount({ time: [300, 600], enabled: true, use_proxy: true }))
}

// 新增一行 Telegram 账号，并默认开启签到功能。
const addTelegramAccount = () => {
  telegramAccounts.value.push(normalizeTelegramAccount({ checkiner: true, monitor: false, messager: false, registrar: false }))
}

// 新增一行 Subsonic 账号，默认采用统一代理和播放时长。
const addSubsonicAccount = () => {
  subsonicAccounts.value.push(normalizeSubsonicAccount({ time: [300, 600], enabled: true, use_proxy: true }))
}

// 从草稿列表移除账号，所有未保存的改动仍会保留在其他步骤。
const removeAccount = (collection: { value: unknown[] }, index: number) => {
  collection.value.splice(index, 1)
}

// 从后端加载当前配置，首次进入时将所有可编辑字段展开成表单。
const loadConfig = async () => {
  loading.value = true
  loadError.value = ''
  try {
    configSource.value = await auth.request<JsonRecord>('/config')
    syncDraft()
    initialSnapshot.value = draftSnapshot()
  } catch (error) {
    logger.error('Failed to load configuration wizard data', error)
    loadError.value = logger.message(error, '配置加载失败，请检查后端连接。')
  } finally {
    loading.value = false
  }
}

// 保存整份配置，让用户在一次提交中完成所有步骤的修改。
const saveConfig = async () => {
  if (!validateAll()) return
  saving.value = true
  saveMessage.value = ''
  try {
    const payload = buildPayload()
    await auth.request('/config', { method: 'POST', body: JSON.stringify(payload) })
    configSource.value = clone(payload)
    initialSnapshot.value = draftSnapshot()
    saveMessage.value = '配置已保存；如果还有未保存修改，离开页面前会收到提示。'
  } catch (error) {
    logger.error('Failed to save configuration wizard data', error)
    validationError.value = logger.message(error, '配置保存失败，请检查输入后重试。')
  } finally {
    saving.value = false
  }
}

// 放弃当前草稿并重新载入最近一次已保存配置。
const resetDraft = () => {
  if (!isDirty.value || window.confirm('放弃当前未保存的全部修改？')) {
    syncDraft()
    initialSnapshot.value = draftSnapshot()
    validationError.value = ''
    saveMessage.value = '已恢复到最近一次保存的配置。'
  }
}

onBeforeRouteLeave(canLeave)
onMounted(() => {
  window.addEventListener('beforeunload', beforeUnload)
  loadConfig()
})
onBeforeUnmount(() => window.removeEventListener('beforeunload', beforeUnload))
</script>

<template>
  <div class="min-h-full pb-28">
    <section class="relative overflow-hidden rounded-[32px] bg-slate-950 px-5 py-7 text-white shadow-[0_28px_70px_-30px_rgba(15,23,42,0.65)] sm:px-8 sm:py-9">
      <div class="pointer-events-none absolute -right-16 -top-20 h-64 w-64 rounded-full bg-cyan-400/20 blur-3xl" />
      <div class="pointer-events-none absolute -bottom-24 left-1/3 h-56 w-56 rounded-full bg-amber-300/15 blur-3xl" />
      <div class="relative max-w-3xl">
        <p class="text-xs font-semibold uppercase tracking-[0.28em] text-cyan-300">Configuration studio</p>
        <h1 class="mt-3 text-3xl font-semibold tracking-tight sm:text-4xl">把配置一次填完整。</h1>
        <p class="mt-3 max-w-2xl text-sm leading-6 text-slate-300 sm:text-base">
          用一份可返回、可检查的表单管理运行开关、账号、代理和通知。每一步都能直接点击切换，最后统一保存。
        </p>
        <div class="mt-6 flex flex-wrap items-center gap-3 text-xs text-slate-300">
          <span class="inline-flex items-center gap-2 rounded-full border border-white/10 bg-white/10 px-3 py-1.5"><CheckCircleIcon class="h-4 w-4 text-cyan-300" />{{ accountTotal }} 个账号草稿</span>
          <span class="inline-flex items-center gap-2 rounded-full border border-white/10 bg-white/10 px-3 py-1.5"><ShieldCheckIcon class="h-4 w-4 text-amber-300" />{{ proxySummary }}</span>
          <span v-if="isDirty" class="inline-flex items-center gap-2 rounded-full border border-amber-300/30 bg-amber-300/10 px-3 py-1.5 text-amber-100">有未保存修改</span>
        </div>
      </div>
    </section>

    <div v-if="loading" class="mt-6 rounded-[28px] border border-slate-200 bg-white p-10 text-center text-sm text-slate-500 shadow-sm">正在读取当前配置…</div>
    <div v-else-if="loadError" class="mt-6 rounded-[28px] border border-rose-200 bg-rose-50 p-6 text-sm text-rose-700">
      <p class="font-semibold">{{ loadError }}</p>
      <UiButton class="mt-4" size="sm" variant="outline" @click="loadConfig">重新加载</UiButton>
    </div>

    <div v-else class="mt-6 grid gap-6 xl:grid-cols-[240px_minmax(0,1fr)]">
      <aside class="h-fit rounded-[28px] border border-slate-200/80 bg-white/90 p-3 shadow-[0_14px_34px_-24px_rgba(15,23,42,0.3)] xl:sticky xl:top-6">
        <p class="px-3 pb-3 pt-2 text-xs font-semibold uppercase tracking-[0.18em] text-slate-400">配置步骤</p>
        <nav class="space-y-1" aria-label="配置步骤">
          <button
            v-for="(step, index) in steps"
            :key="step.id"
            type="button"
            class="group flex w-full items-center gap-3 rounded-[18px] px-3 py-3 text-left transition"
            :class="activeStep === index ? 'bg-slate-950 text-white shadow-lg shadow-slate-900/15' : 'text-slate-600 hover:bg-slate-50'"
            @click="jumpToStep(index)"
          >
            <span class="flex h-9 w-9 shrink-0 items-center justify-center rounded-2xl" :class="activeStep === index ? 'bg-cyan-300 text-slate-950' : 'bg-slate-100 text-slate-500 group-hover:bg-slate-200'"><component :is="step.icon" class="h-4 w-4" /></span>
            <span class="min-w-0">
              <span class="block truncate text-sm font-semibold">{{ step.label }}</span>
              <span class="mt-0.5 block truncate text-xs" :class="activeStep === index ? 'text-slate-300' : 'text-slate-400'">{{ step.hint }}</span>
            </span>
          </button>
        </nav>
        <div class="mt-4 rounded-[20px] bg-slate-50 px-3 py-3 text-xs leading-5 text-slate-500">
          <p class="font-semibold text-slate-700">随时可以返回</p>
          <p class="mt-1">切换步骤不会提交或清空输入，最后在检查页一次保存。</p>
        </div>
      </aside>

      <main class="min-w-0">
        <div class="mb-4 flex flex-wrap items-center justify-between gap-3">
          <div>
            <p class="text-xs font-semibold uppercase tracking-[0.18em] text-primary-600">{{ String(activeStep + 1).padStart(2, '0') }} / {{ String(steps.length).padStart(2, '0') }}</p>
            <h2 class="mt-1 text-2xl font-semibold tracking-tight text-slate-950">{{ currentStep.label }}</h2>
          </div>
          <span class="rounded-full bg-slate-100 px-3 py-1.5 text-xs font-medium text-slate-500">{{ currentStep.hint }}</span>
        </div>

        <div v-if="validationError" class="mb-4 rounded-[20px] border border-rose-200 bg-rose-50 px-4 py-3 text-sm text-rose-700">{{ validationError }}</div>
        <div v-if="saveMessage" class="mb-4 rounded-[20px] border border-emerald-200 bg-emerald-50 px-4 py-3 text-sm text-emerald-700">{{ saveMessage }}</div>

        <section v-if="activeStep === 0" class="space-y-5">
          <UiCard>
            <template #header>
              <div>
                <h3 class="text-lg font-semibold text-slate-950">运行开关</h3>
                <p class="mt-1 text-sm text-slate-500">先决定哪些模块参与运行，后面的账号可以随时补充。</p>
              </div>
            </template>
            <div class="grid gap-3 sm:grid-cols-3">
              <label :class="toggleClass"><span>失败后继续运行</span><input v-model="general.nofail" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label>
              <label :class="toggleClass"><span>不自动退出</span><input v-model="general.noexit" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label>
              <label :class="toggleClass"><span>调试调度日志</span><input v-model="general.debug_cron" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label>
            </div>
          </UiCard>

          <UiCard>
            <template #header>
              <div>
                <h3 class="text-lg font-semibold text-slate-950">任务计划</h3>
                <p class="mt-1 text-sm text-slate-500">时间范围支持单个时间或区间，例如 <code>&lt;11:00AM,11:00PM&gt;</code>。</p>
              </div>
            </template>
            <div class="space-y-5">
              <div class="rounded-[24px] border border-slate-200 bg-slate-50 p-4 sm:p-5">
                <div class="flex flex-wrap items-center justify-between gap-3"><div><h4 class="font-semibold text-slate-900">Telegram 签到</h4><p :class="helperClass">控制签到等待、重试和并发。</p></div><label class="flex items-center gap-2 text-sm text-slate-700"><input v-model="general.checkiner.enabled" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" />启用</label></div>
                <div class="mt-4 grid gap-4 sm:grid-cols-2 lg:grid-cols-4">
                  <label :class="labelClass"><span class="font-medium text-slate-900">执行时段</span><input v-model="general.checkiner.time_range" :class="fieldClass" placeholder="<11:00AM,11:00PM>" /></label>
                  <label :class="labelClass"><span class="font-medium text-slate-900">间隔天数</span><input v-model="general.checkiner.interval_days" :class="fieldClass" placeholder="1 或 <1,3>" /></label>
                  <label :class="labelClass"><span class="font-medium text-slate-900">超时（秒）</span><input v-model.number="general.checkiner.timeout" type="number" min="1" :class="fieldClass" /></label>
                  <label :class="labelClass"><span class="font-medium text-slate-900">重试次数</span><input v-model.number="general.checkiner.retries" type="number" min="0" :class="fieldClass" /></label>
                  <label :class="labelClass"><span class="font-medium text-slate-900">并发数量</span><input v-model.number="general.checkiner.concurrency" type="number" min="1" :class="fieldClass" /></label>
                  <label :class="labelClass"><span class="font-medium text-slate-900">随机启动（分钟）</span><input v-model.number="general.checkiner.random_start" type="number" min="0" :class="fieldClass" /></label>
                </div>
              </div>

              <div class="grid gap-5 lg:grid-cols-2">
                <div v-for="module in [{ key: 'emby', label: 'Emby 保活', form: general.emby }, { key: 'subsonic', label: 'Subsonic 保活', form: general.subsonic }]" :key="module.key" class="rounded-[24px] border border-slate-200 bg-slate-50 p-4 sm:p-5">
                  <div class="flex items-center justify-between gap-3"><div><h4 class="font-semibold text-slate-900">{{ module.label }}</h4><p :class="helperClass">统一账号的默认计划。</p></div><label class="flex items-center gap-2 text-sm text-slate-700"><input v-model="module.form.enabled" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" />启用</label></div>
                  <div class="mt-4 grid gap-4 sm:grid-cols-2">
                    <label :class="labelClass"><span class="font-medium text-slate-900">执行时段</span><input v-model="module.form.time_range" :class="fieldClass" /></label>
                    <label :class="labelClass"><span class="font-medium text-slate-900">间隔天数</span><input v-model="module.form.interval_days" :class="fieldClass" /></label>
                    <label :class="labelClass"><span class="font-medium text-slate-900">并发数量</span><input v-model.number="module.form.concurrency" type="number" min="1" :class="fieldClass" /></label>
                    <label :class="labelClass"><span class="font-medium text-slate-900">重试次数</span><input v-model.number="module.form.retries" type="number" min="0" :class="fieldClass" /></label>
                  </div>
                </div>
              </div>

              <div class="grid gap-4 sm:grid-cols-3">
                <label :class="toggleClass"><span>群组监控</span><input v-model="general.monitor.enabled" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label>
                <label :class="toggleClass"><span>自动水群</span><input v-model="general.messager.enabled" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label>
                <div class="rounded-[18px] border border-slate-200 bg-slate-50 px-4 py-3"><div class="flex items-center justify-between gap-3 text-sm text-slate-700"><span>定时抢注</span><input v-model="general.registrar.enabled" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></div><label class="mt-3 block text-xs text-slate-500">并发数量<input v-model.number="general.registrar.concurrency" type="number" min="1" :class="`${fieldClass} mt-2`" /></label></div>
              </div>
            </div>
          </UiCard>
        </section>

        <section v-else-if="activeStep === 1" class="space-y-5">
          <div class="flex flex-wrap items-center justify-between gap-3 rounded-[24px] border border-cyan-200 bg-cyan-50 px-4 py-4 sm:px-5"><div><p class="text-sm font-semibold text-cyan-950">一个账号一张卡片</p><p class="mt-1 text-xs leading-5 text-cyan-800">可以先添加空白行，稍后返回继续填写；已保存的账号也能直接编辑。</p></div><UiButton size="sm" @click="addEmbyAccount"><PlusIcon class="mr-1.5 h-4 w-4" />添加 Emby 账号</UiButton></div>
          <div v-if="!embyAccounts.length" class="rounded-[28px] border border-dashed border-slate-300 bg-white p-10 text-center"><ServerStackIcon class="mx-auto h-10 w-10 text-slate-300" /><p class="mt-3 font-semibold text-slate-700">还没有 Emby 账号</p><p class="mt-1 text-sm text-slate-500">点击上方按钮开始添加，表单会保留你的输入。</p></div>
          <UiCard v-for="(account, index) in embyAccounts" :key="account.raw.id || index" class="overflow-hidden">
            <template #header><div class="flex items-center justify-between gap-3"><div><p class="text-xs font-semibold uppercase tracking-[0.16em] text-primary-600">账号 {{ String(index + 1).padStart(2, '0') }}</p><h3 class="mt-1 text-lg font-semibold text-slate-950">{{ account.name || account.username || '未命名账号' }}</h3></div><button type="button" class="rounded-full p-2 text-slate-400 transition hover:bg-rose-50 hover:text-rose-600" aria-label="删除 Emby 账号" @click="removeAccount(embyAccounts, index)"><TrashIcon class="h-5 w-5" /></button></div></template>
            <div class="grid gap-4 sm:grid-cols-2">
              <label :class="`${labelClass} sm:col-span-2`"><span class="font-medium text-slate-900">Emby 地址 <b class="text-rose-500">*</b></span><input v-model="account.url" :class="fieldClass" placeholder="https://emby.example.com:443" /></label>
              <label :class="labelClass"><span class="font-medium text-slate-900">用户名 <b class="text-rose-500">*</b></span><input v-model="account.username" :class="fieldClass" /></label>
              <label :class="labelClass"><span class="font-medium text-slate-900">密码 <b class="text-rose-500">*</b></span><input v-model="account.password" type="password" :class="fieldClass" autocomplete="new-password" /></label>
              <label :class="labelClass"><span class="font-medium text-slate-900">显示名称</span><input v-model="account.name" :class="fieldClass" placeholder="可选" /></label>
              <label :class="labelClass"><span class="font-medium text-slate-900">播放 ID</span><input v-model="account.play_id" :class="fieldClass" placeholder="可选，指定媒体播放" /></label>
              <div class="rounded-[18px] border border-slate-200 bg-slate-50 px-4 py-3"><p class="text-xs font-medium text-slate-500">模拟观看时长（秒）</p><div class="mt-2 grid grid-cols-2 gap-2"><input v-model.number="account.time_start" type="number" min="1" :class="fieldClass" aria-label="最短播放时长" /><input v-model.number="account.time_end" type="number" min="1" :class="fieldClass" aria-label="最长播放时长" /></div></div>
              <label :class="labelClass"><span class="font-medium text-slate-900">独立间隔天数</span><input v-model="account.interval_days" :class="fieldClass" placeholder="留空沿用全局" /></label>
              <label :class="labelClass"><span class="font-medium text-slate-900">独立执行时段</span><input v-model="account.time_range" :class="fieldClass" placeholder="留空沿用全局" /></label>
            </div>
            <div class="mt-5 grid gap-3 sm:grid-cols-2 lg:grid-cols-5"><label :class="toggleClass"><span>启用</span><input v-model="account.enabled" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label><label :class="toggleClass"><span>使用代理</span><input v-model="account.use_proxy" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label><label :class="toggleClass"><span>允许流播放</span><input v-model="account.allow_stream" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label><label :class="toggleClass"><span>CF 检测</span><input v-model="account.cf_challenge" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label><label :class="toggleClass"><span>允许多开</span><input v-model="account.allow_multiple" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label></div>
            <button type="button" class="mt-5 inline-flex items-center text-sm font-semibold text-primary-600 hover:text-primary-800" @click="account.advanced = !account.advanced">{{ account.advanced ? '收起高级字段' : '展开高级字段' }}<ArrowRightIcon class="ml-1 h-4 w-4 transition" :class="account.advanced ? 'rotate-90' : ''" /></button>
            <div v-if="account.advanced" class="mt-4 grid gap-4 rounded-[20px] bg-slate-50 p-4 sm:grid-cols-2"><label :class="labelClass"><span>客户端</span><input v-model="account.client" :class="fieldClass" /></label><label :class="labelClass"><span>客户端版本</span><input v-model="account.client_version" :class="fieldClass" /></label><label :class="labelClass"><span>设备名</span><input v-model="account.device" :class="fieldClass" /></label><label :class="labelClass"><span>设备 ID</span><input v-model="account.device_id" :class="fieldClass" /></label><label :class="`${labelClass} sm:col-span-2`"><span>User-Agent</span><input v-model="account.useragent" :class="fieldClass" /></label><label :class="toggleClass"><span>连续播放</span><input v-model="account.continuous" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label></div>
          </UiCard>
        </section>

        <section v-else-if="activeStep === 2" class="space-y-5">
          <div class="flex flex-wrap items-center justify-between gap-3 rounded-[24px] border border-cyan-200 bg-cyan-50 px-4 py-4 sm:px-5"><div><p class="text-sm font-semibold text-cyan-950">账号和功能一起配置</p><p class="mt-1 text-xs leading-5 text-cyan-800">手机号、登录凭据、签到 / 监控 / 水群 / 抢注开关都在同一张卡片里。</p></div><UiButton size="sm" @click="addTelegramAccount"><PlusIcon class="mr-1.5 h-4 w-4" />添加 Telegram 账号</UiButton></div>
          <div v-if="!telegramAccounts.length" class="rounded-[28px] border border-dashed border-slate-300 bg-white p-10 text-center"><ChatBubbleLeftRightIcon class="mx-auto h-10 w-10 text-slate-300" /><p class="mt-3 font-semibold text-slate-700">还没有 Telegram 账号</p><p class="mt-1 text-sm text-slate-500">添加后可在同一页填写模块开关和站点范围。</p></div>
          <UiCard v-for="(account, index) in telegramAccounts" :key="account.raw.phone || index" class="overflow-hidden">
            <template #header><div class="flex items-center justify-between gap-3"><div><p class="text-xs font-semibold uppercase tracking-[0.16em] text-primary-600">账号 {{ String(index + 1).padStart(2, '0') }}</p><h3 class="mt-1 text-lg font-semibold text-slate-950">{{ account.phone || '未填写手机号' }}</h3></div><button type="button" class="rounded-full p-2 text-slate-400 transition hover:bg-rose-50 hover:text-rose-600" aria-label="删除 Telegram 账号" @click="removeAccount(telegramAccounts, index)"><TrashIcon class="h-5 w-5" /></button></div></template>
            <div class="grid gap-4 sm:grid-cols-2">
              <label :class="labelClass"><span class="font-medium text-slate-900">手机号 <b class="text-rose-500">*</b></span><input v-model="account.phone" :class="fieldClass" placeholder="例如加国家区号的手机号" /></label>
              <label :class="labelClass"><span class="font-medium text-slate-900">API ID</span><input v-model="account.api_id" :class="fieldClass" placeholder="可选，使用已有 session 时可留空" /></label>
              <label :class="labelClass"><span class="font-medium text-slate-900">API Hash</span><input v-model="account.api_hash" type="password" :class="fieldClass" autocomplete="new-password" /></label>
              <label :class="toggleClass"><span>启用账号</span><input v-model="account.enabled" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label>
              <label :class="`${labelClass} sm:col-span-2`"><span class="font-medium text-slate-900">Session（可选）</span><textarea v-model="account.session" rows="2" :class="fieldClass" placeholder="已有 Telegram session 字符串可直接粘贴" /></label>
            </div>
            <div class="mt-5 grid gap-3 sm:grid-cols-2 lg:grid-cols-4"><label :class="toggleClass"><span>机器人签到</span><input v-model="account.checkiner" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label><label :class="toggleClass"><span>群组监控</span><input v-model="account.monitor" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label><label :class="toggleClass"><span>自动水群</span><input v-model="account.messager" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label><label :class="toggleClass"><span>定时抢注</span><input v-model="account.registrar" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label></div>
            <button type="button" class="mt-5 inline-flex items-center text-sm font-semibold text-primary-600 hover:text-primary-800" @click="account.advanced = !account.advanced">{{ account.advanced ? '收起站点和独立调度' : '展开站点和独立调度' }}<ArrowRightIcon class="ml-1 h-4 w-4 transition" :class="account.advanced ? 'rotate-90' : ''" /></button>
            <div v-if="account.advanced" class="mt-4 space-y-5 rounded-[20px] bg-slate-50 p-4">
              <div class="grid gap-5 lg:grid-cols-2"><div><p class="text-sm font-semibold text-slate-900">账号专属站点</p><p :class="helperClass">留空表示沿用全局站点范围。</p><div class="mt-3 space-y-4"><div><p class="mb-2 text-xs font-medium text-slate-500">签到站点</p><UiListInput v-model="account.site_checkiner" label="签到站点" placeholder="站点名或 all" /></div><div><p class="mb-2 text-xs font-medium text-slate-500">监控站点</p><UiListInput v-model="account.site_monitor" label="监控站点" /></div><div><p class="mb-2 text-xs font-medium text-slate-500">水群站点</p><UiListInput v-model="account.site_messager" label="水群站点" /></div><div><p class="mb-2 text-xs font-medium text-slate-500">抢注站点</p><UiListInput v-model="account.site_registrar" label="抢注站点" /></div></div></div><div class="space-y-4"><label :class="toggleClass"><span>启用独立签到调度</span><input v-model="account.checkiner_override" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label><div v-if="account.checkiner_override" class="grid gap-3 sm:grid-cols-2"><label :class="labelClass"><span>间隔天数</span><input v-model="account.checkiner_interval_days" :class="fieldClass" /></label><label :class="labelClass"><span>执行时段</span><input v-model="account.checkiner_time_range" :class="fieldClass" /></label><label :class="labelClass"><span>超时（秒）</span><input v-model.number="account.checkiner_timeout" type="number" min="1" :class="fieldClass" /></label><label :class="labelClass"><span>重试次数</span><input v-model.number="account.checkiner_retries" type="number" min="0" :class="fieldClass" /></label><label :class="labelClass"><span>并发数量</span><input v-model.number="account.checkiner_concurrency" type="number" min="1" :class="fieldClass" /></label><label :class="labelClass"><span>随机启动（分钟）</span><input v-model.number="account.checkiner_random_start" type="number" min="0" :class="fieldClass" /></label></div><label :class="toggleClass"><span>启用独立抢注并发</span><input v-model="account.registrar_override" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label><label v-if="account.registrar_override" :class="labelClass"><span>抢注并发数量</span><input v-model.number="account.registrar_concurrency" type="number" min="1" :class="fieldClass" /></label></div></div>
            </div>
          </UiCard>
        </section>

        <section v-else-if="activeStep === 3" class="space-y-5">
          <div class="flex flex-wrap items-center justify-between gap-3 rounded-[24px] border border-cyan-200 bg-cyan-50 px-4 py-4 sm:px-5"><div><p class="text-sm font-semibold text-cyan-950">音乐服务器保活</p><p class="mt-1 text-xs leading-5 text-cyan-800">支持多个 Navidrome / Subsonic 账号，所有账号可直接编辑。</p></div><UiButton size="sm" @click="addSubsonicAccount"><PlusIcon class="mr-1.5 h-4 w-4" />添加 Subsonic 账号</UiButton></div>
          <div v-if="!subsonicAccounts.length" class="rounded-[28px] border border-dashed border-slate-300 bg-white p-10 text-center"><MusicalNoteIcon class="mx-auto h-10 w-10 text-slate-300" /><p class="mt-3 font-semibold text-slate-700">还没有 Subsonic 账号</p><p class="mt-1 text-sm text-slate-500">没有账号时可以直接跳到下一步。</p></div>
          <UiCard v-for="(account, index) in subsonicAccounts" :key="account.raw.id || index" class="overflow-hidden">
            <template #header><div class="flex items-center justify-between gap-3"><div><p class="text-xs font-semibold uppercase tracking-[0.16em] text-primary-600">账号 {{ String(index + 1).padStart(2, '0') }}</p><h3 class="mt-1 text-lg font-semibold text-slate-950">{{ account.name || account.username || '未命名账号' }}</h3></div><button type="button" class="rounded-full p-2 text-slate-400 transition hover:bg-rose-50 hover:text-rose-600" aria-label="删除 Subsonic 账号" @click="removeAccount(subsonicAccounts, index)"><TrashIcon class="h-5 w-5" /></button></div></template>
            <div class="grid gap-4 sm:grid-cols-2">
              <label :class="`${labelClass} sm:col-span-2`"><span class="font-medium text-slate-900">Subsonic 地址 <b class="text-rose-500">*</b></span><input v-model="account.url" :class="fieldClass" placeholder="https://music.example.com:4040" /></label>
              <label :class="labelClass"><span class="font-medium text-slate-900">用户名 <b class="text-rose-500">*</b></span><input v-model="account.username" :class="fieldClass" /></label>
              <label :class="labelClass"><span class="font-medium text-slate-900">密码 <b class="text-rose-500">*</b></span><input v-model="account.password" type="password" :class="fieldClass" autocomplete="new-password" /></label>
              <label :class="labelClass"><span class="font-medium text-slate-900">显示名称</span><input v-model="account.name" :class="fieldClass" /></label>
              <div class="rounded-[18px] border border-slate-200 bg-slate-50 px-4 py-3"><p class="text-xs font-medium text-slate-500">模拟播放时长（秒）</p><div class="mt-2 grid grid-cols-2 gap-2"><input v-model.number="account.time_start" type="number" min="1" :class="fieldClass" aria-label="最短播放时长" /><input v-model.number="account.time_end" type="number" min="1" :class="fieldClass" aria-label="最长播放时长" /></div></div>
              <label :class="labelClass"><span class="font-medium text-slate-900">独立间隔天数</span><input v-model="account.interval_days" :class="fieldClass" placeholder="留空沿用全局" /></label>
              <label :class="labelClass"><span class="font-medium text-slate-900">独立执行时段</span><input v-model="account.time_range" :class="fieldClass" placeholder="留空沿用全局" /></label>
            </div>
            <div class="mt-5 grid gap-3 sm:grid-cols-2"><label :class="toggleClass"><span>启用账号</span><input v-model="account.enabled" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label><label :class="toggleClass"><span>使用代理</span><input v-model="account.use_proxy" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label></div>
            <button type="button" class="mt-5 inline-flex items-center text-sm font-semibold text-primary-600 hover:text-primary-800" @click="account.advanced = !account.advanced">{{ account.advanced ? '收起高级字段' : '展开高级字段' }}<ArrowRightIcon class="ml-1 h-4 w-4 transition" :class="account.advanced ? 'rotate-90' : ''" /></button>
            <div v-if="account.advanced" class="mt-4 grid gap-4 rounded-[20px] bg-slate-50 p-4 sm:grid-cols-2"><label :class="labelClass"><span>客户端</span><input v-model="account.client" :class="fieldClass" /></label><label :class="labelClass"><span>客户端版本</span><input v-model="account.client_version" :class="fieldClass" /></label><label :class="labelClass"><span>设备名</span><input v-model="account.device" :class="fieldClass" /></label><label :class="labelClass"><span>设备 ID</span><input v-model="account.device_id" :class="fieldClass" /></label><label :class="`${labelClass} sm:col-span-2`"><span>User-Agent</span><input v-model="account.useragent" :class="fieldClass" /></label></div>
          </UiCard>
        </section>

        <section v-else-if="activeStep === 4" class="space-y-5">
          <UiCard>
            <template #header><div><h3 class="text-lg font-semibold text-slate-950">网络代理</h3><p class="mt-1 text-sm text-slate-500">Emby 和 Telegram 可以使用统一代理，也可在账号卡片里单独关闭。</p></div></template>
            <label :class="toggleClass"><span>启用统一代理</span><input v-model="proxy.enabled" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label>
            <div class="mt-4 grid gap-4 sm:grid-cols-2"><label :class="labelClass"><span class="font-medium text-slate-900">协议</span><select v-model="proxy.scheme" :disabled="!proxy.enabled" :class="`${fieldClass} disabled:bg-slate-100`"><option value="http">HTTP</option><option value="socks5">SOCKS5</option></select></label><label :class="labelClass"><span class="font-medium text-slate-900">主机</span><input v-model="proxy.hostname" :disabled="!proxy.enabled" :class="`${fieldClass} disabled:bg-slate-100`" placeholder="127.0.0.1" /></label><label :class="labelClass"><span class="font-medium text-slate-900">端口</span><input v-model.number="proxy.port" :disabled="!proxy.enabled" type="number" min="1" :class="`${fieldClass} disabled:bg-slate-100`" placeholder="7890" /></label><label :class="labelClass"><span class="font-medium text-slate-900">用户名</span><input v-model="proxy.username" :disabled="!proxy.enabled" :class="`${fieldClass} disabled:bg-slate-100`" placeholder="可选" /></label><label :class="labelClass"><span class="font-medium text-slate-900">密码</span><input v-model="proxy.password" :disabled="!proxy.enabled" type="password" :class="`${fieldClass} disabled:bg-slate-100`" placeholder="可选" /></label></div>
            <label class="mt-4 flex items-center justify-between gap-3 rounded-[18px] border border-slate-200 bg-slate-50 px-4 py-3 text-sm text-slate-700"><span>Telegram 连接使用代理</span><input v-model="telegramUseProxy" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label>
          </UiCard>
          <UiCard>
            <template #header><div><h3 class="text-lg font-semibold text-slate-950">日志通知</h3><p class="mt-1 text-sm text-slate-500">可使用 Telegram 账号或 Apprise 推送执行结果。</p></div></template>
            <div class="grid gap-4 sm:grid-cols-2"><label :class="toggleClass"><span>启用通知</span><input v-model="notifier.enabled" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label><label :class="labelClass"><span class="font-medium text-slate-900">Telegram 账号</span><select v-model="notifier.account" :class="fieldClass"><option v-for="option in notifierAccountOptions" :key="option.value" :value="option.value">{{ option.label }}</option></select></label><label :class="labelClass"><span class="font-medium text-slate-900">通知方式</span><select v-model="notifier.method" :class="fieldClass"><option value="telegram">Telegram</option><option value="apprise">Apprise</option></select></label><label :class="labelClass"><span class="font-medium text-slate-900">Apprise 地址</span><input v-model="notifier.apprise_uri" :class="fieldClass" placeholder="仅 Apprise 方式需要" /></label><label :class="toggleClass"><span>立即推送</span><input v-model="notifier.immediately" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label><label :class="toggleClass"><span>推送启动时首次任务</span><input v-model="notifier.once" type="checkbox" class="h-4 w-4 rounded border-slate-300 text-primary-600 focus:ring-primary-600" /></label></div>
          </UiCard>
          <UiCard>
            <template #header><div><h3 class="text-lg font-semibold text-slate-950">全局站点范围</h3><p class="mt-1 text-sm text-slate-500">输入站点名后点击添加；留空表示使用默认站点。支持 <code>all</code>、<code>-site</code> 和 <code>+site</code>。</p></div></template>
            <div class="grid gap-5 lg:grid-cols-2"><div><p class="mb-2 text-xs font-medium text-slate-500">签到站点</p><UiListInput v-model="siteScopes.checkiner" label="签到站点" placeholder="例如 cc 或 all" /></div><div><p class="mb-2 text-xs font-medium text-slate-500">监控站点</p><UiListInput v-model="siteScopes.monitor" label="监控站点" placeholder="例如 future" /></div><div><p class="mb-2 text-xs font-medium text-slate-500">水群站点</p><UiListInput v-model="siteScopes.messager" label="水群站点" placeholder="例如 smart_pornfans" /></div><div><p class="mb-2 text-xs font-medium text-slate-500">抢注站点</p><UiListInput v-model="siteScopes.registrar" label="抢注站点" placeholder="例如 templ_a<Bot>" /></div></div>
          </UiCard>
        </section>

        <section v-else class="space-y-5">
          <UiCard>
            <template #header><div><h3 class="text-lg font-semibold text-slate-950">保存前检查</h3><p class="mt-1 text-sm text-slate-500">确认摘要无误后，点击底部「保存全部配置」。</p></div></template>
            <div class="grid gap-3 sm:grid-cols-3"><div class="rounded-[20px] bg-slate-50 p-4"><p class="text-xs text-slate-500">Emby 账号</p><p class="mt-2 text-2xl font-semibold text-slate-950">{{ embyAccounts.length }}</p></div><div class="rounded-[20px] bg-slate-50 p-4"><p class="text-xs text-slate-500">Telegram 账号</p><p class="mt-2 text-2xl font-semibold text-slate-950">{{ telegramAccounts.length }}</p></div><div class="rounded-[20px] bg-slate-50 p-4"><p class="text-xs text-slate-500">Subsonic 账号</p><p class="mt-2 text-2xl font-semibold text-slate-950">{{ subsonicAccounts.length }}</p></div></div>
            <div class="mt-5 divide-y divide-slate-200 rounded-[20px] border border-slate-200"><div class="flex items-center justify-between gap-4 px-4 py-3 text-sm"><span class="text-slate-500">启用模块</span><span class="font-semibold text-slate-900">{{ [general.checkiner.enabled && '签到', general.monitor.enabled && '监控', general.messager.enabled && '水群', general.registrar.enabled && '抢注', general.emby.enabled && 'Emby', general.subsonic.enabled && 'Subsonic'].filter(Boolean).join('、') || '无' }}</span></div><div class="flex items-center justify-between gap-4 px-4 py-3 text-sm"><span class="text-slate-500">网络</span><span class="font-semibold text-slate-900">{{ proxySummary }}</span></div><div class="flex items-center justify-between gap-4 px-4 py-3 text-sm"><span class="text-slate-500">日志通知</span><span class="font-semibold text-slate-900">{{ notifier.enabled ? `${notifier.method} · 账号 ${notifier.account}` : '未启用' }}</span></div><div class="flex items-center justify-between gap-4 px-4 py-3 text-sm"><span class="text-slate-500">站点规则</span><span class="font-semibold text-slate-900">{{ Object.values(siteScopes).reduce((total, values) => total + values.length, 0) }} 条显式规则</span></div></div>
          </UiCard>
          <div class="rounded-[24px] border border-amber-200 bg-amber-50 px-4 py-4 text-sm leading-6 text-amber-900"><p class="font-semibold">提交后会覆盖当前配置文件中由本向导管理的字段。</p><p class="mt-1">账号的未知兼容字段会被保留；如果要配置自定义站点模板，请继续使用「自定义站点」页面。</p></div>
        </section>
      </main>
    </div>

    <div v-if="!loading && !loadError" class="pointer-events-none fixed inset-x-0 bottom-[max(0.75rem,env(safe-area-inset-bottom))] z-40 flex justify-center px-3 xl:left-72">
      <div class="pointer-events-auto flex w-full max-w-4xl flex-wrap items-center justify-between gap-3 rounded-[24px] border border-slate-200 bg-white/95 px-4 py-3 shadow-[0_24px_60px_-24px_rgba(15,23,42,0.35)] backdrop-blur sm:px-5">
        <div class="flex items-center gap-2 text-sm text-slate-500"><span class="h-2 w-2 rounded-full" :class="isDirty ? 'bg-amber-400' : 'bg-emerald-400'" /><span>{{ isDirty ? '草稿尚未保存' : '配置已同步' }}</span></div>
        <div class="flex flex-wrap items-center justify-end gap-2"><UiButton size="sm" variant="ghost" :disabled="!isDirty || saving" @click="resetDraft">重置草稿</UiButton><UiButton v-if="!isFirstStep" size="sm" variant="outline" :disabled="saving" @click="previousStep"><ArrowLeftIcon class="mr-1.5 h-4 w-4" />上一步</UiButton><UiButton v-if="!isLastStep" size="sm" :disabled="saving" @click="nextStep">下一步<ArrowRightIcon class="ml-1.5 h-4 w-4" /></UiButton><UiButton v-else size="sm" :loading="saving" @click="saveConfig"><CheckCircleIcon class="mr-1.5 h-4 w-4" />保存全部配置</UiButton></div>
      </div>
    </div>
  </div>
</template>
