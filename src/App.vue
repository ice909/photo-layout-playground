<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import { useElementSize, useEventListener, useVirtualList } from '@vueuse/core'

interface PhotoAsset {
  id: string
  width: number
  height: number
  src: string
  alt: string
  objectPosition: string
}

interface TimelineSection {
  id: string
  title: string
  photos: PhotoAsset[]
}

interface QueuedPhoto {
  asset: PhotoAsset
  inlineTitle?: string
}

interface RowPhoto extends PhotoAsset {
  frameWidth: number
  frameHeight: number
  gapAfter: number
  inlineTitle?: string
}

interface JustifiedRow {
  id: string
  height: number
  topReserve: number
  items: RowPhoto[]
}

interface TimelineTitleBlock {
  kind: 'title'
  id: string
  title: string
  count: number
}

interface TimelineRowBlock {
  kind: 'row'
  id: string
  row: JustifiedRow
}

type TimelineBlock = TimelineTitleBlock | TimelineRowBlock

const ROW_GAP = 2
const MERGE_SEPARATOR_GAP = 10
const TITLE_BLOCK_HEIGHT = 64
const INLINE_TITLE_SPACE = 24
const CONTAINER_INSET = 20
const OVERSCAN_ROWS = 6
const MIN_TILE_RATIO = 0.62
const MAX_TILE_RATIO = 2.25

const DIMENSION_PRESETS: ReadonlyArray<readonly [number, number]> = [
  [540, 360],
  [640, 430],
  [760, 470],
  [430, 640],
  [500, 760],
  [1200, 640],
  [900, 420],
  [360, 540],
  [420, 420],
  [1080, 720],
  [720, 1080],
  [1600, 840],
  [840, 1600],
  [1024, 512],
  [512, 1024],
]

const SCALE_FACTORS = [0.88, 1, 1.1, 1.24, 1.36]

const TIMELINE_TOTAL = 1000
const TIMELINE_SECTION_TOTAL = 36
const WEEKDAY_ZH = ['周日', '周一', '周二', '周三', '周四', '周五', '周六'] as const

const TIMELINE_SPECS: ReadonlyArray<{ id: string, title: string, count: number }> = (() => {
  const counts = Array.from({ length: TIMELINE_SECTION_TOTAL }, (_, index) => {
    if (index % 9 === 2 || index % 11 === 5)
      return 3 + (index % 4)

    return 24 + ((index * 7) % 12)
  })

  let sum = counts.reduce((total, count) => total + count, 0)
  let cursor = 0

  while (sum < TIMELINE_TOTAL) {
    counts[cursor % counts.length] += 1
    sum += 1
    cursor += 1
  }

  while (sum > TIMELINE_TOTAL) {
    const target = cursor % counts.length
    if (counts[target] > 2) {
      counts[target] -= 1
      sum -= 1
    }
    cursor += 1
  }

  const startDate = new Date('2026-03-22T00:00:00')

  return counts.map((count, index) => {
    const date = new Date(startDate)
    date.setDate(startDate.getDate() - index * 2)

    const yyyy = date.getFullYear()
    const mm = `${date.getMonth() + 1}`.padStart(2, '0')
    const dd = `${date.getDate()}`.padStart(2, '0')
    const id = `${yyyy}-${mm}-${dd}`
    const title = `${id} ${WEEKDAY_ZH[date.getDay()]}`

    return { id, title, count }
  })
})()

const PHOTO_TOTAL = TIMELINE_TOTAL

const photos: PhotoAsset[] = Array.from({ length: PHOTO_TOTAL }, (_, index) => {
  const [baseWidth, baseHeight] = DIMENSION_PRESETS[index % DIMENSION_PRESETS.length]
  const scale = SCALE_FACTORS[(index * 7) % SCALE_FACTORS.length]
  const width = Math.round(baseWidth * scale)
  const height = Math.round(baseHeight * (0.9 + ((index * 11) % 6) * 0.05))

  return {
    id: `photo-${index + 1}`,
    width,
    height,
    src: `https://picsum.photos/seed/virtual-justified-${index + 1}/${width}/${height}`,
    alt: `时间线样片 ${index + 1}`,
    objectPosition: `${20 + ((index * 13) % 64)}% ${22 + ((index * 17) % 58)}%`,
  }
})

