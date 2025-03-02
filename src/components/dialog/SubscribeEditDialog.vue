<script lang="ts" setup>
import { useToast } from 'vue-toast-notification'
import { numberValidator } from '@/@validators'
import api from '@/api'
import type { DownloaderConf, FilterRuleGroup, Site, Subscribe, TransferDirectoryConf } from '@/api/types'
import { useDisplay } from 'vuetify'
import { useConfirm } from 'vuetify-use-dialog'
import { VTextarea, VTextField } from 'vuetify/lib/components/index.mjs'

// 显示器宽度
const display = useDisplay()

// 确认框
const createConfirm = useConfirm()

// 输入参数
const props = defineProps({
  subid: Number,
  default: Boolean,
  type: String,
})

// 定义触发的自定义事件
const emit = defineEmits(['remove', 'save', 'close'])

const activeTab = ref('basic')

// 站点数据列表
const siteList = ref<Site[]>([])

// 下载目录列表
const downloadDirectories = ref<TransferDirectoryConf[]>([])

// 站点选择下载框
const selectSitesOptions = ref<{ [key: number]: string }[]>([])

// 所有规则组列表
const filterRuleGroups = ref<FilterRuleGroup[]>([])

// 订阅编辑表单
const subscribeForm = ref<Subscribe>({
  id: props.subid ?? 0,
  name: '',
  year: '',
  type: '',
  tmdbid: 0,
  state: '',
  last_update: '',
  username: '',
  sites: [],
  best_version: undefined,
  current_priority: 0,
  downloader: '',
  date: '',
  show_edit_dialog: false,
})

// 提示框
const $toast = useToast()

// 下载器选项
const downloaderOptions = ref<{ title: string; value: string }[]>([])

async function loadDownloaderSetting() {
  try {
    const downloaders: DownloaderConf[] = await api.get('download/clients')
    downloaderOptions.value = [
      { title: '默认', value: '' },
      ...downloaders.map((item: { name: any }) => ({
        title: item.name,
        value: item.name,
      })),
    ]
  } catch (error) {
    console.error('加载下载器设置失败:', error)
  }
}

// 加载规则组
async function queryFilterRuleGroups() {
  try {
    const result: { [key: string]: any } = await api.get('system/setting/UserFilterRuleGroups')
    filterRuleGroups.value = result.data?.value ?? []
  } catch (error) {
    console.log(error)
  }
}

// 过滤规则组选择项
const filterRuleGroupOptions = computed(() => {
  return filterRuleGroups.value.map(item => ({
    title: item.name,
    value: item.name,
  }))
})

// 调用API修改订阅
async function updateSubscribeInfo() {
  try {
    const result: { [key: string]: any } = await api.put('subscribe/', subscribeForm.value)
    // 提示
    if (result.success) {
      $toast.success(`${subscribeForm.value.name} 更新成功！`)
      // 通知父组件刷新
      emit('save')
    } else {
      $toast.error(`${subscribeForm.value.name} 更新失败：${result.message}！`)
    }
  } catch (e) {
    console.log(e)
  }
}

// 设置用户设置的默认订阅规则
async function saveDefaultSubscribeConfig() {
  try {
    let subscribe_config_url = ''
    if (props.type === '电影') subscribe_config_url = 'system/setting/DefaultMovieSubscribeConfig'
    else subscribe_config_url = 'system/setting/DefaultTvSubscribeConfig'

    const result: { [key: string]: any } = await api.post(subscribe_config_url, subscribeForm.value)
    if (result.success) $toast.success(`${props.type}订阅默认规则保存成功`)
    else $toast.error(`${props.type}订阅默认规则保存失败！`)

    // 通知父组件刷新
    emit('save')
  } catch (error) {
    console.log(error)
  }
}

// 查询用户设置的默认订阅规则
async function queryDefaultSubscribeConfig() {
  try {
    let subscribe_config_url = ''
    if (props.type === '电影') subscribe_config_url = 'system/setting/DefaultMovieSubscribeConfig'
    else subscribe_config_url = 'system/setting/DefaultTvSubscribeConfig'

    const result: { [key: string]: any } = await api.get(subscribe_config_url)

    if (result.data.value) subscribeForm.value = result.data?.value ?? ''
  } catch (error) {
    console.log(error)
  }
}

// 获取站点列表数据
async function loadSites() {
  try {
    const data: Site[] = await api.get('site/rss')

    // 过滤站点，只有启用的站点才显示
    siteList.value = data.filter(item => item.is_active)
  } catch (error) {
    console.error(error)
  }
}

// 获取站点列表选择框数据
async function getSiteList() {
  // 加载订阅站点列表
  if (!siteList.value.length) await loadSites()

  const maps = siteList.value.map(item => {
    return {
      title: item.name,
      value: item.id,
    }
  })

  selectSitesOptions.value = maps.flat()
}

// 获取订阅信息
async function getSubscribeInfo() {
  try {
    const result: Subscribe = await api.get(`subscribe/${props.subid}`)
    subscribeForm.value = result
    subscribeForm.value.best_version = subscribeForm.value.best_version === 1
    subscribeForm.value.search_imdbid = subscribeForm.value.search_imdbid === 1
  } catch (e) {
    console.log(e)
  }
}

