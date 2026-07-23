<template>
  <div class="embed-graph-page" :style="pageStyle">
    <div v-if="loadError" class="embed-graph-error">{{ loadError }}</div>
    <template v-else-if="config">
      <header class="embed-graph-header">
        <span class="embed-graph-header__badge" :style="badgeStyle">
          <t-icon name="share-nodes" size="18px" />
        </span>
        <div class="embed-graph-header__text">
          <h1 class="embed-graph-header__title">{{ headerTitle }}</h1>
          <p class="embed-graph-header__subtitle">
            {{ $t('embedGraph.subtitle', { nodes: nodeCount, relations: relationCount }) }}
          </p>
        </div>
        <div class="embed-graph-header__actions">
          <t-input
            v-model="searchKeyword"
            :placeholder="$t('embedGraph.searchPlaceholder')"
            clearable
            size="small"
            class="embed-graph-header__search"
          >
            <template #prefix-icon>
              <t-icon name="search" />
            </template>
          </t-input>
          <t-select
            v-model="layoutMode"
            size="small"
            class="embed-graph-header__layout"
          >
            <t-option value="force" :label="$t('embedGraph.layoutForce')" />
            <t-option value="circular" :label="$t('embedGraph.layoutCircular')" />
          </t-select>
          <t-button
            variant="outline"
            size="small"
            :title="$t('embedGraph.reset')"
            @click="resetLayout"
          >
            <template #icon><t-icon name="refresh" /></template>
          </t-button>
        </div>
      </header>

      <div class="embed-graph-body">
        <div v-if="loading" class="embed-graph-loading">{{ $t('embedGraph.loading') }}</div>
        <div v-else-if="!graphData || nodeCount === 0" class="embed-graph-empty">
          {{ $t('embedGraph.empty') }}
        </div>
        <svg
          v-else
          ref="svgRef"
          class="embed-graph-canvas"
          :viewBox="`0 0 ${svgWidth} ${svgHeight}`"
          @mousedown="onCanvasMouseDown"
        >
          <defs>
            <marker
              id="arrowhead"
              markerWidth="10"
              markerHeight="7"
              refX="18"
              refY="3.5"
              orient="auto"
            >
              <polygon points="0 0, 10 3.5, 0 7" fill="var(--embed-graph-edge, #bbb)" />
            </marker>
          </defs>
          <!-- 边 -->
          <g class="embed-graph-edges">
            <line
              v-for="(edge, i) in renderedEdges"
              :key="`e${i}`"
              :x1="edge.x1"
              :y1="edge.y1"
              :x2="edge.x2"
              :y2="edge.y2"
              :stroke="selectedNode && (edge.source === selectedNode || edge.target === selectedNode) ? 'var(--td-brand-color)' : 'var(--embed-graph-edge, #ccc)'"
              :stroke-width="selectedNode && (edge.source === selectedNode || edge.target === selectedNode) ? 2 : 1"
              :marker-end="edge.source === selectedNode ? 'url(#arrowhead)' : ''"
              class="embed-graph-edge"
            />
          </g>
          <!-- 节点 -->
          <g class="embed-graph-nodes">
            <g
              v-for="node in renderedNodes"
              :key="node.name"
              :transform="`translate(${node.x},${node.y})`"
              class="embed-graph-node-group"
              :class="{ 'embed-graph-node-group--selected': node.name === selectedNode }"
              @click.stop="selectNode(node.name)"
              @mousedown.stop="startDrag($event, node)"
            >
              <circle
                :r="nodeRadius(node)"
                :fill="selectedNode === node.name ? 'var(--td-brand-color)' : 'var(--embed-graph-node-fill, #5b8def)'"
                :stroke="selectedNode === node.name ? 'var(--td-brand-color)' : 'var(--embed-graph-node-stroke, #3b6fd6)'"
                stroke-width="2"
                class="embed-graph-node-circle"
              />
              <text
                :y="nodeRadius(node) + 14"
                text-anchor="middle"
                class="embed-graph-node-label"
                :class="{ 'embed-graph-node-label--highlighted': node.name === selectedNode }"
              >{{ node.label }}</text>
            </g>
          </g>
        </svg>

        <!-- 节点详情面板 -->
        <div v-if="selectedNodeData" class="embed-graph-detail">
          <div class="embed-graph-detail__header">
            <span class="embed-graph-detail__title">{{ selectedNodeData.name }}</span>
            <t-button
              variant="text"
              shape="square"
              size="small"
              @click="selectedNode = ''"
            >
              <template #icon><t-icon name="close" /></template>
            </t-button>
          </div>
          <div class="embed-graph-detail__body">
            <div v-if="selectedNodeData.attributes && selectedNodeData.attributes.length" class="embed-graph-detail__section">
              <p class="embed-graph-detail__label">{{ $t('embedGraph.attributes') }}</p>
              <div class="embed-graph-detail__tags">
                <span v-for="attr in selectedNodeData.attributes" :key="attr" class="embed-graph-detail__tag">{{ attr }}</span>
              </div>
            </div>
            <div v-if="selectedNodeRelations.length" class="embed-graph-detail__section">
              <p class="embed-graph-detail__label">{{ $t('embedGraph.relations') }}</p>
              <div class="embed-graph-detail__relations">
                <div
                  v-for="(rel, i) in selectedNodeRelations"
                  :key="`r${i}`"
                  class="embed-graph-detail__relation"
                >
                  <span class="embed-graph-detail__relation-type">{{ rel.type }}</span>
                  <span class="embed-graph-detail__relation-target">{{ rel.otherNode }}</span>
                </div>
              </div>
            </div>
            <div v-if="selectedNodeData.chunks && selectedNodeData.chunks.length" class="embed-graph-detail__section">
              <p class="embed-graph-detail__label">{{ $t('embedGraph.chunks', { count: selectedNodeData.chunks.length }) }}</p>
            </div>
          </div>
        </div>
      </div>
    </template>
    <div v-else-if="awaitingToken" class="embed-graph-loading">{{ $t('embedPublish.awaitingToken') }}</div>
    <div v-else-if="bootstrapping" class="embed-graph-loading">{{ $t('embedPublish.loading') }}</div>
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'
import { useRoute } from 'vue-router'
import { useI18n } from 'vue-i18n'
import {
  exchangeEmbedSession,
  getEmbedConfig,
  getEmbedKnowledgeGraph,
  isEmbedSessionToken,
  onEmbedHostToken,
  parseEmbedTokenFromLocation,
  postEmbedBootstrapRequest,
  type EmbedChannelPublicConfig,
  type EmbedGraphData,
  type EmbedGraphNode,
} from '@/api/embed'