const timelineSections: TimelineSection[] = (() => {
  let cursor = 0

  return TIMELINE_SPECS.map(spec => {
    const sectionPhotos = photos.slice(cursor, cursor + spec.count)
    cursor += spec.count

    return {
      id: spec.id,
      title: spec.title,
      photos: sectionPhotos,
    }
  })
})()

const clamp = (value: number, min: number, max: number) => Math.min(max, Math.max(min, value))

const normalizeRatio = (photo: PhotoAsset) =>
  clamp(photo.width / photo.height, MIN_TILE_RATIO, MAX_TILE_RATIO)

function resolveGapAfter(total: number, index: number, splitIndex?: number) {
  if (index >= total - 1)
    return 0

  if (splitIndex !== undefined && index === splitIndex - 1)
    return MERGE_SEPARATOR_GAP

  return ROW_GAP
}

function sumGaps(total: number, splitIndex?: number) {
  let totalGap = 0
  for (let index = 0; index < total - 1; index += 1) {
    totalGap += resolveGapAfter(total, index, splitIndex)
  }
  return totalGap
}

function normalizeWidths(widths: number[], targetWidth?: number) {
  const output = widths.map(width => Math.max(1, Math.floor(width)))

  if (targetWidth === undefined)
    return output

  let delta = targetWidth - output.reduce((total, width) => total + width, 0)
  let cursor = 0

  while (delta > 0 && output.length > 0) {
    output[cursor % output.length] += 1
    delta -= 1
    cursor += 1
  }

  while (delta < 0 && output.length > 0) {
    const index = cursor % output.length
    if (output[index] > 1) {
      output[index] -= 1
      delta += 1
    }
    cursor += 1
  }

  return output
}

function estimateRowWidth(items: QueuedPhoto[], targetHeight: number, splitIndex?: number) {
  return items.reduce((total, item) => total + normalizeRatio(item.asset) * targetHeight, 0)
    + sumGaps(items.length, splitIndex)
}

function buildRow(
  rowItems: QueuedPhoto[],
  rowId: string,
  availableWidth: number,
  targetRowHeight: number,
  minLastRowHeight: number,
  isLastRow: boolean,
  splitIndex?: number,
): JustifiedRow {
  const gapWidth = sumGaps(rowItems.length, splitIndex)
  const ratioSum = rowItems.reduce((total, item) => total + normalizeRatio(item.asset), 0)
  const exactHeight = (availableWidth - gapWidth) / ratioSum
  const rowHeight = isLastRow
    ? clamp(Math.min(targetRowHeight, exactHeight), minLastRowHeight, targetRowHeight)
    : exactHeight

  const frameWidths = normalizeWidths(
    rowItems.map(item => normalizeRatio(item.asset) * rowHeight),
    isLastRow ? undefined : Math.round(availableWidth - gapWidth),
  )

  const hasInlineTitle = rowItems.some(item => item.inlineTitle)

  return {
    id: rowId,
    height: Math.round(rowHeight),
    topReserve: hasInlineTitle ? INLINE_TITLE_SPACE : 0,
    items: rowItems.map((item, index) => ({
      ...item.asset,
      frameWidth: frameWidths[index],
      frameHeight: Math.round(rowHeight),
      gapAfter: resolveGapAfter(rowItems.length, index, splitIndex),
      inlineTitle: item.inlineTitle,
    })),
  }
}

const totalPhotoCount = photos.length
const sectionCount = timelineSections.length

const containerWidth = ref(0)
const previewPhoto = ref<PhotoAsset | null>(null)

const openPreview = (photo: RowPhoto) => {
  previewPhoto.value = photo
}

const closePreview = () => {
  previewPhoto.value = null
}

useEventListener(window, 'keydown', (event: KeyboardEvent) => {
  if (event.key === 'Escape' && previewPhoto.value) {
    closePreview()
  }
})