// 删除订阅
async function removeSubscribe() {
  const isConfirmed = await createConfirm({
    title: '确认',
    content: `是否确认取消订阅？`,
  })

  if (!isConfirmed) return
  try {
    const result: { [key: string]: any } = await api.delete(`subscribe/${props.subid}`)

    if (result.success) {
      $toast.success(`订阅 ${subscribeForm.value.name} 已取消！`)
      // 通知父组件刷新
      emit('remove')
    }
  } catch (e) {
    console.log(e)
  }
}

// 查询下载目录
async function loadDownloadDirectories() {
  try {
    const result: { [key: string]: any } = await api.get('system/setting/Directories')
    if (result.success && result.data?.value) {
      downloadDirectories.value = result.data.value
    }
  } catch (error) {
    console.log(error)
  }
}

// 保存目录下拉框
const targetDirectories = computed(() => {
  // 去重后的下载目录
  return downloadDirectories.value.map(item => item.download_path)
})

// 质量选择框数据
const qualityOptions = ref([
  {
    title: '全部',
    value: '',
  },
  {
    title: '蓝光原盘',
    value: 'Blu-?Ray.+VC-?1|Blu-?Ray.+AVC|UHD.+blu-?ray.+HEVC|MiniBD',
  },
  {
    title: 'Remux',
    value: 'Remux',
  },
  {
    title: 'BluRay',
    value: 'Blu-?Ray',
  },
  {
    title: 'UHD',
    value: 'UHD|UltraHD',
  },
  {
    title: 'WEB-DL',
    value: 'WEB-?DL|WEB-?RIP',
  },
  {
    title: 'HDTV',
    value: 'HDTV',
  },
  {
    title: 'H265',
    value: '[Hx].?265|HEVC',
  },
  {
    title: 'H264',
    value: '[Hx].?264|AVC',
  },
])

// 分辨率选择框数据
const resolutionOptions = ref([
  {
    title: '全部',
    value: '',
  },
  {
    title: '4k',
    value: '4K|2160p|x2160',
  },
  {
    title: '1080p',
    value: '1080[pi]|x1080',
  },
  {
    title: '720p',
    value: '720[pi]|x720',
  },
])

// 特效选择框数据
const effectOptions = ref([
  {
    title: '全部',
    value: '',
  },
  {
    title: '杜比视界',
    value: 'Dolby[\\s.]+Vision|DOVI|[\\s.]+DV[\\s.]+',
  },
  {
    title: '杜比全景声',
    value: 'Dolby[\\s.]*\\+?Atmos|Atmos',
  },
  {
    title: 'HDR',
    value: '[\\s.]+HDR[\\s.]+|HDR10|HDR10\\+',
  },
  {
    title: 'SDR',
    value: '[\\s.]+SDR[\\s.]+',
  },
])

onMounted(() => {
  queryFilterRuleGroups()
  loadDownloadDirectories()
  getSiteList()
  loadDownloaderSetting()
  if (props.subid) getSubscribeInfo()
  if (props.default) queryDefaultSubscribeConfig()
})
</script>