const { t } = useI18n()
const route = useRoute()

const channelId = ref(String(route.params.channelId || ''))

const token = ref('')
const config = ref<EmbedChannelPublicConfig | null>(null)
const loadError = ref('')
const awaitingToken = ref(false)
const bootstrapping = ref(false)
const loading = ref(true)
const graphData = ref<EmbedGraphData | null>(null)

const searchKeyword = ref('')
const layoutMode = ref<'force' | 'circular'>('force')
const selectedNode = ref('')

const svgRef = ref<SVGSVGElement | null>(null)
const svgWidth = 1200
const svgHeight = 800

let bootstrapped = false
let removeTokenListener: (() => void) | null = null
let animationId = 0
let simulationRunning = false

// 布局节点位置
interface LayoutNode {
  name: string
  label: string
  x: number
  y: number
  vx: number
  vy: number
  degree: number
}

let layoutNodes: LayoutNode[] = []

const bootstrap = async (embedToken: string) => {
  const id = channelId.value
  if (!id || !embedToken || bootstrapped) return
  bootstrapped = true
  awaitingToken.value = false
  bootstrapping.value = true
  token.value = embedToken

  try {
    let apiToken = embedToken
    if (!isEmbedSessionToken(embedToken)) {
      try {
        const exchangeRes = await exchangeEmbedSession(id, embedToken)
        if (exchangeRes?.data?.session_token) {
          apiToken = exchangeRes.data.session_token
        } else if (!import.meta.env.DEV) {
          throw new Error('embed session exchange returned no token')
        }
      } catch (exchangeErr) {
        if (!import.meta.env.DEV) {
          throw exchangeErr
        }
      }
    }

    const res = await getEmbedConfig(id, apiToken)
    if (!res?.success || !res.data) {
      loadError.value = t('embedPublish.invalidChannel')
      return
    }
    config.value = res.data
    token.value = apiToken

    // 加载知识图谱数据
    const graphRes = await getEmbedKnowledgeGraph(id, apiToken)
    if (graphRes?.success && graphRes.data) {
      graphData.value = graphRes.data
    }
    initLayout()
    startSimulation()
  } catch (e: unknown) {
    bootstrapped = false
    const msg = String((e as { message?: string })?.message || '')
    if (msg.includes('disabled')) {
      loadError.value = t('embedPublish.channelDisabled')
    } else {
      loadError.value = msg || t('embedPublish.loadError')
    }
  } finally {
    bootstrapping.value = false
    loading.value = false
  }
}

