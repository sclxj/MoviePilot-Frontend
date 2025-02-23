<script lang="ts" setup>
import type { PropType } from 'vue'
import { formatFileSize } from '@/@core/utils/formatters'
import api from '@/api'
import type { Context } from '@/api/types'
import AddDownloadDialog from '../dialog/AddDownloadDialog.vue'
import { isNullOrEmptyObject } from '@/@core/utils'

// 输入参数
const props = defineProps({
  torrent: Object as PropType<Context>,
  more: Array as PropType<Context[]>,
  width: String,
  height: String,
})

// 更多来源界面
const showMoreTorrents = ref(false)

// 种子信息
const torrent = ref(props.torrent?.torrent_info)

// 媒体信息
const media = ref(props.torrent?.media_info)

// 识别元数据
const meta = ref(props.torrent?.meta_info)

// 当前下载项
const downloadItem = ref(props.torrent)

// 站点图标
const siteIcon = ref('')

// 存储是否已经下载过的记录
const downloaded = ref<string[]>([])

// 添加下载对话框
const addDownloadDialog = ref(false)

// 添加下载成功
function addDownloadSuccess(url: string) {
  addDownloadDialog.value = false
  // 添加下载成功
  downloaded.value.push(url)
}

// 添加下载失败
function addDownloadError(error: string) {
  addDownloadDialog.value = false
}

// 查询站点图标
async function getSiteIcon() {
  try {
    siteIcon.value = (await api.get(`site/icon/${torrent?.value?.site}`)).data.icon
  } catch (error) {
    console.error(error)
  }
}

// 询问并添加下载
async function handleAddDownload(item: Context | null = null) {
  if (item && !isNullOrEmptyObject(item)) {
    downloadItem.value = item
  }
  // 打开下载对话框
  addDownloadDialog.value = true
}

// 打开种子详情页面
function openTorrentDetail() {
  window.open(torrent.value?.page_url, '_blank')
}

// 下载种子文件
async function downloadTorrentFile() {
  window.open(torrent.value?.enclosure, '_blank')
}

// 促销Chip类
function getVolumeFactorClass(downloadVolume: number, uploadVolume: number) {
  if (downloadVolume === 0) return 'text-white bg-lime-500'
  else if (downloadVolume < 1) return 'text-white bg-green-500'
  else if (uploadVolume !== 1) return 'text-white bg-sky-500'
  else return 'text-white bg-gray-500'
}

// 装载时查询站点图标
onMounted(() => {
  getSiteIcon()
})
</script>