const timelineBlocks = computed<TimelineBlock[]>(() => {
  const rawWidth = containerWidth.value
  const availableWidth = Math.max(rawWidth - CONTAINER_INSET * 2, 280)

  const targetRowHeight = rawWidth > 1100 ? 242 : rawWidth > 760 ? 208 : 156
  const minLastRowHeight = Math.round(targetRowHeight * 0.74)

  const blocks: TimelineBlock[] = []

  for (let sectionIndex = 0; sectionIndex < timelineSections.length; sectionIndex += 1) {
    const section = timelineSections[sectionIndex]

    blocks.push({
      kind: 'title',
      id: `title-${section.id}`,
      title: section.title,
      count: section.photos.length,
    })

    let queue: QueuedPhoto[] = []
    let ratioAccumulator = 0
    let rowIndex = 0

    for (const photo of section.photos) {
      queue.push({ asset: photo })
      ratioAccumulator += normalizeRatio(photo)

      const estimatedWidth = ratioAccumulator * targetRowHeight + sumGaps(queue.length)
      if (estimatedWidth < availableWidth)
        continue

      blocks.push({
        kind: 'row',
        id: `row-${section.id}-${rowIndex + 1}`,
        row: buildRow(
          queue,
          `row-${section.id}-${rowIndex + 1}`,
          availableWidth,
          targetRowHeight,
          minLastRowHeight,
          false,
        ),
      })

      rowIndex += 1
      queue = []
      ratioAccumulator = 0
    }

    if (queue.length === 0)
      continue

    const nextSection = timelineSections[sectionIndex + 1]
    if (!nextSection) {
      blocks.push({
        kind: 'row',
        id: `row-${section.id}-${rowIndex + 1}`,
        row: buildRow(
          queue,
          `row-${section.id}-${rowIndex + 1}`,
          availableWidth,
          targetRowHeight,
          minLastRowHeight,
          true,
        ),
      })
      continue
    }

    const isFirstRowOfSection
      = queue[0]?.asset.id !== undefined
        && queue[0].asset.id === section.photos[0]?.id

    if (!isFirstRowOfSection) {
      blocks.push({
        kind: 'row',
        id: `row-${section.id}-${rowIndex + 1}`,
        row: buildRow(
          queue,
          `row-${section.id}-${rowIndex + 1}`,
          availableWidth,
          targetRowHeight,
          minLastRowHeight,
          true,
        ),
      })
      continue
    }

    const splitIndex = queue.length
    const bridgeItems: QueuedPhoto[] = [
      ...queue,
      ...nextSection.photos.map((asset, index) => ({
        asset,
        inlineTitle: index === 0 ? nextSection.title : undefined,
      })),
    ]

    const bridgeEstimate = estimateRowWidth(bridgeItems, targetRowHeight, splitIndex)

    if (bridgeEstimate <= availableWidth) {
      blocks.push({
        kind: 'row',
        id: `row-${section.id}-${rowIndex + 1}-bridge-${nextSection.id}`,
        row: buildRow(
          bridgeItems,
          `row-${section.id}-${rowIndex + 1}-bridge-${nextSection.id}`,
          availableWidth,
          targetRowHeight,
          minLastRowHeight,
          true,
          splitIndex,
        ),
      })

      sectionIndex += 1
      continue
    }

    blocks.push({
      kind: 'row',
      id: `row-${section.id}-${rowIndex + 1}`,
      row: buildRow(
        queue,
        `row-${section.id}-${rowIndex + 1}`,
        availableWidth,
        targetRowHeight,
        minLastRowHeight,
        true,
      ),
    })
  }

  return blocks
})

const { list: virtualRows, containerProps, wrapperProps, scrollTo } = useVirtualList(
  timelineBlocks,
  {
    itemHeight: index => {
      const block = timelineBlocks.value[index]
      if (!block)
        return TITLE_BLOCK_HEIGHT

      return block.kind === 'title'
        ? TITLE_BLOCK_HEIGHT
        : block.row.height + block.row.topReserve + ROW_GAP
    },
    overscan: OVERSCAN_ROWS,
  },
)

const { width: viewportWidth } = useElementSize(containerProps.ref)
watch(viewportWidth, width => {
  containerWidth.value = width
}, { immediate: true })

const rowCount = computed(
  () => timelineBlocks.value.filter(block => block.kind === 'row').length,
)
</script>

