<script setup>
import 'reveal.js/reveal.css'

import { Close, Present } from '@element-plus/icons-vue'
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { useData, useRoute } from 'vitepress'

const { frontmatter } = useData()
const route = useRoute()

const overlayRef = ref(null)
const revealRef = ref(null)
const slidesRef = ref(null)
const hasDocContent = ref(false)
const isOpen = ref(false)
const slideCount = ref(0)

let deck = null
let previousBodyOverflow = ''
let openedFullscreen = false
let slideRun = 0

const isDocPage = computed(() => {
  const layout = frontmatter.value.layout
  return layout !== 'home' && route.path !== '/welcome/' && !route.path.endsWith('/welcome/')
})

const getDocContent = () => {
  const doc = document.querySelector('.VPDoc .vp-doc')
  if (!doc) return null

  const onlyChild = doc.children.length === 1 ? doc.firstElementChild : null
  if (onlyChild?.querySelector?.('h1, h2, h3')) return onlyChild

  return doc
}

const refreshAvailability = async () => {
  await nextTick()
  hasDocContent.value = Boolean(isDocPage.value && getDocContent()?.querySelector('h1, h2, h3'))
}

const isHeading = (node, level) => node.tagName?.toLowerCase() === `h${level}`

const createSlideSection = (nodes, idMap, slideIndex) => {
  const section = document.createElement('section')
  section.className = 'ev-slide-section'

  const page = document.createElement('article')
  page.className = 'ev-slide-page vp-doc'
  page.dataset.slideIndex = String(slideIndex)

  const prefix = `ev-slide-${slideRun}-${slideIndex}-`
  nodes.forEach((node) => {
    const clone = node.cloneNode(true)
    rewriteIds(clone, prefix, idMap)
    page.appendChild(clone)
  })

  section.appendChild(page)
  return section
}

const rewriteIds = (root, prefix, idMap) => {
  const nodes = root.id ? [root, ...root.querySelectorAll('[id]')] : [...root.querySelectorAll('[id]')]

  nodes.forEach((node) => {
    const oldId = node.id
    const newId = `${prefix}${oldId}`
    idMap.set(oldId, newId)
    node.id = newId
  })
}

const rewriteInternalLinks = (root, idMap) => {
  root.querySelectorAll('a[href^="#"]').forEach((link) => {
    const oldTarget = link.getAttribute('href')?.slice(1)
    if (!oldTarget || !idMap.has(oldTarget)) return
    link.setAttribute('href', `#${idMap.get(oldTarget)}`)
  })
}

const groupPageNodes = (source) => {
  const cover = []
  const sections = []
  let currentH2 = null
  let currentH3 = null

  Array.from(source.children).forEach((node) => {
    if (isHeading(node, 2)) {
      currentH2 = {
        intro: [node],
        children: []
      }
      currentH3 = null
      sections.push(currentH2)
      return
    }

    if (isHeading(node, 3) && currentH2) {
      currentH3 = {
        nodes: [node]
      }
      currentH2.children.push(currentH3)
      return
    }

    if (!currentH2) {
      cover.push(node)
      return
    }

    if (currentH3) {
      currentH3.nodes.push(node)
      return
    }

    currentH2.intro.push(node)
  })

  return { cover, sections }
}

const buildSlides = (source, slidesRoot) => {
  const idMap = new Map()
  const { cover, sections } = groupPageNodes(source)
  let slideIndex = 0

  slidesRoot.innerHTML = ''
  slideRun += 1

  if (cover.length) {
    slideIndex += 1
    slidesRoot.appendChild(createSlideSection(cover, idMap, slideIndex))
  }

  if (!sections.length && !cover.length) {
    slideIndex += 1
    slidesRoot.appendChild(createSlideSection(Array.from(source.children), idMap, slideIndex))
  }

  sections.forEach((section) => {
    if (!section.children.length) {
      slideIndex += 1
      slidesRoot.appendChild(createSlideSection(section.intro, idMap, slideIndex))
      return
    }

    const horizontal = document.createElement('section')
    horizontal.className = 'ev-slide-stack'

    slideIndex += 1
    horizontal.appendChild(createSlideSection(section.intro, idMap, slideIndex))

    section.children.forEach((child) => {
      slideIndex += 1
      horizontal.appendChild(createSlideSection(child.nodes, idMap, slideIndex))
    })

    slidesRoot.appendChild(horizontal)
  })

  rewriteInternalLinks(slidesRoot, idMap)
  return slideIndex
}