<template>
  <div>
    <VCard
      :width="props.width"
      :height="props.height"
      :variant="downloaded.includes(torrent?.enclosure || '') ? 'outlined' : 'elevated'"
      @click="handleAddDownload(props.torrent)"
    >
      <template v-if="!showMoreTorrents" #image>
        <VAvatar class="absolute right-2 bottom-2 rounded" variant="flat" rounded="0">
          <VImg :src="siteIcon" />
        </VAvatar>
      </template>
      <VCardItem class="py-1">
        <VCardTitle class="break-words overflow-visible whitespace-break-spaces">
          {{ media?.title ?? meta?.name }} {{ meta?.season_episode }}
          <span class="text-green-700 ms-2 text-sm">↑{{ torrent?.seeders }}</span>
          <span class="text-orange-700 ms-2 text-sm">↓{{ torrent?.peers }}</span>
        </VCardTitle>
        <template #append>
          <div class="me-n3">
            <IconBtn>
              <VIcon icon="mdi-dots-vertical" />
              <VMenu activator="parent" close-on-content-click>
                <VList>
                  <VListItem variant="plain" @click="openTorrentDetail()">
                    <template #prepend>
                      <VIcon icon="mdi-information" />
                    </template>
                    <VListItemTitle>查看详情</VListItemTitle>
                  </VListItem>
                  <VListItem
                    v-if="props.torrent?.torrent_info?.enclosure?.startsWith('http')"
                    variant="plain"
                    @click="downloadTorrentFile()"
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
      </VCardItem>
      <VCardText class="text-subtitle-2">
        {{ torrent?.title }}
      </VCardText>
      <VCardText>【{{ torrent?.site_name }}】{{ torrent?.description }}</VCardText>
      <VCardItem v-if="torrent?.labels" class="pb-3 pt-0 pe-12">
        <VChip v-if="torrent?.hit_and_run" variant="elevated" size="small" class="me-1 mb-1 text-white bg-black">
          H&R
        </VChip>
        <VChip v-if="torrent?.freedate_diff" variant="elevated" color="secondary" size="small" class="me-1 mb-1">
          {{ torrent?.freedate_diff }}
        </VChip>
        <VChip
          v-for="(label, index) in torrent?.labels"
          :key="index"
          variant="elevated"
          size="small"
          color="primary"
          class="me-1 mb-1"
        >
          {{ label }}
        </VChip>
        <VChip v-if="meta?.edition" variant="elevated" size="small" class="me-1 mb-1 text-white bg-red-500">
          {{ meta?.edition }}
        </VChip>
        <VChip v-if="meta?.resource_pix" variant="elevated" size="small" class="me-1 mb-1 text-white bg-red-500">
          {{ meta?.resource_pix }}
        </VChip>
        <VChip v-if="meta?.video_encode" variant="elevated" size="small" class="me-1 mb-1 text-white bg-orange-500">
          {{ meta?.video_encode }}
        </VChip>
        <VChip v-if="torrent?.size" variant="elevated" size="small" class="me-1 mb-1 text-white bg-yellow-500">
          {{ formatFileSize(torrent?.size) }}
        </VChip>
        <VChip v-if="meta?.resource_team" variant="elevated" size="small" class="me-1 mb-1 text-white bg-cyan-500">
          {{ meta?.resource_team }}
        </VChip>
        <VChip
          v-if="torrent?.downloadvolumefactor !== 1 || torrent?.uploadvolumefactor !== 1"
          :class="getVolumeFactorClass(torrent?.downloadvolumefactor, torrent?.uploadvolumefactor)"
          variant="elevated"
          size="small"
          class="me-1 mb-1"
        >
          {{ torrent?.volume_factor }}
        </VChip>
      </VCardItem>
      <VCardActions>
        <VBtn v-if="props.more && props.more.length > 0" @click.stop="showMoreTorrents = !showMoreTorrents">
          <template #append>
            <VIcon :icon="showMoreTorrents ? 'mdi-chevron-up' : 'mdi-chevron-down'" />
          </template>
          更多来源
        </VBtn>
      </VCardActions>
      <VExpandTransition>
        <div v-show="showMoreTorrents">
          <VDivider />
          <VChipGroup class="p-3" column>
            <VChip v-for="(item, index) in props.more" :key="index" @click.stop="handleAddDownload(item)">
              <template #append>
                <VBadge color="primary" :content="`↑${item.torrent_info?.seeders}`" inline size="small" />
                <VBadge
                  v-if="item.torrent_info?.downloadvolumefactor !== 1 || item.torrent_info?.uploadvolumefactor !== 1"
                  :content="item.torrent_info?.volume_factor"
                  inline
                  size="small"
                />
              </template>
              {{ item.torrent_info.site_name }}
            </VChip>
          </VChipGroup>
        </div>
      </VExpandTransition>
    </VCard>
    <AddDownloadDialog
      v-if="addDownloadDialog"
      v-model="addDownloadDialog"
      :title="`${downloadItem?.media_info?.title_year || downloadItem?.meta_info?.name} ${
        downloadItem?.meta_info?.season_episode
      }`"
      :media="downloadItem?.media_info"
      :torrent="downloadItem?.torrent_info"
      @done="addDownloadSuccess"
      @error="addDownloadError"
      @close="addDownloadDialog = false"
    />
  </div>
</template>