const start = async () => {
  removeTokenListener = onEmbedHostToken((providedToken, providedChannelId) => {
    if (providedChannelId && providedChannelId !== channelId.value) return
    bootstrap(providedToken)
  })

  if (!channelId.value) {
    loadError.value = t('embedPublish.missingChannel')
    return
  }

  const initialToken = String(route.query.token || '') || parseEmbedTokenFromLocation()
  if (initialToken) {
    await bootstrap(initialToken)
    return
  }

  if (window.parent !== window) {
    awaitingToken.value = true
    postEmbedBootstrapRequest(channelId.value)
    return
  }

  loadError.value = t('embedPublish.missingChannel')
}

// 计算节点和边的渲染数据
const nodeCount = computed(() => graphData.value?.node?.length ?? 0)
const relationCount = computed(() => graphData.value?.relation?.length ?? 0)

const filteredNodes = computed<EmbedGraphNode[]>(() => {
  const nodes = graphData.value?.node ?? []
  const kw = searchKeyword.value.trim().toLowerCase()
  if (!kw) return nodes
  return nodes.filter((n) => n.name.toLowerCase().includes(kw))
})

const filteredNodeNames = computed(() => new Set(filteredNodes.value.map((n) => n.name)))

const filteredEdges = computed(() => {
  const edges = graphData.value?.relation ?? []
  const names = filteredNodeNames.value
  if (!searchKeyword.value.trim()) return edges
  return edges.filter((e) => names.has(e.node1) && names.has(e.node2))
})

const renderedNodes = computed(() => {
  return layoutNodes.filter((n) => filteredNodeNames.value.has(n.name))
})

const renderedEdges = computed(() => {
  const namePos = new Map(layoutNodes.map((n) => [n.name, n]))
  return filteredEdges.value.map((edge) => {
    const s = namePos.get(edge.node1)
    const tgt = namePos.get(edge.node2)
    return {
      source: edge.node1,
      target: edge.node2,
      x1: s?.x ?? 0,
      y1: s?.y ?? 0,
      x2: tgt?.x ?? 0,
      y2: tgt?.y ?? 0,
    }
  })
})

const selectedNodeData = computed<EmbedGraphNode | null>(() => {
  if (!selectedNode.value) return null
  const nodes = graphData.value?.node ?? []
  return nodes.find((n) => n.name === selectedNode.value) ?? null
})

const selectedNodeRelations = computed(() => {
  if (!selectedNode.value) return []
  const edges = graphData.value?.relation ?? []
  const result: { type: string; otherNode: string }[] = []
  for (const e of edges) {
    if (e.node1 === selectedNode.value) {
      result.push({ type: e.type, otherNode: e.node2 })
    } else if (e.node2 === selectedNode.value) {
      result.push({ type: e.type, otherNode: e.node1 })
    }
  }
  return result
})