const requestOverlayFullscreen = async () => {
  const overlay = overlayRef.value
  if (!overlay?.requestFullscreen || document.fullscreenElement) return

  try {
    await overlay.requestFullscreen()
    openedFullscreen = true
  } catch {
    openedFullscreen = false
  }
}

const getRevealSize = () => {
  if (window.innerWidth <= 768) {
    return {
      width: window.innerWidth,
      height: window.innerHeight,
      margin: 0.02
    }
  }

  return {
    width: 1280,
    height: 720,
    margin: 0.04
  }
}

const openSlides = async () => {
  const source = getDocContent()
  if (!source || isOpen.value) return

  isOpen.value = true
  previousBodyOverflow = document.body.style.overflow
  document.body.style.overflow = 'hidden'

  await nextTick()

  if (!slidesRef.value || !revealRef.value) {
    await closeSlides()
    return
  }

  slideCount.value = buildSlides(source, slidesRef.value)

  const { default: Reveal } = await import('reveal.js')
  const size = getRevealSize()
  deck = new Reveal(revealRef.value, {
    controls: true,
    progress: true,
    hash: false,
    history: false,
    center: false,
    width: size.width,
    height: size.height,
    margin: size.margin,
    minScale: 0.2,
    maxScale: 2,
    transition: 'slide'
  })

  await deck.initialize()
  void requestOverlayFullscreen().then(() => deck?.layout?.())
  overlayRef.value?.focus()
}

const closeSlides = async () => {
  if (!isOpen.value) return

  if (deck) {
    deck.destroy()
    deck = null
  }

  if (openedFullscreen && document.fullscreenElement) {
    try {
      await document.exitFullscreen()
    } catch {
      // Keep closing the overlay even if the browser rejects fullscreen exit.
    }
  }

  openedFullscreen = false
  isOpen.value = false
  slideCount.value = 0
  document.body.style.overflow = previousBodyOverflow

  await nextTick()
  if (slidesRef.value) slidesRef.value.innerHTML = ''
}

watch(
  () => route.path,
  async () => {
    await closeSlides()
    await refreshAvailability()
  }
)

watch(isDocPage, refreshAvailability)

onMounted(refreshAvailability)

onBeforeUnmount(() => {
  if (deck) deck.destroy()
  document.body.style.overflow = previousBodyOverflow
})
</script>

<template>
  <button
    v-if="hasDocContent"
    class="ev-slides-button"
    type="button"
    aria-label="打开幻灯片"
    title="幻灯片"
    @click="openSlides"
  >
    <el-icon :size="16">
      <Present />
    </el-icon>
    <span class="ev-slides-button-label">幻灯片</span>
  </button>

  <Teleport to="body">
    <div
      v-if="isOpen"
      ref="overlayRef"
      class="ev-slides-overlay"
      tabindex="-1"
      role="dialog"
      aria-modal="true"
      aria-label="页面幻灯片播放"
      @keydown.esc.stop.prevent="closeSlides"
    >
      <button
        class="ev-slides-close"
        type="button"
        aria-label="关闭幻灯片"
        title="关闭幻灯片"
        @click="closeSlides"
      >
        <el-icon :size="18">
          <Close />
        </el-icon>
      </button>

      <div class="ev-slides-shell">
        <div
          ref="revealRef"
          class="reveal ev-page-slides"
          :data-slide-count="slideCount"
        >
          <div ref="slidesRef" class="slides" />
        </div>
      </div>
    </div>
  </Teleport>
</template>