<template>
  <main class="layout-shell">
    <header class="layout-head">
      <p class="eyebrow">Virtualized Justified Layout</p>
      <h1>useVirtualList + 时间线分段视图</h1>
      <p class="description">
        当前规则为“整段补齐”：只有当下一时间段图片能完整放进当前尾行时才合并，且分段处间距更大。
      </p>
      <div class="stats">
        <span>{{ totalPhotoCount }} 张图</span>
        <span>{{ sectionCount }} 个时间段</span>
        <span>{{ rowCount }} 行</span>
        <button type="button" @click="scrollTo(0)">
          回到顶部
        </button>
      </div>
    </header>

    <section v-bind="containerProps" class="virtual-viewport">
      <div v-bind="wrapperProps">
        <template v-for="{ data: block } in virtualRows" :key="block.id">
          <header v-if="block.kind === 'title'" class="timeline-title">
            <div class="timeline-dot" />
            <div class="timeline-text">
              <h2>{{ block.title }}</h2>
              <p>{{ block.count }} 张</p>
            </div>
          </header>

          <article
            v-else
            class="photo-row"
            :style="{ height: `${block.row.height + block.row.topReserve}px` }"
          >
            <figure
              v-for="photo in block.row.items"
              :key="photo.id"
              class="tile"
              role="button"
              tabindex="0"
              :style="{
                width: `${photo.frameWidth}px`,
                height: `${photo.frameHeight}px`,
                marginRight: `${photo.gapAfter}px`,
              }"
              @click="openPreview(photo)"
              @keydown.enter.prevent="openPreview(photo)"
              @keydown.space.prevent="openPreview(photo)"
            >
              <span v-if="photo.inlineTitle" class="inline-segment-title">{{ photo.inlineTitle }}</span>
              <img
                :src="photo.src"
                :alt="photo.alt"
                :style="{ objectPosition: photo.objectPosition }"
                loading="lazy"
                decoding="async"
              >
              <figcaption>{{ photo.width }} × {{ photo.height }}</figcaption>
            </figure>
          </article>
        </template>
      </div>
    </section>

    <div
      v-if="previewPhoto"
      class="preview-layer"
      role="dialog"
      aria-modal="true"
      @click.self="closePreview"
    >
      <button type="button" class="preview-close" @click="closePreview">
        关闭
      </button>
      <img
        class="preview-image"
        :src="previewPhoto.src"
        :alt="previewPhoto.alt"
      >
      <p class="preview-meta">
        {{ previewPhoto.alt }} · {{ previewPhoto.width }} × {{ previewPhoto.height }}
      </p>
    </div>
  </main>
</template>

