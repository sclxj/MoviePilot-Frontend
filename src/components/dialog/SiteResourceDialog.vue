<script setup lang="ts">
import { Site } from '@/api/types'
import { useDisplay } from 'vuetify'
import api from '@/api'
import type { TorrentInfo } from '@/api/types'
import { formatFileSize } from '@core/utils/formatters'
import AddDownloadDialog from '../dialog/AddDownloadDialog.vue'

// 显示器宽度
const display = useDisplay()

// 输入参数
const props = defineProps({
  site: Object as PropType<Site>,
})

// 注册事件
const emit = defineEmits(['close'])

// 数据列表
const resourceDataList = ref<TorrentInfo[]>([])

// 搜索
const resourceSearch = ref('')

// 总条数
const resourceTotalItems = ref(0)

// 每页条数
const resourceItemsPerPage = ref(25)

// 加载状态
const resourceLoading = ref(false)

// 种子元数据
const torrent = ref<TorrentInfo>()

// 资源浏览表头
const resourceHeaders = [
  { title: '标题', key: 'title', sortable: false },
  { title: '时间', key: 'pubdate', sortable: true },
  { title: '大小', key: 'size', sortable: true },
  { title: '做种', key: 'seeders', sortable: true },
  { title: '下载', key: 'peers', sortable: true },
  { title: '', key: 'actions', sortable: false },
]

// 打开种子详情页面
function openTorrentDetail(page_url: string) {
  window.open(page_url, '_blank')
}

// 下载种子文件
async function downloadTorrentFile(enclosure: string) {
  window.open(enclosure, '_blank')
}

// 调用API，查询站点资源
async function getResourceList() {
  resourceLoading.value = true
  try {
    resourceDataList.value = await api.get(`site/resource/${props.site?.id}`)
    resourceLoading.value = false
  } catch (error) {
    console.error(error)
  }
}

// 促销Chip类
function getVolumeFactorClass(downloadVolume: number, uploadVolume: number) {
  if (downloadVolume === 0) return 'text-white bg-lime-500'
  else if (downloadVolume < 1) return 'text-white bg-green-500'
  else if (uploadVolume !== 1) return 'text-white bg-sky-500'
  else return 'text-white bg-gray-500'
}

// 添加下载
async function addDownload(_torrent: any) {
  torrent.value = _torrent
  addDownloadDialog.value = true
}

// 添加下载对话框
const addDownloadDialog = ref(false)

// 添加下载成功
function addDownloadSuccess(url: string) {
  addDownloadDialog.value = false
}

// 添加下载失败
function addDownloadError(error: string) {
  addDownloadDialog.value = false
}

// 装载时查询站点图标
onMounted(() => {
  getResourceList()
})
</script>
<template>
  <VDialog max-width="80rem" scrollable z-index="1010" :fullscreen="!display.mdAndUp.value">
    <VCard :title="`浏览 - ${props.site?.name}`">
      <DialogCloseBtn @click="emit('close')" />
      <VDivider />
      <VCardText class="pt-2">
        <VDataTable
          v-model:items-per-page="resourceItemsPerPage"
          :headers="resourceHeaders"
          :items="resourceDataList"
          :items-length="resourceTotalItems"
          :search="resourceSearch"
          :loading="resourceLoading"
          density="compact"
          item-value="title"
          return-object
          fixed-header
          hover
          items-per-page-text="每页条数"
          page-text="{0}-{1} 共 {2} 条"
          loading-text="加载中..."
        >
          <template #item.title="{ item }">
            <a href="javascript:void(0)" @click.stop="addDownload(item)">
              <div class="text-high-emphasis pt-1">
                {{ item.title }}
              </div>
              <div class="text-sm my-1">
                {{ item.description }}
              </div>
              <VChip v-if="item.hit_and_run" variant="elevated" size="small" class="me-1 mb-1 text-white bg-black">
                H&R
              </VChip>
              <VChip v-if="item.freedate_diff" variant="elevated" color="secondary" size="small" class="me-1 mb-1">
                {{ item.freedate_diff }}
              </VChip>
              <VChip
                v-for="(label, index) in item.labels"
                :key="index"
                variant="elevated"
                size="small"
                color="primary"
                class="me-1 mb-1"
              >
                {{ label }}
              </VChip>
              <VChip
                v-if="item.downloadvolumefactor !== 1 || item.uploadvolumefactor !== 1"
                :class="getVolumeFactorClass(item.downloadvolumefactor, item.uploadvolumefactor)"
                variant="elevated"
                size="small"
                class="me-1 mb-1"
              >
                {{ item.volume_factor }}
              </VChip>
            </a>
          </template>
          <template #item.pubdate="{ item }">
            <div>{{ item.date_elapsed }}</div>
            <div class="text-sm">
              {{ item.pubdate }}
            </div>
          </template>
          <template #item.size="{ item }">
            <div class="text-nowrap whitespace-nowrap">
              {{ formatFileSize(item.size) }}
            </div>
          </template>
          <template #item.seeders="{ item }">
            <div>{{ item.seeders }}</div>
          </template>
          <template #item.peers="{ item }">
            <div>{{ item.peers }}</div>
          </template>
          <template #item.actions="{ item }">
            <div class="me-n3">
              <IconBtn>
                <VIcon icon="mdi-dots-vertical" />
                <VMenu activator="parent" close-on-content-click>
                  <VList>
                    <VListItem variant="plain" @click="openTorrentDetail(item.page_url || '')">
                      <template #prepend>
                        <VIcon icon="mdi-information" />
                      </template>
                      <VListItemTitle>查看详情</VListItemTitle>
                    </VListItem>
                    <VListItem
                      v-if="item.enclosure?.startsWith('http')"
                      variant="plain"
                      @click="downloadTorrentFile(item.enclosure)"
                    >
                      <template #prepend>
                        <VIcon icon="mdi-download" />
                      </template>
                      <VListItemTitle>下载种子文件</VListItemTitle>
                    </VListItem>
                  </VList>
                </VMenu>
              </IconBtn>
            </div>
          </template>
          <template #no-data> 没有数据 </template>
        </VDataTable>
      </VCardText>
    </VCard>
    <!-- 添加下载对话框 -->
    <AddDownloadDialog
      v-if="addDownloadDialog"
      v-model="addDownloadDialog"
      :torrent="torrent"
      @done="addDownloadSuccess"
      @error="addDownloadError"
      @close="addDownloadDialog = false"
    />
  </VDialog>
</template>

<style lang="scss" scoped>
.v-table th {
  white-space: nowrap;
}
</style>