<style>
.ev-slides-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  height: 32px;
  min-width: 32px;
  margin-left: 12px;
  padding: 0 12px;
  border: 1px solid var(--vp-c-divider);
  border-radius: 999px;
  background: var(--vp-c-bg-soft);
  color: var(--vp-c-text-1);
  font-size: 13px;
  font-weight: 600;
  line-height: 1;
  cursor: pointer;
}

.ev-slides-button:hover {
  border-color: var(--vp-c-brand);
  color: var(--vp-c-brand);
}

.ev-slides-overlay {
  position: fixed;
  inset: 0;
  z-index: 9999;
  color: #1f2937;
  background: #f8fafc;
  outline: none;
}

.ev-slides-shell {
  width: 100%;
  height: 100%;
}

.ev-slides-close {
  position: fixed;
  top: 18px;
  right: 18px;
  z-index: 10010;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  border: 1px solid rgba(148, 163, 184, 0.42);
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.92);
  color: #334155;
  box-shadow: 0 12px 32px rgba(15, 23, 42, 0.14);
  cursor: pointer;
}

.ev-slides-close:hover {
  border-color: #2563eb;
  color: #2563eb;
}

.ev-page-slides {
  width: 100%;
  height: 100%;
  background: #f8fafc;
  font-family:
    Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI",
    sans-serif;
}

.ev-page-slides .slides {
  text-align: left;
}

.ev-page-slides section {
  height: 100%;
}

.ev-page-slides .ev-slide-section {
  padding: 0;
  background: transparent;
}

.ev-page-slides .ev-slide-page {
  box-sizing: border-box;
  width: 100%;
  height: 100%;
  max-height: 100%;
  padding: 48px 58px;
  overflow: auto;
  border: 1px solid rgba(148, 163, 184, 0.22);
  border-radius: 18px;
  background: #ffffff;
  box-shadow: 0 24px 80px rgba(15, 23, 42, 0.14);
  color: var(--vp-c-text-1);
  font-size: 22px;
  line-height: 1.62;
}

.ev-page-slides .ev-slide-page > :first-child {
  margin-top: 0;
}

.ev-page-slides .ev-slide-page > :last-child {
  margin-bottom: 0;
}

.ev-page-slides .ev-slide-page h1 {
  margin-bottom: 24px;
  font-size: 48px;
  line-height: 1.16;
}

.ev-page-slides .ev-slide-page h2 {
  margin-bottom: 22px;
  font-size: 38px;
  line-height: 1.2;
}

.ev-page-slides .ev-slide-page h3 {
  margin-bottom: 20px;
  font-size: 32px;
  line-height: 1.25;
}

.ev-page-slides .ev-slide-page h4 {
  font-size: 26px;
}

.ev-page-slides .ev-slide-page p,
.ev-page-slides .ev-slide-page li,
.ev-page-slides .ev-slide-page blockquote,
.ev-page-slides .ev-slide-page td,
.ev-page-slides .ev-slide-page th {
  font-size: inherit;
}

.ev-page-slides .ev-slide-page img,
.ev-page-slides .ev-slide-page video,
.ev-page-slides .ev-slide-page canvas,
.ev-page-slides .ev-slide-page svg {
  max-width: 100%;
  height: auto;
}

.ev-page-slides .ev-slide-page table {
  display: table;
  width: 100%;
}

.ev-page-slides .controls {
  color: #2563eb;
}

.ev-page-slides .progress {
  color: #2563eb;
}

@media (max-width: 768px) {
  .ev-slides-button {
    width: 32px;
    margin-left: 8px;
    padding: 0;
  }

  .ev-slides-button-label {
    display: none;
  }

  .ev-page-slides .ev-slide-page {
    padding: 34px 28px;
    border-radius: 14px;
    font-size: 18px;
  }

  .ev-page-slides .ev-slide-page h1 {
    font-size: 34px;
  }

  .ev-page-slides .ev-slide-page h2 {
    font-size: 28px;
  }

  .ev-page-slides .ev-slide-page h3 {
    font-size: 24px;
  }
}
</style>