<style scoped>
.layout-shell {
  --ink: #13212d;
  --ink-muted: #4f6072;
  --card: rgba(255, 255, 255, 0.72);
  --stroke: rgba(18, 37, 59, 0.16);
  --hot: #0f766e;
  height: 100svh;
  min-height: 100svh;
  display: grid;
  grid-template-rows: auto minmax(0, 1fr);
  background:
    radial-gradient(circle at 88% 12%, rgba(56, 189, 248, 0.22), transparent 42%),
    radial-gradient(circle at 14% 10%, rgba(245, 158, 11, 0.2), transparent 36%),
    linear-gradient(165deg, #f8fafc 0%, #eef6ff 48%, #e9f8f1 100%);
  color: var(--ink);
}

.layout-head {
  padding: 24px 22px 14px;
  border-bottom: 1px solid var(--stroke);
  backdrop-filter: blur(8px);
}

.eyebrow {
  margin: 0;
  letter-spacing: 0.13em;
  text-transform: uppercase;
  font-size: 11px;
  color: var(--ink-muted);
}

h1 {
  margin: 10px 0 8px;
  font-size: clamp(22px, 3.2vw, 34px);
  line-height: 1.04;
}

.description {
  margin: 0;
  max-width: 76ch;
  color: var(--ink-muted);
  font-size: 14px;
}

.stats {
  margin-top: 14px;
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 8px;
}

.stats span,
.stats button {
  border: 1px solid var(--stroke);
  border-radius: 999px;
  padding: 6px 12px;
  background: var(--card);
  font-size: 12px;
}

.stats button {
  font: inherit;
  color: inherit;
  cursor: pointer;
}

.stats button:hover {
  border-color: color-mix(in srgb, var(--hot) 50%, var(--stroke));
}

.virtual-viewport {
  height: 100%;
  min-height: 0;
  padding: 8px 20px 0;
  overflow-y: auto;
}

.timeline-title {
  height: 64px;
  margin: 0;
  display: flex;
  align-items: center;
  gap: 10px;
  position: relative;
}

.timeline-title::after {
  content: '';
  position: absolute;
  left: 0;
  right: 0;
  bottom: 8px;
  border-bottom: 1px dashed rgba(20, 37, 56, 0.14);
}

.timeline-dot {
  width: 10px;
  height: 10px;
  border-radius: 999px;
  background: linear-gradient(135deg, #0f766e, #14b8a6);
  box-shadow: 0 0 0 4px rgba(20, 184, 166, 0.18);
  z-index: 1;
}

.timeline-text {
  display: flex;
  align-items: baseline;
  gap: 8px;
  z-index: 1;
  background: linear-gradient(180deg, rgba(244, 250, 255, 0.92), rgba(244, 250, 255, 0.65));
  padding-right: 8px;
}

.timeline-text h2 {
  margin: 0;
  font-size: 14px;
  letter-spacing: 0.02em;
}

.timeline-text p {
  margin: 0;
  font-size: 11px;
  color: var(--ink-muted);
}

.photo-row {
  display: flex;
  align-items: flex-end;
  margin-bottom: 2px;
}

.tile {
  position: relative;
  margin: 0;
  border-radius: 0;
  overflow: visible;
  border: 1px solid rgba(20, 37, 56, 0.08);
  box-shadow: 0 6px 20px rgba(10, 25, 41, 0.16);
  background: linear-gradient(135deg, #dbe7f3 0%, #f8fbff 100%);
  cursor: zoom-in;
}

.inline-segment-title {
  position: absolute;
  left: 0;
  top: -20px;
  height: 18px;
  display: inline-flex;
  align-items: center;
  padding: 0 8px;
  font-size: 10px;
  letter-spacing: 0.04em;
  color: #d7f7f1;
  background: rgba(6, 78, 59, 0.9);
  white-space: nowrap;
}

.tile img {
  width: 100%;
  height: 100%;
  display: block;
  object-fit: cover;
  border-radius: 0;
}

.tile figcaption {
  position: absolute;
  right: 8px;
  bottom: 8px;
  font-size: 10px;
  letter-spacing: 0.04em;
  color: #f8fafc;
  background: rgba(15, 23, 42, 0.68);
  border-radius: 999px;
  padding: 4px 8px;
}

.preview-layer {
  position: fixed;
  inset: 0;
  z-index: 100;
  display: grid;
  place-items: center;
  background: rgba(3, 7, 18, 0.86);
  backdrop-filter: blur(4px);
  padding: 0;
}

.preview-close {
  position: absolute;
  top: 20px;
  right: 20px;
  border: 1px solid rgba(148, 163, 184, 0.5);
  background: rgba(15, 23, 42, 0.82);
  color: #f8fafc;
  padding: 8px 12px;
  font: inherit;
  cursor: pointer;
}

.preview-image {
  width: 100vw;
  height: 100svh;
  object-fit: contain;
  box-shadow: 0 10px 48px rgba(0, 0, 0, 0.34);
}

.preview-meta {
  position: absolute;
  left: 50%;
  bottom: 16px;
  transform: translateX(-50%);
  margin: 0;
  color: #cbd5e1;
  font-size: 12px;
  letter-spacing: 0.02em;
  background: rgba(15, 23, 42, 0.62);
  padding: 4px 10px;
  border-radius: 999px;
}

@media (max-width: 760px) {
  .layout-head {
    padding: 16px 14px 12px;
  }

  .virtual-viewport {
    padding: 6px 12px 0;
  }

  .inline-segment-title {
    top: -18px;
    font-size: 9px;
    padding: 0 6px;
  }

  .photo-row {
    margin-bottom: 2px;
  }
}
</style>
