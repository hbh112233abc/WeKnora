<template>
  <div class="embed-graph-page">
    <div v-if="loadError" class="embed-graph-error">{{ loadError }}</div>
    <template v-else-if="config">
      <!-- 直接复用 WikiBrowser 的 wiki 链接图谱渲染（含 ego 下钻 / bloom 展开 / 缩放平移 / 搜索 / 邻接高亮 / 按类型着色 / 详情抽屉等全部交互） -->
      <WikiBrowser
        :view="'graph'"
        :embed="true"
        :knowledge-base-id="firstKnowledgeBaseId"
        :graph-fetcher="graphFetcher"
        :page-fetcher="pageFetcher"
      />
    </template>
    <div v-else-if="awaitingToken" class="embed-graph-loading">
      {{ $t('embedPublish.awaitingToken') }}
    </div>
    <div v-else-if="bootstrapping" class="embed-graph-loading">
      {{ $t('embedPublish.loading') }}
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { useRoute } from 'vue-router'
import WikiBrowser from '@/views/knowledge/wiki/WikiBrowser.vue'
import {
  getEmbedConfig,
  onEmbedHostToken,
  parseEmbedTokenFromLocation,
  postEmbedBootstrapRequest,
  getEmbedWikiGraph,
  getEmbedWikiPage,
} from '@/api/embed'
import type { EmbedChannelPublicConfig } from '@/api/embed'
import type { WikiGraphQueryParams } from '@/api/wiki'

const route = useRoute()
const channelId = computed(() => route.params.channelId as string)
const config = ref<EmbedChannelPublicConfig | null>(null)

const awaitingToken = ref(true)
const bootstrapping = ref(false)
const loadError = ref<string | null>(null)
let removed = false

// 嵌入通道绑定多个知识库时，取第一个作为占位 knowledgeBaseId（图谱/页面拉取统一走下方注入的 fetcher）
const firstKnowledgeBaseId = computed(() => config.value?.knowledge_base_ids?.[0] ?? '')

const token = ref('')

// 通过注入的 fetcher 适配 embed 鉴权，复用 WikiBrowser 的 wiki 链接图谱渲染与交互
const graphFetcher = (params: WikiGraphQueryParams) =>
  getEmbedWikiGraph(channelId.value, token.value, params)
const pageFetcher = (slug: string) => getEmbedWikiPage(channelId.value, token.value, slug)

// bootstrap 接收 token 字符串（与 useEmbedBridge 一致），加载 embed 通道公共配置。
// 图谱为只读展示，无需创建聊天会话，故不交换/使用会话 token。
async function bootstrap(embedToken: string) {
  if (removed || !embedToken || embedToken === 'undefined') return
  awaitingToken.value = false
  bootstrapping.value = true
  loadError.value = null
  token.value = embedToken
  try {
    const res = await getEmbedConfig(channelId.value, embedToken)
    if (removed) return
    if (!res?.success || !res.data) {
      loadError.value = 'embed channel config unavailable'
      return
    }
    config.value = res.data
  } catch (e: any) {
    loadError.value = e?.message || String(e)
  } finally {
    bootstrapping.value = false
  }
}

const removeHostListener = onEmbedHostToken((providedToken: string, providedChannelId?: string) => {
  if (providedChannelId && providedChannelId !== channelId.value) return
  void bootstrap(providedToken)
})

onMounted(() => {
  // 与同级嵌入页（useEmbedBridge.start）保持一致：优先取路由 query.token，再回退到 iframe/src 中的 #token= 片段
  const initialToken =
    String((route.query.token as string) || '').trim() || parseEmbedTokenFromLocation()
  if (initialToken) {
    void bootstrap(initialToken)
  } else if (window.parent !== window) {
    // 被父页面以 iframe 嵌入且未带 token，等待宿主通过 postMessage 下发
    awaitingToken.value = true
    void postEmbedBootstrapRequest(channelId.value)
  } else {
    loadError.value = 'missing embed token'
  }
})

onUnmounted(() => {
  removed = true
  removeHostListener()
})
</script>

<style scoped>
.embed-graph-page {
  height: 100vh;
  width: 100%;
  overflow: hidden;
  background: var(--td-bg-color-page, #fff);
}
.embed-graph-loading,
.embed-graph-error {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  padding: 24px;
  font-size: 14px;
  text-align: center;
  color: var(--td-text-color-secondary, #666);
}
.embed-graph-error {
  color: var(--td-error-color, #e34d59);
}
</style>