// 布局初始化
function initLayout() {
  const nodes = graphData.value?.node ?? []
  const edges = graphData.value?.relation ?? []

  // 计算节点度
  const degreeMap = new Map<string, number>()
  for (const n of nodes) {
    degreeMap.set(n.name, 0)
  }
  for (const e of edges) {
    degreeMap.set(e.node1, (degreeMap.get(e.node1) ?? 0) + 1)
    degreeMap.set(e.node2, (degreeMap.get(e.node2) ?? 0) + 1)
  }

  if (layoutMode.value === 'circular') {
    initCircularLayout(nodes, degreeMap)
  } else {
    initForceLayout(nodes, degreeMap)
  }
}

function initForceLayout(
  nodes: EmbedGraphNode[],
  degreeMap: Map<string, number>,
) {
  layoutNodes = nodes.map((n, i) => {
    const angle = (i / Math.max(nodes.length, 1)) * Math.PI * 2
    const r = Math.min(svgWidth, svgHeight) * 0.3
    return {
      name: n.name,
      label: n.name.length > 12 ? n.name.slice(0, 11) + '…' : n.name,
      x: svgWidth / 2 + r * Math.cos(angle) + (Math.random() - 0.5) * 20,
      y: svgHeight / 2 + r * Math.sin(angle) + (Math.random() - 0.5) * 20,
      vx: 0,
      vy: 0,
      degree: degreeMap.get(n.name) ?? 0,
    }
  })
}

function initCircularLayout(
  nodes: EmbedGraphNode[],
  degreeMap: Map<string, number>,
) {
  const cx = svgWidth / 2
  const cy = svgHeight / 2
  const r = Math.min(svgWidth, svgHeight) * 0.35
  layoutNodes = nodes.map((n, i) => {
    const angle = (i / Math.max(nodes.length, 1)) * Math.PI * 2
    return {
      name: n.name,
      label: n.name.length > 12 ? n.name.slice(0, 11) + '…' : n.name,
      x: cx + r * Math.cos(angle),
      y: cy + r * Math.sin(angle),
      vx: 0,
      vy: 0,
      degree: degreeMap.get(n.name) ?? 0,
    }
  })
}

function startSimulation() {
  if (layoutMode.value !== 'force' || layoutNodes.length === 0) return
  simulationRunning = true
  runSimulation()
}

function runSimulation() {
  if (!simulationRunning) return
  const edges = graphData.value?.relation ?? []
  const nodeMap = new Map(layoutNodes.map((n) => [n.name, n]))

  const cx = svgWidth / 2
  const cy = svgHeight / 2
  const k = 0.08 // 斥力系数
  const linkDist = 80 // 理想连线长度
  const damping = 0.85
  const gravity = 0.02

  // 斥力
  for (let i = 0; i < layoutNodes.length; i++) {
    for (let j = i + 1; j < layoutNodes.length; j++) {
      const a = layoutNodes[i]
      const b = layoutNodes[j]
      const dx = b.x - a.x
      const dy = b.y - a.y
      const dist = Math.sqrt(dx * dx + dy * dy) || 1
      const force = k * 2000 / (dist * dist)
      const fx = (dx / dist) * force
      const fy = (dy / dist) * force
      a.vx -= fx
      a.vy -= fy
      b.vx += fx
      b.vy += fy
    }
  }

  // 弹簧力
  for (const e of edges) {
    const a = nodeMap.get(e.node1)
    const b = nodeMap.get(e.node2)
    if (!a || !b) continue
    const dx = b.x - a.x
    const dy = b.y - a.y
    const dist = Math.sqrt(dx * dx + dy * dy) || 1
    const force = (dist - linkDist) * 0.04
    const fx = (dx / dist) * force
    const fy = (dy / dist) * force
    a.vx += fx
    a.vy += fy
    b.vx -= fx
    b.vy -= fy
  }

  // 中心引力
  for (const n of layoutNodes) {
    n.vx += (cx - n.x) * gravity
    n.vy += (cy - n.y) * gravity
  }

  // 更新位置
  for (const n of layoutNodes) {
    if (n.name === draggingNodeName.value) continue
    n.vx *= damping
    n.vy *= damping
    n.x += n.vx
    n.y += n.vy
    // 边界约束
    n.x = Math.max(40, Math.min(svgWidth - 40, n.x))
    n.y = Math.max(40, Math.min(svgHeight - 40, n.y))
  }

  animationId = requestAnimationFrame(runSimulation)
}