<template>
  <VDialog scrollable max-width="50rem" :fullscreen="!display.mdAndUp.value">
    <VCard
      :title="`${
        props.default
          ? `${props.type}默认订阅规则`
          : `编辑订阅 - ${subscribeForm.name} ${subscribeForm.season ? `第 ${subscribeForm.season} 季` : ''}`
      }`"
      class="rounded-t"
    >
      <VCardText>
        <DialogCloseBtn @click="emit('close')" />
        <VForm @submit.prevent="() => {}">
          <VTabs v-model="activeTab" show-arrows>
            <VTab value="basic">
              <div>基础</div>
            </VTab>
            <VTab value="advance">
              <div>进阶</div>
            </VTab>
          </VTabs>
          <VWindow v-model="activeTab" class="mt-5 disable-tab-transition" :touch="false">
            <VWindowItem value="basic">
              <div>
                <VRow v-if="!props.default">
                  <VCol cols="12" md="4">
                    <VTextField
                      v-model="subscribeForm.keyword"
                      label="搜索关键词"
                      hint="指定搜索站点时使用的关键词"
                      persistent-hint
                    />
                  </VCol>
                  <VCol v-if="subscribeForm.type === '电视剧'" cols="12" md="4">
                    <VTextField
                      v-model="subscribeForm.total_episode"
                      label="总集数"
                      :rules="[numberValidator]"
                      hint="剧集总集数"
                      persistent-hint
                    />
                  </VCol>
                  <VCol v-if="subscribeForm.type === '电视剧'" cols="12" md="4">
                    <VTextField
                      v-model="subscribeForm.start_episode"
                      label="开始集数"
                      :rules="[numberValidator]"
                      hint="开始订阅集数"
                      persistent-hint
                    />
                  </VCol>
                </VRow>
                <VRow>
                  <VCol cols="12" md="4">
                    <VSelect
                      v-model="subscribeForm.quality"
                      label="质量"
                      :items="qualityOptions"
                      hint="订阅资源质量"
                      persistent-hint
                    />
                  </VCol>
                  <VCol cols="12" md="4">
                    <VSelect
                      v-model="subscribeForm.resolution"
                      label="分辨率"
                      :items="resolutionOptions"
                      hint="订阅资源分辨率"
                      persistent-hint
                    />
                  </VCol>
                  <VCol cols="12" md="4">
                    <VSelect
                      v-model="subscribeForm.effect"
                      label="特效"
                      :items="effectOptions"
                      hint="订阅资源特效"
                      persistent-hint
                    />
                  </VCol>
                </VRow>
                <VRow>
                  <VCol cols="12">
                    <VSelect
                      v-model="subscribeForm.sites"
                      :items="selectSitesOptions"
                      chips
                      label="订阅站点"
                      multiple
                      clearable
                      hint="订阅的站点范围，不选使用系统设置"
                      persistent-hint
                    />
                  </VCol>
                </VRow>
                <VRow>
                  <VCol cols="12" md="6">
                    <VSelect
                      v-model="subscribeForm.downloader"
                      :items="downloaderOptions"
                      label="下载器"
                      hint="指定该订阅使用的下载器"
                      persistent-hint
                    />
                  </VCol>
                  <VCol cols="12" md="6">
                    <VCombobox
                      v-model="subscribeForm.save_path"
                      :items="targetDirectories"
                      label="保存路径"
                      hint="指定该订阅的下载保存路径，留空自动使用设定的下载目录"
                      persistent-hint
                    />
                  </VCol>
                </VRow>
                <VRow>
                  <VCol cols="12" md="4">
                    <VSwitch
                      v-model="subscribeForm.best_version"
                      label="洗版"
                      hint="根据洗版优先级进行洗版订阅"
                      persistent-hint
                    />
                  </VCol>
                  <VCol cols="12" md="4">
                    <VSwitch
                      v-model="subscribeForm.search_imdbid"
                      label="使用 ImdbID 搜索"
                      hint="开使用 ImdbID 精确搜索资源"
                      persistent-hint
                    />
                  </VCol>
                  <VCol v-if="props.default" cols="12" md="4">
                    <VSwitch
                      v-model="subscribeForm.show_edit_dialog"
                      label="订阅时编辑更多规则"
                      hint="添加订阅时显示此编辑订阅对话框"
                      persistent-hint
                    />
                  </VCol>
                </VRow>
              </div>
            </VWindowItem>
            <VWindowItem value="advance">
              <div>
                <VRow>
                  <VCol cols="12" md="6">
                    <VTextField
                      v-model="subscribeForm.include"
                      label="包含（关键字、正则式）"
                      hint="包含规则，支持正则表达式"
                      persistent-hint
                    />
                  </VCol>
                  <VCol cols="12" md="6">
                    <VTextField
                      v-model="subscribeForm.exclude"
                      label="排除（关键字、正则式）"
                      hint="排除规则，支持正则表达式"
                      persistent-hint
                    />
                  </VCol>
                </VRow>
                <VRow>
                  <VCol cols="12" md="6">
                    <VSelect
                      v-model="subscribeForm.filter_groups"
                      :items="filterRuleGroupOptions"
                      chips
                      multiple
                      clearable
                      label="优先级规则组"
                      hint="按选定的过滤规则组对订阅进行过滤"
                      persistent-hint
                    />
                  </VCol>
                  <VCol cols="12" md="6" v-if="!props.default">
                    <VTextField
                      v-model="subscribeForm.media_category"
                      label="自定义类别"
                      hint="指定类别名称，留空自动识别"
                      persistent-hint
                    />
                  </VCol>
                </VRow>
                <VRow v-if="!props.default">
                  <VCol cols="12">
                    <VTextarea
                      v-model="subscribeForm.custom_words"
                      label="自定义识别词"
                      hint="只对该订阅使用的识别词"
                      persistent-hint
                      placeholder="屏蔽词
被替换词 => 替换词
前定位词 <> 后定位词 >> 集偏移量（EP）
被替换词 => 替换词 && 前定位词 <> 后定位词 >> 集偏移量（EP）
其中替换词支持格式：{[tmdbid/doubanid=xxx;type=movie/tv;s=xxx;e=xxx]} 直接指定TMDBID/豆瓣ID识别，其中s、e为季数和集数（可选）"
                    />
                  </VCol>
                </VRow>
              </div>
            </VWindowItem>
          </VWindow>
        </VForm>
      </VCardText>
      <VCardActions class="pt-3">
        <VBtn v-if="!props.default" color="error" @click="removeSubscribe" variant="outlined" class="me-3">
          取消订阅
        </VBtn>
        <VSpacer />
        <VBtn
          variant="elevated"
          @click=";`${props.default ? saveDefaultSubscribeConfig() : updateSubscribeInfo()}`"
          prepend-icon="mdi-content-save"
          class="px-5"
        >
          保存
        </VBtn>
      </VCardActions>
    </VCard>
  </VDialog>
</template>
