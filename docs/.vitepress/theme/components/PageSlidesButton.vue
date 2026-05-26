<script setup>
import 'reveal.js/reveal.css'

import { Close, Download, Present } from '@element-plus/icons-vue'
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { useData, useRoute, withBase } from 'vitepress'
import { downloadFile } from './CopyOrDownloadAsMarkdownButtons/utils'

const { frontmatter, page, theme } = useData()
const route = useRoute()

const overlayRef = ref(null)
const revealRef = ref(null)
const slidesRef = ref(null)
const hasDocContent = ref(false)
const isOpen = ref(false)
const isPreparingSlides = ref(false)
const isCourseMode = ref(false)
const isExporting = ref(false)
const slideSourceCount = ref(0)
const slideSourceError = ref('')
const slideCount = ref(0)

let deck = null
let previousBodyOverflow = ''
let openedFullscreen = false
let slideRun = 0
let openRequest = 0
let lastWheelNavigationAt = 0

const FIT_TOLERANCE = 2
const CONTENT_IMAGE_WAIT_TIMEOUT = 2500
const COURSE_SLIDE_REVEAL_JS = 'https://cdn.jsdelivr.net/npm/reveal.js@6.0.1/dist/reveal.esm.min.js'
const COURSE_SLIDE_REVEAL_CSS = 'https://cdn.jsdelivr.net/npm/reveal.js@6.0.1/dist/reveal.css'
const REVEAL_EXPORT_TITLE_SUFFIX = '课程PPT.html'
const MIN_RENDER_FIT_ZOOM = 0.62
const MIN_OVERSIZED_BLOCK_SCALE = 0.35
const MIN_HEADING_FIT_SCALE = 0.74
const HEADING_FIT_STEP = 0.04
const HEADING_SHORT_LINE_FONT_RATIO = 1.65
const WHEEL_NAVIGATION_COOLDOWN = 520

const VISUAL_TOPIC_PROFILES = [
  {
    key: 'debug',
    test: /错误|报错|bug|异常|调试|查错|日志|console|修复|排查|失败|怎么办/i,
    badge: 'Debug Flow',
    title: '错误排查路线',
    symbol: '!',
    notes: [
      ['收集现象', 'Error'],
      ['定位原因', 'Trace'],
      ['修复验证', 'Pass']
    ],
    chips: ['报错信息', '复现步骤', '日志线索', '验证修复'],
    footer: '把错误现象、上下文和验证结果整理成可执行的排查路径',
    colors: ['#fb7185', '#f97316', '#38bdf8']
  },
  {
    key: 'roadmap',
    test: /学习地图|路线|路径|阶段|成长|学完|下一步|课程|计划|目标/i,
    badge: 'Learning Map',
    title: '学习路线图',
    symbol: '→',
    notes: [
      ['当前起点', 'Start'],
      ['阶段任务', 'Milestone'],
      ['交付成果', 'Outcome']
    ],
    chips: ['学习路径', '阶段推进', '能力成长', '产品原型'],
    footer: '把目标、阶段和成果串成一条能持续推进的学习路线',
    colors: ['#60a5fa', '#2dd4bf', '#a78bfa']
  },
  {
    key: 'prompt',
    test: /提问|提示词|prompt|沟通|截图|复制|解释|上下文|需求|说话/i,
    badge: 'Prompt Lab',
    title: '高质量提问',
    symbol: '?',
    notes: [
      ['补齐上下文', 'Context'],
      ['约束输出', 'Format'],
      ['追问迭代', 'Iterate']
    ],
    chips: ['背景信息', '截图证据', '目标约束', '反馈循环'],
    footer: '把模糊问题变成 AI 能理解、能执行、能继续迭代的请求',
    colors: ['#a78bfa', '#38bdf8', '#f472b6']
  },
  {
    key: 'workspace',
    test: /ide|trae|cursor|qoder|codebuddy|工具|环境|安装|终端|编辑器|按钮|界面|本地/i,
    badge: 'IDE Setup',
    title: '开发工具工作台',
    symbol: '</>',
    notes: [
      ['准备环境', 'Setup'],
      ['打开项目', 'IDE'],
      ['运行检查', 'Run']
    ],
    chips: ['编辑器', '终端', '文件树', '预览窗口'],
    footer: '把工具、项目和运行反馈放到同一个可操作的开发工作台里',
    colors: ['#22d3ee', '#818cf8', '#34d399']
  },
  {
    key: 'prototype',
    test: /原型|产品|搭建|开发|应用|demo|网页|项目|实现|小游戏|页面/i,
    badge: 'Prototype',
    title: '从想法到原型',
    symbol: '▣',
    notes: [
      ['明确范围', 'Scope'],
      ['搭出界面', 'UI'],
      ['演示闭环', 'Demo']
    ],
    chips: ['需求拆解', '页面结构', '交互闭环', '可演示'],
    footer: '把自然语言里的想法变成能点击、能运行、能展示的产品雏形',
    colors: ['#2563eb', '#14b8a6', '#f59e0b']
  },
  {
    key: 'ai',
    test: /ai|模型|大语言|智能|agent|rag|生成|能力|多模态|llm/i,
    badge: 'AI System',
    title: 'AI 能力链路',
    symbol: 'AI',
    notes: [
      ['输入资料', 'Input'],
      ['模型处理', 'Model'],
      ['结果落地', 'Output']
    ],
    chips: ['上下文', '模型选择', '能力调用', '结果评估'],
    footer: '把 AI 能力拆成输入、处理和落地结果，方便接入真实产品',
    colors: ['#8b5cf6', '#06b6d4', '#22c55e']
  },
  {
    key: 'deploy',
    test: /部署|上线|发布|服务器|云|域名|docker|vercel|pages|github/i,
    badge: 'Release Path',
    title: '上线发布链路',
    symbol: '↑',
    notes: [
      ['构建产物', 'Build'],
      ['发布服务', 'Deploy'],
      ['线上验证', 'Live']
    ],
    chips: ['构建检查', '静态资源', '访问地址', '回归验证'],
    footer: '把本地项目变成稳定可访问、可验证、可持续更新的线上服务',
    colors: ['#38bdf8', '#22c55e', '#facc15']
  },
  {
    key: 'data',
    test: /数据|数据库|表格|接口|api|请求|响应|json|supabase/i,
    badge: 'Data Flow',
    title: '数据流转图',
    symbol: '{}',
    notes: [
      ['定义结构', 'Schema'],
      ['连接接口', 'API'],
      ['形成洞察', 'Insight']
    ],
    chips: ['字段结构', '请求响应', '状态变化', '数据看板'],
    footer: '把数据结构、接口连接和页面状态组织成清晰的数据流',
    colors: ['#10b981', '#3b82f6', '#f59e0b']
  },
  {
    key: 'design',
    test: /设计|视觉|ui|figma|素材|布局|组件|体验|样式/i,
    badge: 'Design Frame',
    title: '界面设计稿',
    symbol: '◇',
    notes: [
      ['定义布局', 'Layout'],
      ['组合组件', 'Parts'],
      ['打磨体验', 'UX']
    ],
    chips: ['视觉层级', '组件状态', '响应式', '体验细节'],
    footer: '把内容、布局和交互状态组织成更容易理解的界面表达',
    colors: ['#f472b6', '#818cf8', '#2dd4bf']
  }
]

const DEFAULT_VISUAL_TOPIC = {
  key: 'concept',
  badge: 'Concept Map',
  title: '核心概念图',
  symbol: '*',
  notes: [
    ['抓住主题', 'Topic'],
    ['拆成步骤', 'Steps'],
    ['形成输出', 'Result']
  ],
  chips: ['关键概念', '操作步骤', '示例场景', '学习产出'],
  footer: '把这一页的核心内容整理成一张便于理解和记忆的主题图',
  colors: ['#60a5fa', '#2dd4bf', '#f59e0b']
}

const ANNOTATED_MEDIA_SLIDES = [
  {
    image: 'index-2026-01-14-14-30-00.png',
    eyebrow: '操作要点',
    title: '打开项目文件夹',
    variant: 'wide',
    columns: 'minmax(250px, 0.68fr) minmax(460px, 1.32fr)',
    imageHeight: 'min(48vh, 500px)',
    points: [
      { label: '项目位置', text: '选中 easy-vibe-电商' },
      { label: '打开方式', text: '让 AI IDE 接管目录' },
      { label: '关键动作', text: '确认路径，点击 Open' },
      { label: '下一步', text: '粘贴提示词生成代码' }
    ]
  },
  {
    image: 'index-2026-01-14-15-57-14.png',
    eyebrow: '截图修正',
    title: '模板库功能还没闭环',
    variant: 'portrait',
    columns: 'minmax(280px, 0.72fr) minmax(360px, 1.28fr)',
    imageHeight: 'min(52vh, 520px)',
    imageFit: 'cover',
    imagePosition: 'center 46%',
    points: [
      { label: '核心问题', text: '模板收藏缺口' },
      { label: '截图证据', text: '结果卡片无入口' },
      { label: '追问方向', text: '收藏 / 复用 / 参数' },
      { label: '迭代原则', text: '截图反馈，继续校准' }
    ]
  }
]

const isDocPage = computed(() => {
  const layout = frontmatter.value.layout
  return layout !== 'home' && route.path !== '/welcome/' && !route.path.endsWith('/welcome/')
})