function stopSimulation() {
  simulationRunning = false
  if (animationId) {
    cancelAnimationFrame(animationId)
    animationId = 0
  }
}

function resetLayout() {
  stopSimulation()
  initLayout()
  if (layoutMode.value === 'force') {
    startSimulation()
  }
}

watch(layoutMode, () => {
  stopSimulation()
  initLayout()
  if (layoutMode.value === 'force') {
    startSimulation()
  }
})

// 节点选择
function selectNode(name: string) {
  selectedNode.value = selectedNode.value === name ? '' : name
}

// 拖拽
const draggingNodeName = ref('')
let dragStartX = 0
let dragStartY = 0
let dragNodeStartX = 0
let dragNodeStartY = 0

function startDrag(ev: MouseEvent, node: LayoutNode) {
  draggingNodeName.value = node.name
  const svg = svgRef.value
  if (!svg) return
  const rect = svg.getBoundingClientRect()
  dragStartX = ev.clientX - rect.left
  dragStartY = ev.clientY - rect.top
  dragNodeStartX = node.x
  dragNodeStartY = node.y
  window.addEventListener('mousemove', onDragMove)
  window.addEventListener('mouseup', onDragEnd)
}

function onDragMove(ev: MouseEvent) {
  const name = draggingNodeName.value
  if (!name) return
  const node = layoutNodes.find((n) => n.name === name)
  if (!node) return
  const svg = svgRef.value
  if (!svg) return
  const rect = svg.getBoundingClientRect()
  const scaleX = svgWidth / rect.width
  const scaleY = svgHeight / rect.height
  node.x = dragNodeStartX + (ev.clientX - dragStartX) * scaleX
  node.y = dragNodeStartY + (ev.clientY - dragStartY) * scaleY
}

function onDragEnd() {
  draggingNodeName.value = ''
  window.removeEventListener('mousemove', onDragMove)
  window.removeEventListener('mouseup', onDragEnd)
}

// 画布点击取消选择
function onCanvasMouseDown() {
  selectedNode.value = ''
}

function nodeRadius(node: LayoutNode): number {
  // 度越高节点越大
  return 6 + Math.min(node.degree * 2, 16)
}

// 样式计算
const pageStyle = computed(() => {
  const color = config.value?.primary_color
  if (!color) return {}
  return {
    '--embed-primary': color,
    '--td-brand-color': color,
    '--td-brand-color-hover': color,
    '--td-brand-color-active': color,
  } as Record<string, string>
})

const badgeStyle = computed(() => {
  const color = config.value?.primary_color
  if (!color) return {}
  return {
    background: `color-mix(in srgb, ${color} 12%, transparent)`,
    color,
  } as Record<string, string>
})

const channelDisplayTitle = computed(() => {
  const cfg = config.value
  if (!cfg) return ''
  return (
    cfg.display_title?.trim()
    || cfg.page_title?.trim()
    || cfg.name?.trim()
    || cfg.agent_name?.trim()
    || t('embedGraph.defaultTitle')
  )
})

const headerTitle = computed(() => channelDisplayTitle.value)

watch(headerTitle, (title) => {
  if (title) document.title = title
}, { immediate: true })

onMounted(() => {
  start()
})

onUnmounted(() => {
  stopSimulation()
  removeTokenListener?.()
  window.removeEventListener('mousemove', onDragMove)
  window.removeEventListener('mouseup', onDragEnd)
})
</script>

<style scoped lang="less">
.embed-graph-page {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: var(--td-bg-color-container, #fff);
  overflow: hidden;
  --td-brand-color: var(--embed-primary, var(--td-brand-color));
  --td-brand-color-hover: var(--embed-primary, var(--td-brand-color-hover));
  --td-brand-color-active: var(--embed-primary, var(--td-brand-color-active));
}

.embed-graph-header {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  border-bottom: 1px solid var(--td-component-stroke);
  background: var(--td-bg-color-container);
  flex-shrink: 0;

  &__badge {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 36px;
    height: 36px;
    border-radius: 10px;
    flex-shrink: 0;
    background: color-mix(in srgb, var(--td-brand-color) 10%, transparent);
    color: var(--td-brand-color);
  }

  &__text {
    min-width: 0;
    flex: 0 1 auto;
  }

  &__title {
    margin: 0;
    font-size: 15px;
    font-weight: 600;
    line-height: 1.35;
    color: var(--td-text-color-primary);
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__subtitle {
    margin: 2px 0 0;
    font-size: 12px;
    line-height: 1.4;
    color: var(--td-text-color-secondary);
    white-space: nowrap;
  }

  &__actions {
    display: flex;
    align-items: center;
    gap: 8px;
    flex: 1;
    justify-content: flex-end;
  }

  &__search {
    width: 180px;
  }

  &__layout {
    width: 120px;
  }
}

.embed-graph-body {
  flex: 1;
  position: relative;
  overflow: hidden;
}

.embed-graph-canvas {
  width: 100%;
  height: 100%;
  cursor: grab;
  background: var(--td-bg-color-page, #f5f5f5);

  &:active {
    cursor: grabbing;
  }
}

.embed-graph-node-group {
  cursor: pointer;

  &--selected {
    .embed-graph-node-circle {
      filter: drop-shadow(0 0 6px var(--td-brand-color));
    }
  }
}

.embed-graph-node-circle {
  transition: r 0.2s ease;
}

.embed-graph-node-label {
  font-size: 11px;
  fill: var(--td-text-color-primary);
  pointer-events: none;
  user-select: none;

  &--highlighted {
    font-weight: 600;
    fill: var(--td-brand-color);
  }
}

.embed-graph-edge {
  transition: stroke 0.15s ease;
}

.embed-graph-loading,
.embed-graph-error,
.embed-graph-empty {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: var(--td-text-color-placeholder);
  font-size: 14px;
  text-align: center;
}

.embed-graph-error {
  color: var(--td-error-color);
}

.embed-graph-detail {
  position: absolute;
  top: 16px;
  right: 16px;
  width: 280px;
  max-height: calc(100% - 32px);
  overflow-y: auto;
  background: var(--td-bg-color-container);
  border: 1px solid var(--td-component-stroke);
  border-radius: 12px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.08);
  z-index: 10;

  &__header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 12px 16px;
    border-bottom: 1px solid var(--td-component-stroke);
  }

  &__title {
    font-size: 14px;
    font-weight: 600;
    color: var(--td-text-color-primary);
    word-break: break-all;
  }

  &__body {
    padding: 12px 16px;
  }

  &__section {
    margin-bottom: 16px;

    &:last-child {
      margin-bottom: 0;
    }
  }

  &__label {
    margin: 0 0 8px;
    font-size: 12px;
    font-weight: 500;
    color: var(--td-text-color-secondary);
  }

  &__tags {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  &__tag {
    display: inline-block;
    padding: 2px 8px;
    font-size: 11px;
    background: color-mix(in srgb, var(--td-brand-color) 8%, transparent);
    color: var(--td-brand-color);
    border-radius: 4px;
  }

  &__relations {
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  &__relation {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 4px 0;
    font-size: 12px;
  }

  &__relation-type {
    color: var(--td-brand-color);
    font-weight: 500;
    white-space: nowrap;
  }

  &__relation-target {
    color: var(--td-text-color-primary);
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
}
</style>