const normalizeRoutePath = (value) => {
  if (!value) return '/'

  const [pathOnly] = value.split(/[?#]/)
  const withoutHtml = pathOnly.replace(/\.html$/, '/')
  const withLeadingSlash = withoutHtml.startsWith('/') ? withoutHtml : `/${withoutHtml}`

  return withLeadingSlash.endsWith('/') ? withLeadingSlash : `${withLeadingSlash}/`
}

const getComparablePaths = (value) => {
  const path = normalizeRoutePath(value)
  const basePath = normalizeRoutePath(withBase(value))

  return [...new Set([path, basePath])]
}

const routeMatchesPath = (currentPath, targetPath, mode = 'startsWith') => {
  const candidates = getComparablePaths(targetPath)
  return candidates.some((candidate) =>
    mode === 'exact' ? currentPath === candidate : currentPath.startsWith(candidate)
  )
}

const getRouteTitle = () => {
  return (
    frontmatter.value.title ||
    page.value.title ||
    page.value.description ||
    route.path
      .split('/')
      .filter(Boolean)
      .at(-1)
  )
}

const flattenSidebarItems = (items, groupText = '') => {
  return (items || []).flatMap((item) => {
    const children = item.items ? flattenSidebarItems(item.items, item.text || groupText) : []
    const current = item.link
      ? [
          {
            groupText,
            text: item.text,
            link: item.link
          }
        ]
      : []

    return [...current, ...children]
  })
}

const getActiveNavText = (currentPath, navItems = []) => {
  const candidates = navItems
    .filter((item) => {
      const activeMatch = item.activeMatch && routeMatchesPath(currentPath, item.activeMatch)
      const linkMatch = item.link && routeMatchesPath(currentPath, item.link)
      return activeMatch || linkMatch
    })
    .sort((first, second) => {
      const firstPath = normalizeRoutePath(first.activeMatch || first.link || '')
      const secondPath = normalizeRoutePath(second.activeMatch || second.link || '')
      return secondPath.length - firstPath.length
    })

  return candidates[0]?.text
}

const getActiveSidebarContext = (currentPath, sidebar = {}) => {
  if (Array.isArray(sidebar)) return { matchedKey: '', items: sidebar }

  const matchedKey = Object.keys(sidebar)
    .filter((key) => routeMatchesPath(currentPath, key))
    .sort((first, second) => second.length - first.length)[0]

  return {
    matchedKey: matchedKey || '',
    items: matchedKey ? sidebar[matchedKey] : []
  }
}

const getActiveSidebarItems = (currentPath, sidebar = {}) => {
  return getActiveSidebarContext(currentPath, sidebar).items
}

const getCourseSidebarItems = (currentPath, sidebar = {}) => {
  const context = getActiveSidebarContext(currentPath, sidebar)
  if (!context.matchedKey || !Array.isArray(context.items) || !context.items.length) return []

  const normalizedPrefix = normalizeRoutePath(context.matchedKey)
  const seen = new Set()
  const collected = flattenSidebarItems(context.items, '')
    .map((item) => ({
      groupText: item.groupText,
      text: item.text,
      link: normalizeRoutePath(item.link || '')
    }))
    .filter((item) => {
      if (!item.link) return false
      if (!routeMatchesPath(item.link, normalizedPrefix)) return false
      if (seen.has(item.link)) return false
      seen.add(item.link)
      return true
    })

  return collected
}

const slideBreadcrumb = computed(() => {
  const currentPath = normalizeRoutePath(route.path)
  const themeConfig = theme.value || {}
  const activeNavText = getActiveNavText(currentPath, themeConfig.nav)
  const sidebarItems = getActiveSidebarItems(currentPath, themeConfig.sidebar)
  const currentItem = flattenSidebarItems(sidebarItems).find(
    (item) => routeMatchesPath(currentPath, item.link, 'exact')
  )

  return [activeNavText, currentItem?.groupText, currentItem?.text || getRouteTitle()]
    .filter(Boolean)
    .filter((value, index, values) => values.indexOf(value) === index)
})

const courseBreadcrumb = computed(() => {
  const currentPath = normalizeRoutePath(route.path)
  const themeConfig = theme.value || {}
  const activeNavText = getActiveNavText(currentPath, themeConfig.nav)

  return activeNavText ? [activeNavText] : []
})

const courseSidebarItems = computed(() => {
  const themeConfig = theme.value || {}
  return getCourseSidebarItems(normalizeRoutePath(route.path), themeConfig.sidebar)
})

const hasCourseSlides = computed(() => courseSidebarItems.value.length > 1)
const showSingleSlideButton = computed(() => hasDocContent.value && !hasCourseSlides.value)

const getDocContent = () => {
  const doc = document.querySelector('.VPDoc .vp-doc')
  if (!doc) return null

  const onlyChild = doc.children.length === 1 ? doc.firstElementChild : null
  if (onlyChild?.querySelector?.('h1, h2, h3')) return onlyChild

  return doc
}

const getCoursePageUrl = (routePath) =>
  `${window.location.origin}${withBase(normalizeRoutePath(routePath))}`

const isAbsoluteOrSpecialUrl = (value) =>
  /^(#|data:|mailto:|tel:|https?:|\/\/)/.test(value)

const normalizeAttributeUrl = (value, baseUrl) => {
  if (!value || isAbsoluteOrSpecialUrl(value)) return value

  try {
    return new URL(value, baseUrl).toString()
  } catch {
    return value
  }
}

const normalizeSrcset = (value, baseUrl) => {
  if (!value || typeof value !== 'string') return value

  return value
    .split(',')
    .map((piece) => piece.trim())
    .filter(Boolean)
    .map((piece) => {
      const [src, ...rest] = piece.split(/\s+/)
      const normalizedSrc = normalizeAttributeUrl(src, baseUrl)

      return [normalizedSrc, ...rest].join(' ')
    })
    .join(', ')
}

const normalizeFetchedSource = (source, sourceUrl) => {
  const selectors = [
    ['a', 'href'],
    ['img', 'src'],
    ['img', 'srcset'],
    ['video', 'src'],
    ['video', 'poster'],
    ['source', 'src'],
    ['iframe', 'src']
  ]

  selectors.forEach(([tagName, attr]) => {
    source.querySelectorAll(`${tagName}[${attr}]`).forEach((node) => {
      const value = node.getAttribute(attr)
      if (!value) return
      if (attr === 'srcset') {
        node.setAttribute(attr, normalizeSrcset(value, sourceUrl))
      } else {
        const normalizedValue = normalizeAttributeUrl(value, sourceUrl)
        node.setAttribute(attr, normalizedValue)
      }
    })
  })
}

const prepareSourceForSlides = (source, fallbackTitle) => {
  const sourceClone = source.cloneNode(true)

  if (!sourceClone.querySelector('h1, h2, h3')) {
    const heading = document.createElement('h1')
    heading.textContent = fallbackTitle || getRouteTitle()
    sourceClone.prepend(heading)
  }

  return sourceClone
}

const loadCoursePageSource = async (link, fallbackTitle) => {
  const url = getCoursePageUrl(link)
  const response = await fetch(url)
  if (!response.ok) {
    throw new Error(`fetch failed: ${url} (${response.status})`)
  }

  const html = await response.text()
  const dom = new DOMParser().parseFromString(html, 'text/html')
  const source = dom.querySelector('.VPDoc .vp-doc') || dom.querySelector('.vp-doc')
  if (!source) return null

  normalizeFetchedSource(source, url)
  return prepareSourceForSlides(source, fallbackTitle)
}

const buildCourseSlideSources = async () => {
  const descriptors = courseSidebarItems.value
  const normalizedCurrent = normalizeRoutePath(route.path)
  const results = []
  const failed = []
  slideSourceError.value = ''

  for (const item of descriptors) {
    try {
      const targetLink = normalizeRoutePath(item.link || '')
      if (!targetLink) continue

      if (targetLink === normalizedCurrent) {
        const source = getDocContent()
        if (!source) {
          failed.push(item.text || targetLink)
          continue
        }

        results.push({
          breadcrumbItems: [...courseBreadcrumb.value, item.groupText, item.text].filter(Boolean),
          source: prepareSourceForSlides(source, item.text),
          link: targetLink,
          title: item.text
        })
        continue
      }

      const source = await loadCoursePageSource(targetLink, item.text)
      if (!source) {
        failed.push(item.text || targetLink)
        continue
      }

      results.push({
        breadcrumbItems: [...courseBreadcrumb.value, item.groupText, item.text].filter(Boolean),
        source,
        link: targetLink,
        title: item.text
      })
    } catch {
      failed.push(item.text || item.link)
    }
  }

  slideSourceError.value = failed.length ? `部分课程页加载失败：${failed.join('，')}` : ''

  return results
}

const refreshAvailability = async () => {
  await nextTick()
  hasDocContent.value = Boolean(isDocPage.value && getDocContent()?.querySelector('h1, h2, h3'))
}

const waitForContentImages = async (source) => {
  const pendingImages = Array.from(source.querySelectorAll('img')).filter((image) => !image.complete)

  if (!pendingImages.length) return

  const imageSettled = Promise.all(
    pendingImages.map((image) => {
      if (image.complete) return Promise.resolve()

      return new Promise((resolve) => {
        image.addEventListener('load', resolve, { once: true })
        image.addEventListener('error', resolve, { once: true })
      })
    })
  )

  const timeout = new Promise((resolve) =>
    window.setTimeout(resolve, CONTENT_IMAGE_WAIT_TIMEOUT)
  )

  await Promise.race([imageSettled, timeout])
}

const isHeading = (node, level) => node.tagName?.toLowerCase() === `h${level}`

const isPrimarySlideHeading = (node) => ['h1', 'h2', 'h3'].includes(node.tagName?.toLowerCase())

const removeIds = (root) => {
  if (root.id) root.removeAttribute('id')
  root.querySelectorAll?.('[id]').forEach((node) => node.removeAttribute('id'))
}

const removeHeaderAnchors = (root) => {
  root
    .querySelectorAll?.('a.header-anchor, a[aria-label^="Permalink"]')
    .forEach((node) => node.remove())
}

const getCleanHeadingText = (heading) => {
  if (!heading) return ''

  const clone = heading.cloneNode(true)
  removeHeaderAnchors(clone)
  clone.querySelectorAll?.('.ev-slide-continuation-label').forEach((node) => node.remove())

  return (clone.textContent || '')
    .replace(/\u200b/g, '')
    .replace(/#/g, '')
    .replace(/\s+/g, ' ')
    .trim()
}

const getBreadcrumbItems = (sectionItems = []) =>
  [...slideBreadcrumb.value, ...sectionItems]
    .filter(Boolean)
    .filter((value, index, values) => values.indexOf(value) === index)

const createSlideBreadcrumb = (items, currentSlideTitle = '') => {
  const normalizedSlideTitle = currentSlideTitle.trim()
  const breadcrumbItems = getBreadcrumbItems(items).filter((item) => item !== normalizedSlideTitle)
  if (!breadcrumbItems.length) return null

  const breadcrumb = document.createElement('div')
  breadcrumb.className = 'ev-slide-breadcrumb'
  breadcrumb.setAttribute('aria-label', '课程位置')

  breadcrumbItems.forEach((item, index) => {
    const itemNode = document.createElement('span')
    itemNode.className = 'ev-slide-breadcrumb-item'
    itemNode.textContent = item
    breadcrumb.appendChild(itemNode)

    if (index < breadcrumbItems.length - 1) {
      const separator = document.createElement('span')
      separator.className = 'ev-slide-breadcrumb-separator'
      separator.setAttribute('aria-hidden', 'true')
      separator.textContent = '/'
      breadcrumb.appendChild(separator)
    }
  })

  return breadcrumb
}

const createContinuationHeading = (heading) => {
  if (!heading) return null

  const clone = heading.cloneNode(true)
  removeIds(clone)
  removeHeaderAnchors(clone)
  clone.classList.add('ev-slide-continuation-heading')

  const label = document.createElement('span')
  label.className = 'ev-slide-continuation-label'
  label.textContent = '续页'
  clone.appendChild(label)

  return clone
}

const syncClonedState = (source, clone) => {
  const sourceNodes = [source, ...(source.querySelectorAll?.('*') ?? [])]
  const clonedNodes = [clone, ...(clone.querySelectorAll?.('*') ?? [])]

  sourceNodes.forEach((sourceNode, index) => {
    const clonedNode = clonedNodes[index]
    if (!clonedNode) return

    const tagName = sourceNode.tagName?.toLowerCase()

    if (tagName === 'canvas') {
      clonedNode.width = sourceNode.width
      clonedNode.height = sourceNode.height

      try {
        clonedNode.getContext?.('2d')?.drawImage(sourceNode, 0, 0)
      } catch {
        // Some canvases are tainted or WebGL-backed; keep the structural clone in those cases.
      }
      return
    }

    if (tagName === 'input') {
      if (sourceNode.type === 'checkbox' || sourceNode.type === 'radio') {
        clonedNode.checked = sourceNode.checked
      } else {
        clonedNode.value = sourceNode.value
      }
      return
    }

    if (tagName === 'textarea' || tagName === 'select') {
      clonedNode.value = sourceNode.value
      return
    }

    if (tagName === 'details') {
      clonedNode.open = sourceNode.open
    }
  })
}

const cloneContentNode = (node, options = {}) => {
  const clone = node.cloneNode(true)
  syncClonedState(node, clone)
  if (options.stripIds) removeIds(clone)
  return clone
}

const createMeasuredClone = (node) => cloneContentNode(node, { stripIds: true })

const createMeasureContext = (size) => {
  const container = document.createElement('div')
  container.className = 'ev-page-slides ev-slide-measure'
  container.style.width = `${size.width}px`
  container.style.height = `${size.height}px`

  const page = document.createElement('article')
  page.className = 'ev-slide-page vp-doc'
  container.appendChild(page)
  document.body.appendChild(container)

  return {
    container,
    page,
    destroy: () => container.remove()
  }
}

const hasGeneratedVisual = (nodes) =>
  nodes.some((node) => node.classList?.contains('ev-slide-visual-card'))

const hasCardLikeContent = (nodes) =>
  nodes.some((node) =>
    Boolean(
      node.matches?.('.el-card, details, .custom-block') ||
        node.querySelector?.('.el-card, details, .custom-block')
    )
  )

const hasVisualMedia = (nodes) =>
  nodes.some((node) => {
    const tagName = node.tagName?.toLowerCase()
    if (['img', 'video', 'canvas', 'svg', 'table', 'pre'].includes(tagName)) return true
    return Boolean(node.querySelector?.('img, video, canvas, svg, table, pre'))
  })

const displayMediaSelector = 'img, video, canvas, svg, iframe'

const hasDisplayMedia = (nodes) =>
  nodes.some((node) => {
    const tagName = node.tagName?.toLowerCase()
    if (displayMediaSelector.split(', ').includes(tagName)) return true
    return Boolean(node.querySelector?.(displayMediaSelector))
  })

const isDecorativeSlideNode = (node) => node.tagName?.toLowerCase() === 'hr'

const hasMeaningfulSlideContent = (node) => {
  if (!node || isDecorativeSlideNode(node)) return false
  if (node.classList?.contains('ev-slide-visual-card')) return true

  const tagName = node.tagName?.toLowerCase()
  if (['img', 'video', 'canvas', 'svg', 'table', 'pre', 'figure', 'iframe'].includes(tagName)) {
    return true
  }

  if (node.querySelector?.('img, video, canvas, svg, table, pre, figure, iframe, .ev-slide-visual-card')) {
    return true
  }

  return Boolean((node.textContent || '').replace(/\u200b/g, '').trim())
}

const getSlideTextLength = (nodes) =>
  nodes.reduce((total, node) => total + (node.textContent || '').trim().length, 0)

const renderMeasurePage = (measureContext, nodes) => {
  measureContext.page.classList.toggle('ev-slide-page--with-visual', hasGeneratedVisual(nodes))
  measureContext.page.classList.toggle('ev-slide-page--with-media', hasDisplayMedia(nodes))
  measureContext.page.classList.toggle('ev-slide-page--annotated-media', hasAnnotatedMediaSlide(nodes))
  measureContext.page.replaceChildren(
    ...nodes.map((node) => {
      const annotatedMediaConfig = getAnnotatedMediaConfig(node)
      return annotatedMediaConfig ? createAnnotatedMediaNode(node, annotatedMediaConfig, { stripIds: true }) : createMeasuredClone(node)
    })
  )
}

const doesPageFit = (measureContext, nodes) => {
  renderMeasurePage(measureContext, nodes)

  return (
    measureContext.page.scrollHeight <= measureContext.page.clientHeight + FIT_TOLERANCE &&
    measureContext.page.scrollWidth <= measureContext.page.clientWidth + FIT_TOLERANCE
  )
}

const splitListNode = (node) => {
  const children = Array.from(node.children)
  if (children.length <= 1) return [node]

  return children.map((child) => {
    const list = node.cloneNode(false)
    list.appendChild(cloneContentNode(child))
    return list
  })
}

const splitTableNode = (node) => {
  const tagName = node.tagName?.toLowerCase()
  const table = tagName === 'table' ? node : node.querySelector?.(':scope table')
  if (!table) return [node]

  const hasOnlyTableContent =
    table === node ||
    Array.from(node.children).every((child) => child === table || child.contains(table))

  if (!hasOnlyTableContent) return [node]

  const rows = Array.from(table.querySelectorAll(':scope > tbody > tr'))
  const fallbackRows = rows.length ? rows : Array.from(table.querySelectorAll(':scope > tr'))
  if (fallbackRows.length <= 1) return [node]

  const caption = table.querySelector(':scope > caption')
  const colGroups = Array.from(table.querySelectorAll(':scope > colgroup'))
  const thead = table.querySelector(':scope > thead')

  const tablePieces = fallbackRows.map((row) => {
    const tablePiece = table.cloneNode(false)
    if (caption) tablePiece.appendChild(cloneContentNode(caption))
    colGroups.forEach((colGroup) => tablePiece.appendChild(cloneContentNode(colGroup)))
    if (thead) tablePiece.appendChild(cloneContentNode(thead))

    const tbody = document.createElement('tbody')
    tbody.appendChild(cloneContentNode(row))
    tablePiece.appendChild(tbody)

    return tablePiece
  })

  if (table === node) return tablePieces

  return tablePieces.map((tablePiece) => {
    const wrapper = node.cloneNode(false)
    wrapper.appendChild(tablePiece)

    return wrapper
  })
}

const splitCodeLikeNode = (node) => {
  const pre = node.tagName?.toLowerCase() === 'pre' ? node : node.querySelector?.('pre')
  const code = pre?.querySelector('code')
  const text = code?.innerText || pre?.innerText || ''
  const lines = text.split('\n')

  if (!pre || lines.length <= 14) return [node]

  const chunks = []
  for (let index = 0; index < lines.length; index += 14) {
    const wrapper = node.tagName?.toLowerCase() === 'pre' ? null : node.cloneNode(false)
    const preClone = pre.cloneNode(false)
    const codeClone = code ? code.cloneNode(false) : document.createElement('code')

    codeClone.textContent = lines.slice(index, index + 14).join('\n')
    preClone.appendChild(codeClone)

    if (wrapper) {
      wrapper.appendChild(preClone)
      chunks.push(wrapper)
    } else {
      chunks.push(preClone)
    }
  }

  return chunks
}

const isDisplayMediaNode = (node) => {
  if (node.nodeType !== Node.ELEMENT_NODE) return false

  const element = /** @type {Element} */ (node)
  return element.matches(displayMediaSelector) || Boolean(element.querySelector(displayMediaSelector))
}

const splitMixedDisplayMediaNode = (node) => {
  const tagName = node.tagName?.toLowerCase()
  if (tagName !== 'p' && tagName !== 'figure') return [node]
  if (!hasDisplayMedia([node])) return [node]

  const pieces = []
  let textPiece = node.cloneNode(false)

  const flushTextPiece = () => {
    if (hasMeaningfulSlideContent(textPiece)) pieces.push(textPiece)
    textPiece = node.cloneNode(false)
  }

  Array.from(node.childNodes).forEach((child) => {
    if (isDisplayMediaNode(child)) {
      flushTextPiece()

      const mediaPiece = node.cloneNode(false)
      mediaPiece.appendChild(cloneContentNode(child))
      pieces.push(mediaPiece)
      return
    }

    textPiece.appendChild(cloneContentNode(child))
  })

  flushTextPiece()

  const hasTextPiece = pieces.some((piece) => !hasDisplayMedia([piece]))
  const hasMediaPiece = pieces.some((piece) => hasDisplayMedia([piece]))
  return hasTextPiece && hasMediaPiece ? pieces : [node]
}

const getAnnotatedMediaConfig = (node) => {
  const image = node.matches?.('img') ? node : node.querySelector?.('img')
  const src = image?.getAttribute('src') || ''

  return ANNOTATED_MEDIA_SLIDES.find((config) => src.includes(config.image))
}

const hasAnnotatedMediaSlide = (nodes) => nodes.some((node) => getAnnotatedMediaConfig(node))

const createAnnotatedMediaNode = (sourceNode, config, options = {}) => {
  const sourceImage = sourceNode.matches?.('img') ? sourceNode : sourceNode.querySelector?.('img')
  if (!sourceImage) return null

  const wrapper = document.createElement('div')
  wrapper.className = `ev-slide-annotated-media ev-slide-annotated-media--${config.variant || 'default'}`
  if (config.columns) wrapper.style.setProperty('--ev-slide-annotated-columns', config.columns)
  if (config.imageHeight) wrapper.style.setProperty('--ev-slide-annotated-image-height', config.imageHeight)
  if (config.imageFit) wrapper.style.setProperty('--ev-slide-annotated-object-fit', config.imageFit)
  if (config.imagePosition) {
    wrapper.style.setProperty('--ev-slide-annotated-object-position', config.imagePosition)
  }

  const summary = document.createElement('aside')
  summary.className = 'ev-slide-annotated-media-summary'

  const eyebrow = document.createElement('span')
  eyebrow.className = 'ev-slide-annotated-media-eyebrow'
  eyebrow.textContent = config.eyebrow

  const title = document.createElement('strong')
  title.className = 'ev-slide-annotated-media-title'
  title.textContent = config.title

  const list = document.createElement('ul')
  list.className = 'ev-slide-annotated-media-list'
  config.points.forEach((point) => {
    const item = document.createElement('li')
    if (typeof point === 'string') {
      item.textContent = point
    } else {
      const label = document.createElement('strong')
      label.textContent = point.label
      item.append(label, document.createTextNode(`：${point.text}`))
    }
    list.appendChild(item)
  })

  summary.append(eyebrow, title, list)

  const figure = document.createElement('figure')
  figure.className = 'ev-slide-annotated-media-figure'

  const image = cloneContentNode(sourceImage, options)
  image.classList.remove('img-tall', 'img-very-tall', 'img-ultra-tall', 'img-limit-width', 'img-limit-height')
  image.classList.add('ev-slide-annotated-media-image')
  image.setAttribute('alt', image.getAttribute('alt') || config.title)
  figure.appendChild(image)

  wrapper.append(summary, figure)
  return wrapper
}

const getSplitPieces = (node) => {
  const tagName = node.tagName?.toLowerCase()

  if (tagName === 'ul' || tagName === 'ol') return splitListNode(node)
  if (tagName === 'table') return splitTableNode(node)

  const codePieces = splitCodeLikeNode(node)
  if (codePieces.length > 1) return codePieces

  return [node]
}

const createScaledBlock = (node, measureContext, prefixNodes) => {
  renderMeasurePage(measureContext, [...prefixNodes, node])

  const measuredNode = measureContext.page.lastElementChild
  if (!measuredNode) return node

  const nodeRect = measuredNode.getBoundingClientRect()
  const pageRect = measureContext.page.getBoundingClientRect()
  const availableHeight = Math.max(120, pageRect.bottom - nodeRect.top)
  const availableWidth = Math.max(120, pageRect.width)
  const scale = Math.max(
    MIN_OVERSIZED_BLOCK_SCALE,
    Math.min(1, (availableHeight - FIT_TOLERANCE) / nodeRect.height, availableWidth / nodeRect.width)
  )

  if (!Number.isFinite(scale) || scale >= 1) return node

  const wrapper = document.createElement('div')
  wrapper.className = 'ev-slide-scaled-block'
  wrapper.style.height = `${Math.ceil(nodeRect.height * scale)}px`

  const inner = document.createElement('div')
  inner.className = 'ev-slide-scaled-block-inner'
  inner.style.transform = `scale(${scale})`
  inner.style.width = `${Math.ceil(100 / scale)}%`

  const annotatedMediaConfig = getAnnotatedMediaConfig(node)
  inner.appendChild(
    annotatedMediaConfig ? createAnnotatedMediaNode(node, annotatedMediaConfig) : cloneContentNode(node)
  )

  wrapper.appendChild(inner)
  return wrapper
}

const shouldAddVisualCard = (nodes, role) => {
  if (!nodes.length || hasVisualMedia(nodes)) return false
  if (hasCardLikeContent(nodes)) return false
  if (role === 'cover') return getSlideTextLength(nodes) <= 520
  if (role === 'intro') return getSlideTextLength(nodes) <= 260
  if (role === 'body') return getSlideTextLength(nodes) <= 180
  return false
}

const cleanVisualText = (text) =>
  (text || '')
    .replace(/#/g, '')
    .replace(/\s+/g, ' ')
    .trim()

const getVisualSourceText = (nodes) => cleanVisualText(nodes.map((node) => node.textContent || '').join(' '))

const getVisualHeadingText = (nodes) => {
  const heading = nodes.find(isPrimarySlideHeading)
  const headingText = cleanVisualText(heading?.textContent)

  if (headingText) return headingText

  return cleanVisualText(nodes[0]?.textContent || '')
}

const truncateVisualText = (text, maxLength = 18) => {
  const cleanText = cleanVisualText(text)
  if (cleanText.length <= maxLength) return cleanText

  return `${cleanText.slice(0, maxLength)}...`
}

const createVisualNote = ([eyebrow, label], className) => {
  const note = document.createElement('div')
  note.className = `ev-slide-visual-note ${className}`

  const eyebrowNode = document.createElement('span')
  eyebrowNode.textContent = eyebrow

  const labelNode = document.createElement('b')
  labelNode.textContent = label

  note.append(eyebrowNode, labelNode)
  return note
}

const getVisualKeywords = (sourceText, profile) => {
  const matchedChips = profile.chips.filter((chip) => sourceText.includes(chip))
  const titleTerms =
    sourceText
      .match(/[A-Za-z][A-Za-z0-9+#.-]{1,}|[\u4e00-\u9fa5]{2,6}/g)
      ?.filter((term) => !['Permalink', 'to', '什么', '为什么', '怎么', '一个', '这个'].includes(term))
      ?.slice(0, 4) ?? []

  return [...new Set([...matchedChips, ...titleTerms, ...profile.chips])].slice(0, 4)
}

const getVisualTopicProfile = (nodes) => {
  const sourceText = getVisualSourceText(nodes)
  const headingText = getVisualHeadingText(nodes)
  const matchedProfile =
    VISUAL_TOPIC_PROFILES.find((profile) => profile.test.test(sourceText)) ?? DEFAULT_VISUAL_TOPIC
  const keywords = getVisualKeywords(sourceText, matchedProfile)

  return {
    ...matchedProfile,
    badge: matchedProfile.badge,
    heading: truncateVisualText(headingText || matchedProfile.title, 22),
    keywords
  }
}

const applyVisualTopicColors = (card, colors) => {
  const [primary, secondary, tertiary] = colors
  card.style.setProperty('--ev-slide-topic-a', primary)
  card.style.setProperty('--ev-slide-topic-b', secondary)
  card.style.setProperty('--ev-slide-topic-c', tertiary)
  card.style.setProperty('--ev-slide-topic-soft', `color-mix(in srgb, ${primary} 22%, transparent)`)
  card.style.setProperty('--ev-slide-topic-soft-2', `color-mix(in srgb, ${secondary} 16%, transparent)`)
  card.style.setProperty('--ev-slide-topic-border', `color-mix(in srgb, ${primary} 42%, transparent)`)
}

const createSlideVisualCard = (nodes, role) => {
  const profile = getVisualTopicProfile(nodes)
  const card = document.createElement('figure')
  card.className = `ev-slide-visual-card ev-slide-visual-card--${role} ev-slide-visual-card--${profile.key}`
  card.setAttribute('aria-label', `${profile.heading} 主题插图`)
  applyVisualTopicColors(card, profile.colors)

  const panel = document.createElement('div')
  panel.className = 'ev-slide-visual-panel'

  const header = document.createElement('div')
  header.className = 'ev-slide-visual-header'

  const badge = document.createElement('span')
  badge.className = 'ev-slide-visual-badge'
  badge.textContent = profile.badge

  const title = document.createElement('strong')
  title.textContent = profile.heading

  header.append(badge, title)

  const stage = document.createElement('div')
  stage.className = 'ev-slide-visual-stage'

  const promptCard = createVisualNote(profile.notes[0], 'ev-slide-visual-note--prompt')
  const aiCard = createVisualNote(profile.notes[1], 'ev-slide-visual-note--ai')
  const resultCard = createVisualNote(profile.notes[2], 'ev-slide-visual-note--result')

  const connector = document.createElement('div')
  connector.className = 'ev-slide-visual-connector'

  const screen = document.createElement('div')
  screen.className = 'ev-slide-visual-screen'

  const windowControls = document.createElement('div')
  windowControls.className = 'ev-slide-visual-window'
  windowControls.append(document.createElement('span'), document.createElement('span'), document.createElement('span'))

  const symbol = document.createElement('div')
  symbol.className = 'ev-slide-visual-symbol'
  symbol.textContent = profile.symbol

  const screenTitle = document.createElement('div')
  screenTitle.className = 'ev-slide-visual-screen-title'
  screenTitle.textContent = profile.title

  const rows = ['ev-slide-visual-code-row--wide', '', 'ev-slide-visual-code-row--short'].map(
    (modifier, index) => {
      const row = document.createElement('div')
      row.className = `ev-slide-visual-code-row ${modifier}`.trim()
      row.style.setProperty('--ev-row-index', String(index))
      return row
    }
  )

  const preview = document.createElement('div')
  preview.className = 'ev-slide-visual-preview'
  profile.keywords.forEach((keyword) => {
    const chip = document.createElement('span')
    chip.textContent = keyword
    preview.appendChild(chip)
  })

  screen.append(windowControls, symbol, screenTitle, ...rows, preview)

  stage.append(promptCard, connector, aiCard, resultCard, screen)

  const footer = document.createElement('figcaption')
  footer.className = 'ev-slide-visual-caption'
  footer.textContent = profile.footer

  panel.append(header, stage, footer)
  card.appendChild(panel)

  return card
}

const decorateSparseSlide = (nodes, role) => {
  if (!shouldAddVisualCard(nodes, role)) return nodes
  return [...nodes, createSlideVisualCard(nodes, role)]
}

const paginateLogicalSlide = (nodes, measureContext) => {
  const contentNodes = nodes.filter((node) => !isDecorativeSlideNode(node))
  if (!contentNodes.length) return []

  const primaryHeading = contentNodes.find(isPrimarySlideHeading)
  const pages = []
  let currentNodes = []

  const isBaseSlideNode = (node) =>
    node === primaryHeading || node.classList?.contains('ev-slide-continuation-heading')

  const currentHasBodyContent = () =>
    currentNodes.some((node) => !isBaseSlideNode(node) && hasMeaningfulSlideContent(node))

  const currentHasDisplayMedia = () =>
    currentNodes.some((node) => !isBaseSlideNode(node) && hasDisplayMedia([node]))

  const startContinuationPage = () => {
    const heading = createContinuationHeading(primaryHeading)
    currentNodes = heading ? [heading] : []
  }

  const commitCurrentPage = () => {
    if (currentHasBodyContent()) pages.push(currentNodes)
    startContinuationPage()
  }

  const addNode = (node) => {
    const mixedMediaPieces = splitMixedDisplayMediaNode(node)
    if (mixedMediaPieces.length > 1) {
      mixedMediaPieces.forEach(addNode)
      return
    }

    const nodeHasDisplayMedia = hasDisplayMedia([node])

    if (!nodeHasDisplayMedia && currentHasDisplayMedia()) {
      commitCurrentPage()
    }

    const candidateNodes = [...currentNodes, node]
    if (doesPageFit(measureContext, candidateNodes)) {
      currentNodes.push(node)
      return
    }

    if (currentHasBodyContent()) {
      commitCurrentPage()
      addNode(node)
      return
    }

    const pieces = getSplitPieces(node)
    if (pieces.length > 1) {
      pieces.forEach(addNode)
      return
    }

    const scaledBlock = createScaledBlock(node, measureContext, currentNodes)
    currentNodes.push(scaledBlock)
  }

  contentNodes.forEach(addNode)
  if (currentHasBodyContent()) pages.push(currentNodes)

  return pages
}

const createSlideSection = (
  nodes,
  idMap,
  slideIndex,
  breadcrumbItems = [],
  runId = slideRun
) => {
  const section = document.createElement('section')
  section.className = 'ev-slide-section'

  const page = document.createElement('article')
  page.className = 'ev-slide-page vp-doc'
  page.classList.toggle('ev-slide-page--with-visual', hasGeneratedVisual(nodes))
  page.classList.toggle('ev-slide-page--with-media', hasDisplayMedia(nodes))
  page.classList.toggle('ev-slide-page--annotated-media', hasAnnotatedMediaSlide(nodes))
  page.dataset.slideIndex = String(slideIndex)

  const prefix = `ev-slide-${runId}-${slideIndex}-`
  const currentSlideTitle = getCleanHeadingText(nodes.find(isPrimarySlideHeading))
  const breadcrumb = createSlideBreadcrumb(breadcrumbItems, currentSlideTitle)
  if (breadcrumb) page.appendChild(breadcrumb)

  nodes.forEach((node) => {
    const annotatedMediaConfig = getAnnotatedMediaConfig(node)
    const clone = annotatedMediaConfig
      ? createAnnotatedMediaNode(node, annotatedMediaConfig)
      : cloneContentNode(node)
    if (!clone) return
    rewriteIds(clone, prefix, idMap)
    page.appendChild(clone)
  })

  const pageNumber = document.createElement('span')
  pageNumber.className = 'ev-slide-page-number'
  pageNumber.setAttribute('aria-label', `第 ${slideIndex} 页`)
  pageNumber.textContent = String(slideIndex)
  page.appendChild(pageNumber)

  section.appendChild(page)
  return section
}

const applySlideNumbers = (slidesRoot, totalSlides) => {
  Array.from(slidesRoot.querySelectorAll('.ev-slide-page')).forEach((page, index) => {
    const slideNumber = Array.from(page.children).find((child) =>
      child.classList?.contains('ev-slide-page-number')
    )

    if (!slideNumber) return

    const current = index + 1
    slideNumber.textContent = `${current} / ${totalSlides}`
    slideNumber.setAttribute('aria-label', `第 ${current} 页，共 ${totalSlides} 页`)
  })
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

const getTextLineBoxes = (node) => {
  const range = document.createRange()
  range.selectNodeContents(node)

  const rects = Array.from(range.getClientRects()).filter((rect) => rect.width > 1 && rect.height > 1)
  range.detach?.()

  return rects
    .reduce((lines, rect) => {
      const line = lines.find((item) => Math.abs(item.top - rect.top) <= 2)

      if (line) {
        line.left = Math.min(line.left, rect.left)
        line.right = Math.max(line.right, rect.right)
        line.top = Math.min(line.top, rect.top)
        line.bottom = Math.max(line.bottom, rect.bottom)
        return lines
      }

      lines.push({
        top: rect.top,
        bottom: rect.bottom,
        left: rect.left,
        right: rect.right
      })

      return lines
    }, [])
    .sort((first, second) => first.top - second.top)
    .map((line) => ({
      ...line,
      width: line.right - line.left
    }))
}

const hasShortFinalHeadingLine = (heading) => {
  const lines = getTextLineBoxes(heading)
  if (lines.length < 2) return false

  const lastLine = lines.at(-1)
  const previousLine = lines.at(-2)
  const headingRect = heading.getBoundingClientRect()
  const fontSize = Number.parseFloat(window.getComputedStyle(heading).fontSize)
  const shortLineWidth = Math.min(fontSize * HEADING_SHORT_LINE_FONT_RATIO, headingRect.width * 0.18)

  return lastLine.width <= shortLineWidth && previousLine.width > headingRect.width * 0.42
}

const fitSlideHeadings = (pages) => {
  pages.forEach((page) => {
    page.querySelectorAll('h1, h2, h3').forEach((heading) => {
      heading.style.removeProperty('--ev-slide-heading-fit')
      heading.classList.remove('ev-slide-heading-fitted')

      let fitScale = 1
      while (fitScale > MIN_HEADING_FIT_SCALE && hasShortFinalHeadingLine(heading)) {
        fitScale = Math.max(MIN_HEADING_FIT_SCALE, fitScale - HEADING_FIT_STEP)
        heading.style.setProperty('--ev-slide-heading-fit', String(Number(fitScale.toFixed(2))))
        heading.classList.add('ev-slide-heading-fitted')
      }
    })
  })
}

const alignGeneratedVisualCards = (pages) => {
  pages.forEach((page) => {
    const visualCard = page.querySelector('.ev-slide-visual-card')
    if (!visualCard) return

    visualCard.style.removeProperty('--ev-slide-visual-offset')
    visualCard.classList.remove('ev-slide-visual-card--aligned')

    const heading = page.querySelector('h1, h2, h3')
    if (!heading) return

    const headingRect = heading.getBoundingClientRect()
    const cardRect = visualCard.getBoundingClientRect()
    const offset = Math.max(0, Math.round(headingRect.top - cardRect.top))

    if (offset <= 1) return

    visualCard.style.setProperty('--ev-slide-visual-offset', `${offset}px`)
    visualCard.classList.add('ev-slide-visual-card--aligned')
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
        title: getCleanHeadingText(node),
        intro: [node],
        children: []
      }
      currentH3 = null
      sections.push(currentH2)
      return
    }

    if (isHeading(node, 3) && currentH2) {
      currentH3 = {
        title: getCleanHeadingText(node),
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

const buildSlidesForSource = ({
  source,
  size,
  runId,
  startSlideIndex = 0,
  breadcrumbItems = [],
  applyNumbers = false
}) => {
  const idMap = new Map()
  const { cover, sections } = groupPageNodes(source)
  const measureContext = createMeasureContext(size)
  let slideIndex = startSlideIndex

  const verticalStack = document.createElement('section')
  verticalStack.className = 'ev-slide-stack ev-slide-vertical-stack'

  const appendPages = (container, pages, additionalBreadcrumbItems = []) => {
    pages.forEach((pageNodes) => {
      slideIndex += 1
      container.appendChild(
        createSlideSection(
          pageNodes,
          idMap,
          slideIndex,
          [...breadcrumbItems, ...additionalBreadcrumbItems],
          runId
        )
      )
    })
  }

  try {
    if (cover.length) {
      appendPages(
        verticalStack,
        paginateLogicalSlide(decorateSparseSlide(cover, 'cover'), measureContext)
      )
    }

    if (!sections.length && !cover.length) {
      appendPages(verticalStack, paginateLogicalSlide(Array.from(source.children), measureContext))
    }

    sections.forEach((section) => {
      const introPages = paginateLogicalSlide(decorateSparseSlide(section.intro, 'intro'), measureContext)
      const sectionBreadcrumb = [section.title]

      if (!section.children.length) {
        appendPages(verticalStack, introPages, sectionBreadcrumb)
        return
      }

      appendPages(verticalStack, introPages, sectionBreadcrumb)
      section.children.forEach((child) =>
        appendPages(
          verticalStack,
          paginateLogicalSlide(decorateSparseSlide(child.nodes, 'body'), measureContext),
          [section.title, child.title]
        )
      )
    })
  } finally {
    measureContext.destroy()
  }

  if (!slideIndex || !verticalStack.children.length) {
    return { count: 0, sectionRoot: null, idMap }
  }

  rewriteInternalLinks(verticalStack, idMap)
  if (applyNumbers) applySlideNumbers(verticalStack, slideIndex)

  return { count: slideIndex - startSlideIndex, sectionRoot: verticalStack, idMap }
}

const fitRenderedSlides = () => {
  if (!revealRef.value) return

  const pages = Array.from(revealRef.value.querySelectorAll('.ev-slide-page'))
  pages.forEach((page) => {
    page.style.removeProperty('zoom')
    page.classList.remove('ev-slide-render-fitted')
  })

  fitSlideHeadings(pages)
  alignGeneratedVisualCards(pages)

  pages.forEach((page) => {
    if (
      page.scrollHeight <= page.clientHeight + FIT_TOLERANCE &&
      page.scrollWidth <= page.clientWidth + FIT_TOLERANCE
    ) {
      return
    }

    const heightScale = page.clientHeight / page.scrollHeight
    const widthScale = page.clientWidth / page.scrollWidth
    const scale = Math.max(MIN_RENDER_FIT_ZOOM, Math.min(0.98, heightScale, widthScale) - 0.01)

    page.style.zoom = String(scale)
    page.classList.add('ev-slide-render-fitted')
  })
}

const navigateSlides = (direction) => {
  if (!deck || isPreparingSlides.value) return

  if (direction === 'next') {
    deck.down()
    return
  }

  deck.up()
}

const handleSlideKeydown = (event) => {
  if (event.key === 'Escape') {
    event.preventDefault()
    event.stopPropagation()
    void closeSlides()
    return
  }

  const nextKeys = ['ArrowDown', 'PageDown', ' ', 'Spacebar']
  const previousKeys = ['ArrowUp', 'PageUp', 'Backspace']

  if (nextKeys.includes(event.key)) {
    event.preventDefault()
    event.stopPropagation()
    navigateSlides('next')
    return
  }

  if (previousKeys.includes(event.key)) {
    event.preventDefault()
    event.stopPropagation()
    navigateSlides('previous')
  }
}

const handleSlideWheel = (event) => {
  if (!deck || isPreparingSlides.value || Math.abs(event.deltaY) < 12) return

  event.preventDefault()
  event.stopPropagation()

  const now = window.performance.now()
  if (now - lastWheelNavigationAt < WHEEL_NAVIGATION_COOLDOWN) return

  lastWheelNavigationAt = now
  navigateSlides(event.deltaY > 0 ? 'next' : 'previous')
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

const buildCourseSlides = async (sources, size) => {
  if (!slidesRef.value) return 0

  slidesRef.value.innerHTML = ''
  let total = 0

  for (const sourceItem of sources) {
    if (!sourceItem?.source) continue

    slideRun += 1
    const result = buildSlidesForSource({
      source: sourceItem.source,
      size,
      runId: slideRun,
      startSlideIndex: total,
      breadcrumbItems: sourceItem.breadcrumbItems || [],
      applyNumbers: false
    })

    if (!result.sectionRoot || !result.count) continue

    slidesRef.value.appendChild(result.sectionRoot)
    total += result.count
  }

  if (total) applySlideNumbers(slidesRef.value, total)
  return total
}

const getDeckExportHtml = () => {
  if (!slidesRef.value) return ''

  const cssText = [
    ...Array.from(document.querySelectorAll('style')).flatMap((style) => [style.textContent || '']),
    ...Array.from(document.styleSheets).flatMap((sheet) => {
      try {
        return Array.from(sheet.cssRules).map((rule) => rule.cssText)
      } catch {
        return []
      }
    })
  ].join('\n')

  const title = (slideBreadcrumb.value[0] || page.value.title || '课程').replace(/[\\/:*?"<>|]/g, '-')
  const exportTitle = `${title}-${REVEAL_EXPORT_TITLE_SUFFIX}`

  return `<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>${exportTitle}</title>
    <link rel="stylesheet" href="${COURSE_SLIDE_REVEAL_CSS}" />
    <style>${cssText}</style>
  </head>
  <body>
    <div class="reveal ev-page-slides">
      <div class="slides">${slidesRef.value.innerHTML}</div>
    </div>
    <script type="module">
      import Reveal from '${COURSE_SLIDE_REVEAL_JS}'
      const deck = new Reveal(document.querySelector('.reveal'), {
        controls: true,
        controlsTutorial: false,
        progress: true,
        hash: false,
        history: false,
        keyboard: true,
        center: false,
        width: 1280,
        height: 720,
        margin: 0.04,
        minScale: 0.2,
        maxScale: 2,
        transition: 'slide'
      })
      deck.initialize()
    ${'</' + 'script>'}
  </body>
</html>
`
}

const downloadCourseSlides = () => {
  if (!isOpen.value || !slidesRef.value || isExporting.value || !slideCount.value) return

  isExporting.value = true
  try {
    const title = (slideBreadcrumb.value[0] || page.value.title || '课程').replace(/[\\/:*?"<>|]/g, '-')
    const exportTitle = `${title}-${REVEAL_EXPORT_TITLE_SUFFIX}`
    const html = getDeckExportHtml()
    downloadFile(exportTitle, html, 'text/html')
  } finally {
    isExporting.value = false
  }
}

const openSlides = async (mode = 'single') => {
  if (isOpen.value) return

  const size = getRevealSize()
  const shouldRenderCourse = mode === 'course' && hasCourseSlides.value
  const requestId = openRequest + 1
  openRequest = requestId
  const sourceItems = shouldRenderCourse ? await buildCourseSlideSources() : []

  if (!shouldRenderCourse) {
    const source = getDocContent()
    if (!source) return

    sourceItems.push({
      source: prepareSourceForSlides(source, getRouteTitle()),
      breadcrumbItems: slideBreadcrumb.value,
      link: normalizeRoutePath(route.path)
    })
  }

  if (!sourceItems.length && shouldRenderCourse) {
    const source = getDocContent()
    if (source) {
      const fallbackMessage = slideSourceError.value
        ? `课程PPT加载失败：${slideSourceError.value}，当前页演示降级展示`
        : '课程PPT加载失败，当前页演示降级展示'

      sourceItems.push({
        source: prepareSourceForSlides(source, getRouteTitle()),
        breadcrumbItems: slideBreadcrumb.value,
        link: normalizeRoutePath(route.path)
      })
      slideSourceError.value = fallbackMessage
    } else {
      slideSourceError.value = slideSourceError.value || '课程PPT加载失败，当前页内容未就绪'
    }
  }

  if (!sourceItems.length) return

  isOpen.value = true
  isPreparingSlides.value = true
  slideSourceCount.value = sourceItems.length
  isCourseMode.value = shouldRenderCourse
  previousBodyOverflow = document.body.style.overflow
  document.body.style.overflow = 'hidden'

  await nextTick()
  if (!isOpen.value || requestId !== openRequest) return

  await Promise.all(
    sourceItems.map((entry) =>
      entry.link === normalizeRoutePath(route.path) ? waitForContentImages(entry.source) : Promise.resolve()
    )
  )
  await nextTick()
  if (!isOpen.value || requestId !== openRequest) return

  if (!slidesRef.value || !revealRef.value) {
    await closeSlides()
    return
  }

  if (shouldRenderCourse) {
    slideCount.value = await buildCourseSlides(sourceItems, size)
  } else {
    slideRun += 1
    const result = buildSlidesForSource({
      source: sourceItems[0].source,
      size,
      runId: slideRun,
      startSlideIndex: 0,
      breadcrumbItems: sourceItems[0].breadcrumbItems || [],
      applyNumbers: true
    })

    if (result.sectionRoot && slidesRef.value) {
      slidesRef.value.innerHTML = ''
      slidesRef.value.appendChild(result.sectionRoot)
      slideCount.value = result.count
    } else {
      await closeSlides()
      return
    }
  }

  if (!slideCount.value || !slidesRef.value.children.length) {
    await closeSlides()
    return
  }

  const { default: Reveal } = await import('reveal.js')
  deck = new Reveal(revealRef.value, {
    controls: true,
    controlsTutorial: false,
    progress: true,
    hash: false,
    history: false,
    keyboard: false,
    center: false,
    width: size.width,
    height: size.height,
    margin: size.margin,
    minScale: 0.2,
    maxScale: 2,
    transition: 'slide'
  })

  await deck.initialize()
  fitRenderedSlides()
  isPreparingSlides.value = false
  void requestOverlayFullscreen().then(() => {
    deck?.layout?.()
    fitRenderedSlides()
  })
  overlayRef.value?.focus()
}

const closeSlides = async () => {
  if (!isOpen.value) return

  openRequest += 1
  isPreparingSlides.value = false

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
  lastWheelNavigationAt = 0
  isOpen.value = false
  isPreparingSlides.value = false
  isCourseMode.value = false
  slideSourceCount.value = 0
  slideCount.value = 0
  isExporting.value = false
  slideSourceError.value = ''
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
  <div v-if="hasDocContent" class="ev-slides-buttons">
    <button
      v-if="showSingleSlideButton"
      class="ev-slides-button"
      type="button"
      aria-label="打开幻灯片"
      title="幻灯片"
      @click="() => openSlides('single')"
    >
      <el-icon :size="16">
        <Present />
      </el-icon>
      <span class="ev-slides-button-label">幻灯片</span>
    </button>

    <button
      v-if="hasCourseSlides"
      class="ev-slides-button"
      type="button"
      aria-label="打开课程PPT"
      title="课程PPT"
      @click="() => openSlides('course')"
    >
      <el-icon :size="16">
        <Present />
      </el-icon>
      <span class="ev-slides-button-label">课程PPT</span>
    </button>
  </div>

  <Teleport to="body">
    <div
      v-if="isOpen"
      ref="overlayRef"
      class="ev-slides-overlay"
      tabindex="-1"
      role="dialog"
      aria-modal="true"
      aria-label="页面幻灯片播放"
      @keydown="handleSlideKeydown"
      @wheel.capture="handleSlideWheel"
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

      <button
        v-if="!isPreparingSlides && slideCount && isOpen"
        class="ev-slides-download"
        type="button"
        aria-label="下载课程PPT HTML"
        title="下载课程PPT HTML"
        @click="downloadCourseSlides"
      >
        <el-icon :size="18">
          <Download />
        </el-icon>
        <span class="ev-slides-download-label">导出</span>
      </button>

      <div class="ev-slides-shell">
        <div
          v-if="isPreparingSlides"
          class="ev-slides-loading"
          role="status"
          aria-live="polite"
        >
          <span class="ev-slides-loading-dot" aria-hidden="true" />
          <span>
            正在生成幻灯片{{ isCourseMode ? `（课程，共 ${slideSourceCount} 页）` : '' }}
          </span>
        </div>

        <div v-if="slideSourceError" class="ev-slides-error">
          {{ slideSourceError }}
        </div>

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

.ev-slides-buttons {
  display: inline-flex;
  align-items: center;
}

.ev-slides-button:hover {
  border-color: var(--vp-c-brand);
  color: var(--vp-c-brand);
}

.ev-slides-download {
  position: fixed;
  top: 18px;
  right: 70px;
  z-index: 10010;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  height: 40px;
  min-width: 76px;
  padding: 0 14px;
  border: 1px solid var(--ev-slide-border-strong);
  border-radius: 999px;
  background: var(--ev-slide-surface-raised);
  color: var(--ev-slide-brand);
  font-size: 13px;
  font-weight: 700;
  line-height: 1;
  cursor: pointer;
}

.ev-slides-download:hover,
.ev-slides-download:focus-visible {
  border-color: var(--ev-slide-brand);
}

.ev-slides-download-label {
  white-space: nowrap;
}

.ev-slides-error {
  position: fixed;
  top: 72px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 10000;
  padding: 10px 16px;
  border: 1px solid var(--vp-c-danger-2, #ef4444);
  border-radius: 10px;
  background: color-mix(in srgb, var(--vp-c-danger-2, #ef4444) 15%, transparent);
  color: var(--vp-c-danger-1, #b91c1c);
}

.ev-slides-overlay,
.ev-page-slides {
  --ev-slide-overlay-bg: #f8fafc;
  --ev-slide-surface: #ffffff;
  --ev-slide-surface-soft: #f8fafc;
  --ev-slide-surface-raised: rgba(255, 255, 255, 0.92);
  --ev-slide-text: #1e293b;
  --ev-slide-heading: #0f172a;
  --ev-slide-muted: #475569;
  --ev-slide-subtle: #64748b;
  --ev-slide-border: rgba(148, 163, 184, 0.22);
  --ev-slide-border-strong: rgba(148, 163, 184, 0.42);
  --ev-slide-brand: #2563eb;
  --ev-slide-brand-soft: rgba(37, 99, 235, 0.08);
  --ev-slide-brand-border: rgba(37, 99, 235, 0.24);
  --ev-slide-shadow: 0 24px 80px rgba(15, 23, 42, 0.14);
  --ev-slide-control-shadow: 0 12px 32px rgba(15, 23, 42, 0.14);
  --ev-slide-table-border: rgba(148, 163, 184, 0.32);
  --ev-slide-table-header: rgba(241, 245, 249, 0.78);
  --ev-slide-code-bg: #f8fafc;
  --ev-slide-code-text: #0f172a;
  --ev-slide-topic-a: var(--ev-slide-brand);
  --ev-slide-topic-b: #14b8a6;
  --ev-slide-topic-c: #f59e0b;
  --ev-slide-topic-soft: rgba(37, 99, 235, 0.14);
  --ev-slide-topic-soft-2: rgba(20, 184, 166, 0.1);
  --ev-slide-topic-border: var(--ev-slide-brand-border);
  --ev-slide-visual-bg: #ffffff;
  --ev-slide-visual-gradient: linear-gradient(
    135deg,
    var(--ev-slide-topic-soft),
    var(--ev-slide-topic-soft-2)
  );
  --ev-slide-visual-note-bg: rgba(255, 255, 255, 0.92);
  --ev-slide-visual-screen-bg: #0f172a;
}

.dark .ev-slides-overlay,
.dark .ev-page-slides {
  --ev-slide-overlay-bg: #020617;
  --ev-slide-surface: #0f172a;
  --ev-slide-surface-soft: #111827;
  --ev-slide-surface-raised: rgba(15, 23, 42, 0.94);
  --ev-slide-text: #e5e7eb;
  --ev-slide-heading: #f8fafc;
  --ev-slide-muted: #cbd5e1;
  --ev-slide-subtle: #94a3b8;
  --ev-slide-border: rgba(148, 163, 184, 0.3);
  --ev-slide-border-strong: rgba(148, 163, 184, 0.44);
  --ev-slide-brand: #60a5fa;
  --ev-slide-brand-soft: rgba(96, 165, 250, 0.16);
  --ev-slide-brand-border: rgba(96, 165, 250, 0.4);
  --ev-slide-shadow: 0 26px 90px rgba(0, 0, 0, 0.48);
  --ev-slide-control-shadow: 0 16px 40px rgba(0, 0, 0, 0.34);
  --ev-slide-table-border: rgba(148, 163, 184, 0.28);
  --ev-slide-table-header: rgba(30, 41, 59, 0.82);
  --ev-slide-code-bg: #020617;
  --ev-slide-code-text: #e2e8f0;
  --ev-slide-topic-a: var(--ev-slide-brand);
  --ev-slide-topic-b: #2dd4bf;
  --ev-slide-topic-c: #fbbf24;
  --ev-slide-topic-soft: rgba(96, 165, 250, 0.2);
  --ev-slide-topic-soft-2: rgba(45, 212, 191, 0.12);
  --ev-slide-topic-border: var(--ev-slide-brand-border);
  --ev-slide-visual-bg: #111827;
  --ev-slide-visual-gradient: linear-gradient(
    135deg,
    var(--ev-slide-topic-soft),
    var(--ev-slide-topic-soft-2)
  );
  --ev-slide-visual-note-bg: rgba(15, 23, 42, 0.92);
  --ev-slide-visual-screen-bg: #020617;
}

.ev-slides-overlay {
  position: fixed;
  inset: 0;
  z-index: 9999;
  color: var(--ev-slide-text);
  background: var(--ev-slide-overlay-bg);
  outline: none;
}

.ev-slides-shell {
  position: relative;
  width: 100%;
  height: 100%;
}

.ev-slides-loading {
  position: fixed;
  inset: 0;
  z-index: 10000;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  background: var(--ev-slide-overlay-bg);
  color: var(--ev-slide-muted);
  font-size: 18px;
  font-weight: 700;
}

.ev-slides-loading-dot {
  width: 12px;
  height: 12px;
  border-radius: 999px;
  background: var(--ev-slide-brand);
  animation: ev-slides-pulse 1s ease-in-out infinite;
}

@keyframes ev-slides-pulse {
  0%,
  100% {
    opacity: 0.35;
    transform: scale(0.86);
  }

  50% {
    opacity: 1;
    transform: scale(1);
  }
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
  border: 1px solid var(--ev-slide-border-strong);
  border-radius: 999px;
  background: var(--ev-slide-surface-raised);
  color: var(--ev-slide-muted);
  box-shadow: var(--ev-slide-control-shadow);
  cursor: pointer;
}

.ev-slides-close:hover {
  border-color: var(--ev-slide-brand);
  color: var(--ev-slide-brand);
}

.ev-slide-breadcrumb {
  position: absolute;
  top: 24px;
  left: 58px;
  right: 58px;
  z-index: 2;
  display: flex;
  align-items: center;
  max-width: none;
  min-height: 32px;
  padding: 0 12px;
  overflow: hidden;
  border: 1px solid var(--ev-slide-border-strong);
  border-radius: 999px;
  background: var(--ev-slide-surface-soft);
  color: var(--ev-slide-muted);
  font-size: 13px;
  font-weight: 800;
  line-height: 1;
  pointer-events: none;
}

.ev-slide-breadcrumb-item {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.ev-slide-breadcrumb-item:last-child {
  color: var(--ev-slide-heading);
}

.ev-slide-breadcrumb-separator {
  flex: 0 0 auto;
  margin: 0 8px;
  color: var(--ev-slide-subtle);
}

.ev-page-slides {
  width: 100%;
  height: 100%;
  background: var(--ev-slide-overlay-bg);
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
  position: relative;
  box-sizing: border-box;
  width: 100%;
  height: 100%;
  max-height: 100%;
  padding: 82px 58px 68px;
  overflow: hidden;
  border: 1px solid var(--ev-slide-border);
  border-radius: 18px;
  background: var(--ev-slide-surface);
  box-shadow: var(--ev-slide-shadow);
  color: var(--ev-slide-text);
  font-size: 22px;
  line-height: 1.62;
}

.ev-page-slides .ev-slide-page--with-visual {
  display: grid;
  grid-template-columns: minmax(0, 1fr) minmax(320px, 420px);
  gap: 28px 38px;
  align-content: start;
}

.ev-page-slides .ev-slide-page--with-visual > :not(.ev-slide-visual-card) {
  grid-column: 1;
}

.ev-page-slides .ev-slide-page--with-visual > .ev-slide-visual-card {
  grid-column: 2;
  grid-row: 1 / span 8;
  margin-top: var(--ev-slide-visual-offset, 0);
}

.ev-page-slides .ev-slide-page > :first-child {
  margin-top: 0;
}

.ev-page-slides .ev-slide-page > :last-child {
  margin-bottom: 0;
}

.ev-page-slides .ev-slide-page-number {
  position: absolute;
  right: 30px;
  bottom: 22px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 68px;
  height: 28px;
  padding: 0 10px;
  border: 1px solid var(--ev-slide-border);
  border-radius: 999px;
  background: var(--ev-slide-surface-raised);
  color: var(--ev-slide-subtle);
  font-size: 12px;
  font-weight: 800;
  line-height: 1;
  letter-spacing: 0;
  pointer-events: none;
}

.ev-page-slides .ev-slide-page :where(h1, h2, h3, h4, strong) {
  color: var(--ev-slide-heading);
}

.ev-page-slides .ev-slide-page :where(h1, h2, h3) {
  max-width: 100%;
  overflow-wrap: normal;
  text-wrap: balance;
  word-break: normal;
}

.ev-page-slides .ev-slide-page :where(a:not(.header-anchor)) {
  color: var(--ev-slide-brand);
}

.ev-page-slides .ev-slide-page h1 {
  margin-bottom: 24px;
  font-size: calc(48px * var(--ev-slide-heading-fit, 1));
  line-height: 1.16;
}

.ev-page-slides .ev-slide-page h2 {
  margin-bottom: 22px;
  font-size: calc(38px * var(--ev-slide-heading-fit, 1));
  line-height: 1.2;
}

.ev-page-slides .ev-slide-page h3 {
  margin-bottom: 20px;
  font-size: calc(32px * var(--ev-slide-heading-fit, 1));
  line-height: 1.25;
}

.ev-page-slides .ev-slide-page h4 {
  font-size: 26px;
}

.ev-page-slides .ev-slide-continuation-heading {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 10px;
  color: var(--ev-slide-muted);
}

.ev-page-slides .ev-slide-continuation-label {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 3px 9px;
  border: 1px solid var(--ev-slide-brand-border);
  border-radius: 999px;
  background: var(--ev-slide-brand-soft);
  color: var(--ev-slide-brand);
  font-size: 0.42em;
  font-weight: 700;
  line-height: 1.25;
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

.ev-page-slides .ev-slide-page--with-media > :where(p, figure):has(> img:only-child),
.ev-page-slides .ev-slide-page--with-media > :where(p, figure):has(> video:only-child),
.ev-page-slides .ev-slide-page--with-media > :where(p, figure):has(> canvas:only-child),
.ev-page-slides .ev-slide-page--with-media > :where(p, figure):has(> svg:only-child) {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  margin: 14px 0 0;
}

.ev-page-slides .ev-slide-page--with-media :where(img, video, canvas, svg) {
  display: block;
  max-width: 100% !important;
  max-height: 520px !important;
  width: auto !important;
  height: auto !important;
  object-fit: contain;
  margin-inline: auto;
}

.ev-page-slides .ev-slide-page--with-media > :where(p, figure):has(> img:only-child) > :where(img, video, canvas, svg),
.ev-page-slides .ev-slide-page--with-media > :where(p, figure):has(> video:only-child) > :where(img, video, canvas, svg),
.ev-page-slides .ev-slide-page--with-media > :where(p, figure):has(> canvas:only-child) > :where(img, video, canvas, svg),
.ev-page-slides .ev-slide-page--with-media > :where(p, figure):has(> svg:only-child) > :where(img, video, canvas, svg) {
  width: min(100%, 1040px) !important;
  height: min(50vh, 520px) !important;
  max-height: min(50vh, 520px) !important;
}

.ev-page-slides .ev-slide-annotated-media {
  display: grid;
  grid-template-columns: var(
    --ev-slide-annotated-columns,
    minmax(280px, 0.82fr) minmax(360px, 1.18fr)
  );
  align-items: stretch;
  gap: 30px;
  width: 100%;
  margin-top: 12px;
}

.ev-page-slides .ev-slide-annotated-media-summary {
  display: flex;
  flex-direction: column;
  justify-content: center;
  min-height: 320px;
  padding: 26px 30px;
  border: 2px solid var(--ev-slide-brand-border);
  border-radius: 8px;
  background:
    linear-gradient(135deg, color-mix(in srgb, var(--ev-slide-brand) 13%, transparent), transparent 58%),
    color-mix(in srgb, var(--ev-slide-brand) 5%, var(--ev-slide-surface-raised));
  box-shadow:
    inset 0 1px 0 color-mix(in srgb, white 68%, transparent),
    0 16px 34px rgba(15, 23, 42, 0.1);
}

.ev-page-slides .ev-slide-annotated-media-eyebrow {
  width: fit-content;
  margin-bottom: 14px;
  padding: 5px 12px;
  border-radius: 999px;
  background: var(--ev-slide-brand-soft);
  color: var(--ev-slide-brand);
  font-size: 15px;
  font-weight: 800;
  letter-spacing: 0;
}

.ev-page-slides .ev-slide-annotated-media-title {
  margin-bottom: 18px;
  color: var(--ev-slide-heading);
  font-size: 30px;
  font-weight: 850;
  line-height: 1.22;
}

.ev-page-slides .ev-slide-annotated-media-list {
  display: grid;
  gap: 12px;
  margin: 0;
  padding-left: 1.18em;
  color: var(--ev-slide-text);
  font-size: 22px;
  line-height: 1.42;
}

.ev-page-slides .ev-slide-annotated-media-list li {
  padding-left: 5px;
}

.ev-page-slides .ev-slide-annotated-media-list strong {
  color: var(--ev-slide-heading);
  font-weight: 850;
}

.ev-page-slides .ev-slide-annotated-media-list li::marker {
  color: var(--ev-slide-brand);
}

.ev-page-slides .ev-slide-annotated-media-figure {
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 360px;
  margin: 0;
  padding: 14px;
  border: 1px solid var(--ev-slide-border);
  border-radius: 10px;
  background: color-mix(in srgb, var(--ev-slide-surface-raised) 82%, var(--ev-slide-brand-soft));
  box-shadow: 0 18px 38px rgba(15, 23, 42, 0.12);
}

.ev-page-slides .ev-slide-annotated-media-image {
  width: 100% !important;
  max-width: 100% !important;
  height: var(--ev-slide-annotated-image-height, min(50vh, 500px)) !important;
  max-height: var(--ev-slide-annotated-image-height, min(50vh, 500px)) !important;
  object-fit: var(--ev-slide-annotated-object-fit, contain);
  object-position: var(--ev-slide-annotated-object-position, center);
}

.ev-page-slides .ev-slide-page table {
  display: table;
  width: 100%;
  border-collapse: collapse;
  table-layout: fixed;
  font-size: 0.8em;
  line-height: 1.42;
}

.ev-page-slides .ev-slide-page th {
  background: var(--ev-slide-table-header);
}

.ev-page-slides .ev-slide-page th,
.ev-page-slides .ev-slide-page td {
  padding: 8px 10px;
  border: 1px solid var(--ev-slide-table-border);
  word-break: break-word;
}

.ev-page-slides .ev-slide-page :where(.vp-adaptive-theme, [class*="language-"]) {
  max-width: 100%;
  overflow: hidden;
}

.ev-page-slides .ev-slide-page pre {
  max-width: 100%;
  max-height: 100%;
  overflow: hidden;
  border: 1px solid var(--ev-slide-table-border);
  background: var(--ev-slide-code-bg);
  color: var(--ev-slide-code-text);
  font-size: 0.72em;
  line-height: 1.45;
  white-space: pre-wrap;
  word-break: break-word;
}

.ev-page-slides .ev-slide-page code {
  color: var(--ev-slide-code-text);
  font-size: 0.86em;
}

.ev-page-slides .ev-slide-page .custom-block {
  margin: 16px 0;
  padding: 14px 18px;
  border-color: var(--ev-slide-border);
  background: var(--ev-slide-surface-soft);
  color: var(--ev-slide-text);
  font-size: 0.92em;
  line-height: 1.5;
}

.ev-page-slides .ev-slide-page .el-card {
  border-color: var(--ev-slide-border);
  background: var(--ev-slide-surface-raised);
  color: var(--ev-slide-text);
}

.ev-page-slides .ev-slide-page .el-card__header {
  border-color: var(--ev-slide-border);
  background: var(--ev-slide-surface-soft);
  color: var(--ev-slide-heading);
}

.ev-page-slides .ev-slide-page .el-card :where(h1, h2, h3, h4, strong) {
  color: var(--ev-slide-heading);
}

.ev-page-slides .ev-slide-visual-card {
  align-self: start;
  box-sizing: border-box;
  width: 100%;
  min-height: 472px;
  margin: 0;
  padding: 0;
  border: 1px solid var(--ev-slide-topic-border);
  border-radius: 20px;
  background:
    radial-gradient(
      circle at 82% 16%,
      color-mix(in srgb, var(--ev-slide-topic-c) 16%, transparent),
      transparent 34%
    ),
    linear-gradient(135deg, var(--ev-slide-topic-soft), var(--ev-slide-topic-soft-2)),
    var(--ev-slide-visual-bg);
  box-shadow: var(--ev-slide-shadow);
  color: var(--ev-slide-text);
}

.ev-page-slides .ev-slide-visual-panel {
  display: flex;
  flex-direction: column;
  height: 100%;
  min-height: inherit;
  padding: 22px;
}

.ev-page-slides .ev-slide-visual-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  margin-bottom: 24px;
}

.ev-page-slides .ev-slide-visual-header strong {
  max-width: 60%;
  color: var(--ev-slide-heading);
  font-size: 18px;
  line-height: 1.25;
  text-align: right;
  overflow-wrap: anywhere;
}

.ev-page-slides .ev-slide-visual-badge {
  display: inline-flex;
  align-items: center;
  min-height: 28px;
  padding: 0 10px;
  border: 1px solid var(--ev-slide-topic-border);
  border-radius: 999px;
  background: color-mix(in srgb, var(--ev-slide-topic-a) 12%, var(--ev-slide-surface-raised));
  color: var(--ev-slide-topic-a);
  font-size: 12px;
  font-weight: 800;
  letter-spacing: 0;
  text-transform: uppercase;
}

.ev-page-slides .ev-slide-visual-stage {
  position: relative;
  flex: 1;
  min-height: 340px;
}

.ev-page-slides .ev-slide-visual-note {
  position: absolute;
  z-index: 2;
  display: flex;
  flex-direction: column;
  gap: 4px;
  width: 132px;
  padding: 13px 14px;
  border: 1px solid var(--ev-slide-topic-border);
  border-radius: 16px;
  background: var(--ev-slide-visual-note-bg);
  box-shadow: var(--ev-slide-control-shadow);
}

.ev-page-slides .ev-slide-visual-note span {
  color: var(--ev-slide-subtle);
  font-size: 12px;
  font-weight: 700;
  line-height: 1.2;
}

.ev-page-slides .ev-slide-visual-note b {
  color: var(--ev-slide-heading);
  font-size: 19px;
  line-height: 1.15;
}

.ev-page-slides .ev-slide-visual-note--prompt {
  top: 4px;
  left: 0;
}

.ev-page-slides .ev-slide-visual-note--ai {
  top: 74px;
  right: 4px;
  border-color: color-mix(in srgb, var(--ev-slide-topic-b) 44%, transparent);
}

.ev-page-slides .ev-slide-visual-note--result {
  right: 52px;
  bottom: 12px;
  border-color: color-mix(in srgb, var(--ev-slide-topic-c) 46%, transparent);
}

.ev-page-slides .ev-slide-visual-connector {
  position: absolute;
  top: 58px;
  left: 92px;
  width: 190px;
  height: 118px;
  border-top: 3px solid color-mix(in srgb, var(--ev-slide-topic-a) 46%, transparent);
  border-right: 3px solid color-mix(in srgb, var(--ev-slide-topic-b) 42%, transparent);
  border-radius: 0 26px 0 0;
}

.ev-page-slides .ev-slide-visual-screen {
  position: absolute;
  left: 8px;
  right: 18px;
  bottom: 48px;
  z-index: 1;
  min-height: 170px;
  padding: 18px;
  border: 1px solid color-mix(in srgb, var(--ev-slide-topic-a) 36%, var(--ev-slide-table-border));
  border-radius: 18px;
  background: var(--ev-slide-visual-screen-bg);
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.08);
}

.ev-page-slides .ev-slide-visual-window {
  display: flex;
  gap: 6px;
  margin-bottom: 18px;
}

.ev-page-slides .ev-slide-visual-window span {
  width: 9px;
  height: 9px;
  border-radius: 999px;
  background: var(--ev-slide-topic-a);
}

.ev-page-slides .ev-slide-visual-window span:nth-child(2) {
  background: var(--ev-slide-topic-b);
}

.ev-page-slides .ev-slide-visual-window span:nth-child(3) {
  background: var(--ev-slide-topic-c);
}

.ev-page-slides .ev-slide-visual-symbol {
  position: absolute;
  top: 14px;
  right: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 44px;
  height: 44px;
  border: 1px solid color-mix(in srgb, var(--ev-slide-topic-a) 34%, transparent);
  border-radius: 14px;
  background: color-mix(in srgb, var(--ev-slide-topic-a) 14%, transparent);
  color: var(--ev-slide-topic-a);
  font-size: 19px;
  font-weight: 900;
  line-height: 1;
}

.ev-page-slides .ev-slide-visual-screen-title {
  max-width: calc(100% - 64px);
  margin: 0 0 16px;
  color: #e2e8f0;
  font-size: 15px;
  font-weight: 800;
  line-height: 1.25;
  overflow-wrap: anywhere;
}

.ev-page-slides .ev-slide-visual-code-row {
  width: 58%;
  height: 10px;
  margin-bottom: 10px;
  border-radius: 999px;
  background: color-mix(in srgb, var(--ev-slide-topic-b) 42%, rgba(226, 232, 240, 0.76));
}

.ev-page-slides .ev-slide-visual-code-row--wide {
  width: 78%;
  background: color-mix(in srgb, var(--ev-slide-topic-a) 48%, rgba(226, 232, 240, 0.76));
}

.ev-page-slides .ev-slide-visual-code-row--short {
  width: 42%;
  background: color-mix(in srgb, var(--ev-slide-topic-c) 46%, rgba(226, 232, 240, 0.76));
}

.ev-page-slides .ev-slide-visual-preview {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 8px;
  margin-top: 20px;
}

.ev-page-slides .ev-slide-visual-preview span {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 46px;
  min-width: 0;
  padding: 0 8px;
  border: 1px solid color-mix(in srgb, var(--ev-slide-topic-a) 30%, transparent);
  border-radius: 12px;
  background: color-mix(in srgb, var(--ev-slide-topic-a) 18%, var(--ev-slide-visual-note-bg));
  color: var(--ev-slide-heading);
  font-size: 11px;
  font-weight: 800;
  line-height: 1.12;
  text-align: center;
  overflow: hidden;
  overflow-wrap: anywhere;
}

.ev-page-slides .ev-slide-visual-preview span:nth-child(2) {
  border-color: color-mix(in srgb, var(--ev-slide-topic-b) 30%, transparent);
  background: color-mix(in srgb, var(--ev-slide-topic-b) 18%, var(--ev-slide-visual-note-bg));
}

.ev-page-slides .ev-slide-visual-preview span:nth-child(3) {
  border-color: color-mix(in srgb, var(--ev-slide-topic-c) 32%, transparent);
  background: color-mix(in srgb, var(--ev-slide-topic-c) 18%, var(--ev-slide-visual-note-bg));
}

.ev-page-slides .ev-slide-visual-preview span:nth-child(4) {
  border-color: color-mix(in srgb, var(--ev-slide-topic-a) 24%, transparent);
  background: color-mix(in srgb, var(--ev-slide-topic-a) 12%, var(--ev-slide-visual-note-bg));
}

.ev-page-slides .ev-slide-visual-caption {
  margin-top: 20px;
  color: var(--ev-slide-muted);
  font-size: 14px;
  font-weight: 700;
  line-height: 1.45;
}

.ev-page-slides .controls {
  color: var(--ev-slide-brand);
}

.ev-page-slides .controls .navigate-left,
.ev-page-slides .controls .navigate-right {
  display: none;
}

.ev-page-slides .progress {
  color: var(--ev-slide-brand);
}

.ev-slide-scaled-block {
  width: 100%;
  overflow: hidden;
}

.ev-slide-scaled-block-inner {
  transform-origin: top left;
}

.ev-slide-measure {
  position: fixed;
  top: 0;
  left: -100000px;
  z-index: -1;
  overflow: hidden;
  visibility: hidden;
  pointer-events: none;
}

.ev-slide-measure .ev-slide-page {
  box-shadow: none;
}

@media (max-width: 768px) {
  .ev-slides-buttons {
    justify-content: center;
  }

  .ev-slides-button {
    width: 32px;
    margin-left: 8px;
    padding: 0;
  }

  .ev-slides-button-label {
    display: none;
  }

  .ev-slides-download {
    top: 58px;
    right: 8px;
    height: 32px;
    min-width: 32px;
    padding: 0;
  }

  .ev-slides-download-label {
    display: none;
  }

  .ev-page-slides .ev-slide-page {
    padding: 72px 28px 56px;
    border-radius: 14px;
    font-size: 18px;
  }

  .ev-page-slides .ev-slide-page-number {
    right: 18px;
    bottom: 16px;
    min-width: 58px;
    height: 24px;
    font-size: 11px;
  }

  .ev-slide-breadcrumb {
    top: 18px;
    left: 28px;
    right: 28px;
    max-width: none;
    min-height: 30px;
    padding: 0 10px;
    font-size: 11px;
  }

  .ev-slide-breadcrumb-separator {
    margin: 0 6px;
  }

  .ev-page-slides .ev-slide-page--with-visual {
    display: block;
  }

  .ev-page-slides .ev-slide-visual-card {
    min-height: 260px;
    margin-top: 22px;
  }

  .ev-page-slides .ev-slide-visual-panel {
    padding: 16px;
  }

  .ev-page-slides .ev-slide-visual-header {
    margin-bottom: 14px;
  }

  .ev-page-slides .ev-slide-visual-header strong,
  .ev-page-slides .ev-slide-visual-caption {
    display: none;
  }

  .ev-page-slides .ev-slide-visual-stage {
    min-height: 214px;
  }

  .ev-page-slides .ev-slide-visual-note {
    width: 112px;
    padding: 10px;
  }

  .ev-page-slides .ev-slide-visual-note b {
    font-size: 15px;
  }

  .ev-page-slides .ev-slide-visual-screen {
    right: 4px;
    bottom: 4px;
    min-height: 108px;
    padding: 12px;
  }

  .ev-page-slides .ev-slide-visual-symbol {
    top: 10px;
    right: 10px;
    width: 34px;
    height: 34px;
    border-radius: 11px;
    font-size: 14px;
  }

  .ev-page-slides .ev-slide-visual-screen-title {
    max-width: calc(100% - 48px);
    margin-bottom: 10px;
    font-size: 12px;
  }

  .ev-page-slides .ev-slide-visual-code-row {
    height: 7px;
    margin-bottom: 6px;
  }

  .ev-page-slides .ev-slide-visual-preview {
    gap: 6px;
    margin-top: 10px;
  }

  .ev-page-slides .ev-slide-visual-preview span {
    height: 28px;
    padding: 0 4px;
    font-size: 9px;
  }

  .ev-page-slides .ev-slide-page--with-media :where(img, video, canvas, svg) {
    max-height: 320px !important;
  }

  .ev-page-slides .ev-slide-page--with-media > :where(p, figure):has(> img:only-child) > :where(img, video, canvas, svg),
  .ev-page-slides .ev-slide-page--with-media > :where(p, figure):has(> video:only-child) > :where(img, video, canvas, svg),
  .ev-page-slides .ev-slide-page--with-media > :where(p, figure):has(> canvas:only-child) > :where(img, video, canvas, svg),
  .ev-page-slides .ev-slide-page--with-media > :where(p, figure):has(> svg:only-child) > :where(img, video, canvas, svg) {
    height: min(44vh, 320px) !important;
    max-height: min(44vh, 320px) !important;
  }

  .ev-page-slides .ev-slide-annotated-media {
    grid-template-columns: 1fr;
    gap: 16px;
    margin-top: 14px;
  }

  .ev-page-slides .ev-slide-annotated-media-summary {
    min-height: auto;
    padding: 18px 20px;
  }

  .ev-page-slides .ev-slide-annotated-media-title {
    font-size: 22px;
  }

  .ev-page-slides .ev-slide-annotated-media-list {
    gap: 9px;
    font-size: 16px;
  }

  .ev-page-slides .ev-slide-annotated-media-figure {
    min-height: auto;
    padding: 8px;
  }

  .ev-page-slides .ev-slide-annotated-media-image {
    height: min(32vh, 260px) !important;
    max-height: min(32vh, 260px) !important;
  }

  .ev-page-slides .ev-slide-page h1 {
    font-size: calc(34px * var(--ev-slide-heading-fit, 1));
  }

  .ev-page-slides .ev-slide-page h2 {
    font-size: calc(28px * var(--ev-slide-heading-fit, 1));
  }

  .ev-page-slides .ev-slide-page h3 {
    font-size: calc(24px * var(--ev-slide-heading-fit, 1));
  }
}
</style>
