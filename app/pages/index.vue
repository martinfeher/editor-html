<script setup lang="ts">
export interface TextBlock {
  kind: 'text'
  id: string
  content: string
  x: number
  y: number
  widthPct: number
  heightPct: number
  fontSize: number
  color: string
  fontWeight: '400' | '700'
  fontStyle: 'normal' | 'italic'
  textAlign: 'left' | 'center' | 'right'
  /** CSS `font-family` stack (one of `TEXT_FONT_OPTIONS` values). */
  fontFamily: string
}

export interface ImageBlock {
  kind: 'image'
  id: string
  src: string
  x: number
  y: number
  widthPct: number
  heightPct: number
  opacity: number
  borderRadius: number
  objectFit: 'cover' | 'contain' | 'fill'
  brightness: number
}

export type CardBlock = TextBlock | ImageBlock

interface EditorPage {
  id: string
  blocks: CardBlock[]
}

/** Preset stacks using common system fonts (no extra webfont loading). */
const TEXT_FONT_OPTIONS = [
  { label: 'System UI', value: 'system-ui, -apple-system, "Segoe UI", Roboto, sans-serif' },
  { label: 'Helvetica / Arial', value: 'Helvetica Neue, Helvetica, Arial, sans-serif' },
  { label: 'Arial', value: 'Arial, Helvetica, sans-serif' },
  { label: 'Verdana', value: 'Verdana, Geneva, sans-serif' },
  { label: 'Tahoma', value: 'Tahoma, Geneva, sans-serif' },
  { label: 'Trebuchet MS', value: '"Trebuchet MS", Helvetica, sans-serif' },
  { label: 'Georgia', value: 'Georgia, "Times New Roman", Times, serif' },
  { label: 'Times New Roman', value: '"Times New Roman", Times, serif' },
  { label: 'Palatino', value: 'Palatino, "Palatino Linotype", "Book Antiqua", Georgia, serif' },
  { label: 'Courier New', value: '"Courier New", Courier, monospace' },
] as const

const DEFAULT_TEXT_FONT_FAMILY = TEXT_FONT_OPTIONS[0].value

function normalizeTextFontFamily(raw: unknown): string {
  if (typeof raw !== 'string' || !raw.trim()) return DEFAULT_TEXT_FONT_FAMILY
  return TEXT_FONT_OPTIONS.some((o) => o.value === raw) ? raw : DEFAULT_TEXT_FONT_FAMILY
}

type ResizeCorner = 'nw' | 'ne' | 'sw' | 'se'

const MIN_BOX_PCT = 6

/** Base card size (px) used for zoom shell; card fills this box. */
const CARD_BASE_W = 420
const CARD_BASE_H = (CARD_BASE_W * 2) / 3.5

/** Mini previews in page strip (frame clips scaled card). */
const PAGE_THUMB_FRAME_W = 72
const PAGE_THUMB_SCALE = PAGE_THUMB_FRAME_W / CARD_BASE_W
const PAGE_THUMB_FRAME_H = CARD_BASE_H * PAGE_THUMB_SCALE

const editorZoom = ref(2)
const ZOOM_MIN = 0.25
const ZOOM_MAX = 4
const ZOOM_STEP = 0.1

function clampZoom(z: number) {
  return Math.min(ZOOM_MAX, Math.max(ZOOM_MIN, Math.round(z * 100) / 100))
}

function zoomIn() {
  editorZoom.value = clampZoom(editorZoom.value + ZOOM_STEP)
}

function zoomOut() {
  editorZoom.value = clampZoom(editorZoom.value - ZOOM_STEP)
}

function zoomReset() {
  editorZoom.value = 1
}

const canvasViewportRef = ref<HTMLElement | null>(null)

function onCanvasWheel(e: WheelEvent) {
  if (!e.ctrlKey && !e.metaKey) return
  e.preventDefault()
  const delta = e.deltaY > 0 ? -ZOOM_STEP : ZOOM_STEP
  editorZoom.value = clampZoom(editorZoom.value + delta)
}

onMounted(() => {
  const el = canvasViewportRef.value
  if (el) el.addEventListener('wheel', onCanvasWheel, { passive: false })
})

const zoomPercentUi = computed({
  get: () => Math.round(editorZoom.value * 100),
  set: (v: number) => {
    const n = Number(v)
    if (Number.isNaN(n)) return
    editorZoom.value = clampZoom(n / 100)
  },
})

let idCounter = 0
function newId() {
  return `block-${++idCounter}`
}

let pageIdCounter = 0
function newEditorPageId() {
  return `page-${++pageIdCounter}`
}

function revokeBlobUrlsInPages(pageList: readonly EditorPage[]) {
  for (const p of pageList) {
    revokeBlobUrlsInBlocks(p.blocks)
  }
}

function syncPageIdCounterFromPages(pageList: readonly EditorPage[]) {
  let max = 0
  for (const p of pageList) {
    const m = /^page-(\d+)$/.exec(p.id)
    if (m) max = Math.max(max, Number(m[1]))
  }
  pageIdCounter = max
}

/** --- Project save / load (JSON file, images as data URLs) --- */
const PROJECT_FORMAT = 'business-card-editor' as const
const PROJECT_VERSION = 2 as const
const PROJECT_VERSION_V1 = 1 as const

type SerializableText = Omit<TextBlock, 'kind'> & { kind: 'text' }
type SerializableImage = Omit<ImageBlock, 'kind'> & { kind: 'image'; src: string }
type SerializableBlock = SerializableText | SerializableImage

interface ProjectFileV1 {
  format: typeof PROJECT_FORMAT
  version: typeof PROJECT_VERSION_V1
  editorZoom: number
  blocks: SerializableBlock[]
}

interface SerializablePage {
  id?: string
  blocks: SerializableBlock[]
}

interface ProjectFileV2 {
  format: typeof PROJECT_FORMAT
  version: typeof PROJECT_VERSION
  editorZoom: number
  activePageIndex: number
  pages: SerializablePage[]
}

const projectInputRef = ref<HTMLInputElement | null>(null)
const projectBusy = ref(false)
const exportPdfBusy = ref(false)
const projectMessage = ref<string | null>(null)
let projectMsgTimer: ReturnType<typeof setTimeout> | null = null

function flashProjectMessage(msg: string) {
  projectMessage.value = msg
  if (projectMsgTimer) clearTimeout(projectMsgTimer)
  projectMsgTimer = setTimeout(() => {
    projectMessage.value = null
    projectMsgTimer = null
  }, 4500)
}

function revokeBlobUrlsInBlocks(list: readonly CardBlock[]) {
  for (const b of list) {
    if (b.kind === 'image' && b.src.startsWith('blob:')) URL.revokeObjectURL(b.src)
  }
}

async function imageSrcToDataUrl(src: string): Promise<string> {
  if (src.startsWith('data:')) return src
  const res = await fetch(src)
  if (!res.ok) throw new Error('fetch image')
  const blob = await res.blob()
  return new Promise((resolve, reject) => {
    const fr = new FileReader()
    fr.onload = () => resolve(fr.result as string)
    fr.onerror = () => reject(fr.error)
    fr.readAsDataURL(blob)
  })
}

async function blockToSerializable(b: CardBlock): Promise<SerializableBlock> {
  if (b.kind === 'text') {
    return {
      kind: 'text',
      id: b.id,
      content: b.content,
      x: b.x,
      y: b.y,
      widthPct: b.widthPct,
      heightPct: b.heightPct,
      fontSize: b.fontSize,
      color: b.color,
      fontWeight: b.fontWeight,
      fontStyle: b.fontStyle,
      textAlign: b.textAlign,
      fontFamily: b.fontFamily,
    }
  }
  const src = await imageSrcToDataUrl(b.src)
  return {
    kind: 'image',
    id: b.id,
    src,
    x: b.x,
    y: b.y,
    widthPct: b.widthPct,
    heightPct: b.heightPct,
    opacity: b.opacity,
    borderRadius: b.borderRadius,
    objectFit: b.objectFit,
    brightness: b.brightness,
  }
}

async function saveProject() {
  projectBusy.value = true
  try {
    const serialPages: SerializablePage[] = []
    for (const p of pages.value) {
      const blocksOut = await Promise.all(p.blocks.map(blockToSerializable))
      serialPages.push({ id: p.id, blocks: blocksOut })
    }
    const payload: ProjectFileV2 = {
      format: PROJECT_FORMAT,
      version: PROJECT_VERSION,
      editorZoom: editorZoom.value,
      activePageIndex: activePageIndex.value,
      pages: serialPages,
    }
    const json = JSON.stringify(payload, null, 2)
    const blob = new Blob([json], { type: 'application/json' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    const stamp = new Date().toISOString().slice(0, 19).replace(/[:T]/g, '-')
    a.href = url
    a.download = `business-card-project-${stamp}.json`
    a.rel = 'noopener'
    a.click()
    URL.revokeObjectURL(url)
    flashProjectMessage('Project saved — check your downloads.')
  } catch {
    flashProjectMessage('Could not save (images must load first).')
  } finally {
    projectBusy.value = false
  }
}

function openProjectPicker() {
  projectInputRef.value?.click()
}

function isRecord(v: unknown): v is Record<string, unknown> {
  return typeof v === 'object' && v !== null && !Array.isArray(v)
}

function isProjectFileV1(v: unknown): v is ProjectFileV1 {
  if (!isRecord(v)) return false
  if (v.format !== PROJECT_FORMAT) return false
  if (v.version !== PROJECT_VERSION_V1) return false
  if (typeof v.editorZoom !== 'number') return false
  if (!Array.isArray(v.blocks)) return false
  return true
}

function isProjectFileV2(v: unknown): v is ProjectFileV2 {
  if (!isRecord(v)) return false
  if (v.format !== PROJECT_FORMAT) return false
  if (v.version !== PROJECT_VERSION) return false
  if (typeof v.editorZoom !== 'number') return false
  if (typeof v.activePageIndex !== 'number') return false
  if (!Array.isArray(v.pages)) return false
  return true
}

function parseTextBlock(raw: Record<string, unknown>, id: string): TextBlock | null {
  if (typeof raw.content !== 'string') return null
  const x = Number(raw.x)
  const y = Number(raw.y)
  const widthPct = Number(raw.widthPct)
  const heightPct = Number(raw.heightPct)
  const fontSize = Number(raw.fontSize)
  if ([x, y, widthPct, heightPct, fontSize].some((n) => Number.isNaN(n))) return null
  const color = typeof raw.color === 'string' ? raw.color : '#1a1a1a'
  const fw = raw.fontWeight === '700' || raw.fontWeight === '400' ? raw.fontWeight : '400'
  const fs = raw.fontStyle === 'italic' || raw.fontStyle === 'normal' ? raw.fontStyle : 'normal'
  const ta =
    raw.textAlign === 'left' || raw.textAlign === 'center' || raw.textAlign === 'right'
      ? raw.textAlign
      : 'left'
  const fontFamily = normalizeTextFontFamily(raw.fontFamily)
  return {
    kind: 'text',
    id,
    content: raw.content,
    x,
    y,
    widthPct,
    heightPct,
    fontSize,
    color,
    fontWeight: fw,
    fontStyle: fs,
    textAlign: ta,
    fontFamily,
  }
}

function parseImageBlock(raw: Record<string, unknown>, id: string): ImageBlock | null {
  if (typeof raw.src !== 'string' || !raw.src.startsWith('data:')) return null
  const x = Number(raw.x)
  const y = Number(raw.y)
  const widthPct = Number(raw.widthPct)
  const heightPct = Number(raw.heightPct)
  const opacity = Number(raw.opacity)
  const borderRadius = Number(raw.borderRadius)
  const brightness = Number(raw.brightness)
  if ([x, y, widthPct, heightPct, opacity, borderRadius, brightness].some((n) => Number.isNaN(n))) return null
  const objectFit =
    raw.objectFit === 'cover' || raw.objectFit === 'contain' || raw.objectFit === 'fill'
      ? raw.objectFit
      : 'cover'
  return {
    kind: 'image',
    id,
    src: raw.src,
    x,
    y,
    widthPct,
    heightPct,
    opacity,
    borderRadius,
    objectFit,
    brightness,
  }
}

function parseBlockEntry(raw: unknown): CardBlock | null {
  if (!isRecord(raw)) return null
  const id = typeof raw.id === 'string' ? raw.id : newId()
  if (raw.kind === 'text') return parseTextBlock(raw, id)
  if (raw.kind === 'image') return parseImageBlock(raw, id)
  return null
}

function syncIdCounterFromPages(pageList: readonly EditorPage[]) {
  let max = 0
  for (const p of pageList) {
    for (const b of p.blocks) {
      const m = /^block-(\d+)$/.exec(b.id)
      if (m) max = Math.max(max, Number(m[1]))
    }
  }
  idCounter = max
}

function parseSerializedPage(raw: unknown): EditorPage | null {
  if (!isRecord(raw)) return null
  if (!Array.isArray(raw.blocks)) return null
  const blocksOut: CardBlock[] = []
  for (const entry of raw.blocks) {
    const b = parseBlockEntry(entry)
    if (b) blocksOut.push(b)
  }
  const id = typeof raw.id === 'string' && raw.id.trim() ? raw.id : newEditorPageId()
  return { id, blocks: blocksOut }
}

function clampActivePageIndex(i: number, len: number) {
  if (len <= 0) return 0
  return Math.min(len - 1, Math.max(0, Math.floor(i)))
}

async function onProjectFileChange(e: Event) {
  const input = e.target as HTMLInputElement
  const file = input.files?.[0]
  input.value = ''
  if (!file) return

  projectBusy.value = true
  try {
    const text = await file.text()
    const data: unknown = JSON.parse(text)

    if (isProjectFileV2(data)) {
      const nextPages: EditorPage[] = []
      for (const pg of data.pages) {
        const p = parseSerializedPage(pg)
        if (p) nextPages.push(p)
      }
      if (nextPages.length === 0) {
        flashProjectMessage('Project has no pages.')
        return
      }
      revokeBlobUrlsInPages(pages.value)
      pages.value = nextPages
      editorZoom.value = clampZoom(data.editorZoom)
      activePageIndex.value = clampActivePageIndex(data.activePageIndex, nextPages.length)
      syncIdCounterFromPages(nextPages)
      syncPageIdCounterFromPages(nextPages)
      const ab = activeBlocks()
      selectedId.value = ab[0]?.id ?? null
      dragging.value = null
      resizing.value = null
      let n = 0
      for (const p of nextPages) n += p.blocks.length
      flashProjectMessage(`Loaded ${nextPages.length} page(s), ${n} layer(s).`)
      return
    }

    if (isProjectFileV1(data)) {
      const next: CardBlock[] = []
      for (const entry of data.blocks) {
        const b = parseBlockEntry(entry)
        if (b) next.push(b)
      }
      revokeBlobUrlsInPages(pages.value)
      pages.value = [
        { id: newEditorPageId(), blocks: next },
        { id: newEditorPageId(), blocks: defaultBackBlocks() },
      ]
      editorZoom.value = clampZoom(data.editorZoom)
      activePageIndex.value = 0
      syncIdCounterFromPages(pages.value)
      syncPageIdCounterFromPages(pages.value)
      selectedId.value = next[0]?.id ?? null
      dragging.value = null
      resizing.value = null
      flashProjectMessage(`Loaded v1 project — ${next.length} layer(s) on page 1; page 2 is default.`)
      return
    }

    flashProjectMessage('Not a valid project file for this editor.')
  } catch {
    flashProjectMessage('Could not read that file.')
  } finally {
    projectBusy.value = false
  }
}

function createTextBlock(partial?: Partial<Omit<TextBlock, 'kind'>>): TextBlock {
  return {
    kind: 'text',
    id: newId(),
    content: partial?.content ?? 'Your text',
    x: partial?.x ?? 8,
    y: partial?.y ?? 12,
    widthPct: partial?.widthPct ?? 44,
    heightPct: partial?.heightPct ?? 18,
    fontSize: partial?.fontSize ?? 16,
    color: partial?.color ?? '#1a1a1a',
    fontWeight: partial?.fontWeight ?? '400',
    fontStyle: partial?.fontStyle ?? 'normal',
    textAlign: partial?.textAlign ?? 'left',
    fontFamily: partial?.fontFamily ?? DEFAULT_TEXT_FONT_FAMILY,
  }
}

function createImageBlock(src: string, partial?: Partial<Omit<ImageBlock, 'kind' | 'id' | 'src'>>): ImageBlock {
  return {
    kind: 'image',
    id: newId(),
    src,
    x: partial?.x ?? 55,
    y: partial?.y ?? 15,
    widthPct: partial?.widthPct ?? 38,
    heightPct: partial?.heightPct ?? 70,
    opacity: partial?.opacity ?? 1,
    borderRadius: partial?.borderRadius ?? 4,
    objectFit: partial?.objectFit ?? 'cover',
    brightness: partial?.brightness ?? 100,
  }
}

function defaultFrontBlocks(): CardBlock[] {
  return [
    createTextBlock({ content: 'Seb Carl', y: 10, fontSize: 14, fontWeight: '400' }),
    createTextBlock({ content: 'Product Designer', y: 28, fontSize: 13, color: '#555' }),
    createTextBlock({ content: 'jane@example.com', y: 72, fontSize: 11 }),
  ]
}

function defaultBackBlocks(): CardBlock[] {
  return [
    createTextBlock({
      content: 'Back side',
      y: 42,
      fontSize: 14,
      textAlign: 'center',
      x: 15,
      widthPct: 70,
      heightPct: 18,
    }),
  ]
}

const pages = ref<EditorPage[]>([
  { id: newEditorPageId(), blocks: defaultFrontBlocks() },
  { id: newEditorPageId(), blocks: defaultBackBlocks() },
])

const activePageIndex = ref(0)

function activeBlocks(): CardBlock[] {
  return pages.value[activePageIndex.value]!.blocks
}

const activeBlocksList = computed(() => pages.value[activePageIndex.value]?.blocks ?? [])

const selectedId = ref<string | null>(pages.value[0]?.blocks[0]?.id ?? null)

const selected = computed(() => activeBlocksList.value.find((b) => b.id === selectedId.value) ?? null)
const selectedText = computed(() => (selected.value?.kind === 'text' ? selected.value : null))
const selectedImage = computed(() => (selected.value?.kind === 'image' ? selected.value : null))

/** UI list order: first row = front on canvas (on top); last row = back (behind). */
const layersFrontFirst = computed(() => {
  const arr = activeBlocksList.value
  const out: { block: CardBlock; actualIndex: number }[] = []
  for (let i = arr.length - 1; i >= 0; i--) {
    out.push({ block: arr[i]!, actualIndex: i })
  }
  return out
})

function setActivePage(index: number) {
  if (index < 0 || index >= pages.value.length) return
  activePageIndex.value = index
  dragging.value = null
  resizing.value = null
  selectedId.value = pages.value[index]!.blocks[0]?.id ?? null
}

const imageInputRef = ref<HTMLInputElement | null>(null)
let imageInputMode: 'add' | 'replace' = 'add'

function addTextBlock() {
  const list = activeBlocks()
  const b = createTextBlock({
    y: Math.min(85, 12 + list.filter((x) => x.kind === 'text').length * 14),
  })
  list.push(b)
  selectedId.value = b.id
}

function openImagePicker(mode: 'add' | 'replace') {
  imageInputMode = mode
  imageInputRef.value?.click()
}

function onImageFileChange(e: Event) {
  const input = e.target as HTMLInputElement
  const file = input.files?.[0]
  input.value = ''
  if (!file || !file.type.startsWith('image/')) return

  const url = URL.createObjectURL(file)

  if (imageInputMode === 'replace' && selectedImage.value) {
    const prev = selectedImage.value.src
    if (prev.startsWith('blob:')) URL.revokeObjectURL(prev)
    selectedImage.value.src = url
    return
  }

  const list = activeBlocks()
  const b = createImageBlock(url)
  list.push(b)
  selectedId.value = b.id
}

function removeSelected() {
  if (!selectedId.value) return
  const list = activeBlocks()
  const idx = list.findIndex((b) => b.id === selectedId.value)
  if (idx === -1) return
  const removed = list[idx]
  if (removed.kind === 'image' && removed.src.startsWith('blob:')) {
    URL.revokeObjectURL(removed.src)
  }
  list.splice(idx, 1)
  selectedId.value = list[Math.max(0, idx - 1)]?.id ?? null
}

function moveLayer(delta: number) {
  if (!selectedId.value) return
  const list = activeBlocks()
  const i = list.findIndex((b) => b.id === selectedId.value)
  if (i === -1) return
  moveLayerAt(i, delta)
}

/** Swap layer at `index` with neighbor `index + delta` (delta ±1). */
function moveLayerAt(index: number, delta: number) {
  const list = activeBlocks()
  const j = index + delta
  if (j < 0 || j >= list.length) return
  const tmp = list[index]!
  list[index] = list[j]!
  list[j] = tmp
}

onBeforeUnmount(() => {
  if (projectMsgTimer) {
    clearTimeout(projectMsgTimer)
    projectMsgTimer = null
  }
  canvasViewportRef.value?.removeEventListener('wheel', onCanvasWheel)
  revokeBlobUrlsInPages(pages.value)
})

function layerLabel(b: CardBlock): string {
  if (b.kind === 'text') return b.content.trim() ? b.content.slice(0, 28) : '(empty text)'
  return 'Image'
}

const cardRef = ref<HTMLElement | null>(null)

/** US business card print size: 3.5 × 2 in */
const PDF_CARD_W_MM = 88.9
const PDF_CARD_H_MM = 50.8

async function exportCardToPdf() {
  if (!import.meta.client) return
  const el = cardRef.value
  if (!el) {
    flashProjectMessage('Card is not ready yet.')
    return
  }
  exportPdfBusy.value = true
  try {
    const [{ default: html2canvas }, { default: jsPDF }] = await Promise.all([
      import('html2canvas'),
      import('jspdf'),
    ])
    const canvas = await html2canvas(el, {
      scale: 3,
      useCORS: true,
      allowTaint: true,
      backgroundColor: '#ffffff',
      logging: false,
      ignoreElements: (node) =>
        node instanceof HTMLElement && node.classList.contains('resize-handle'),
      onclone: (_doc, cloned) => {
        cloned.querySelectorAll('.resize-handle').forEach((n) => n.remove())
        cloned.querySelectorAll('.card-item').forEach((n) => {
          n.classList.remove('selected')
        })
        cloned.style.boxShadow = 'none'
        cloned.style.border = '1px solid #e5e7eb'
      },
    })
    const imgData = canvas.toDataURL('image/png')
    const pdf = new jsPDF({
      unit: 'mm',
      format: [PDF_CARD_W_MM, PDF_CARD_H_MM],
      orientation: 'landscape',
      compress: true,
    })
    pdf.addImage(imgData, 'PNG', 0, 0, PDF_CARD_W_MM, PDF_CARD_H_MM, undefined, 'SLOW')
    const stamp = new Date().toISOString().slice(0, 19).replace(/[:T]/g, '-')
    pdf.save(`business-card-${stamp}.pdf`)
    flashProjectMessage('PDF exported — check your downloads.')
  } catch {
    flashProjectMessage('Could not create PDF. Try again or reload images.')
  } finally {
    exportPdfBusy.value = false
  }
}

const dragging = ref<{ id: string; startX: number; startY: number; origX: number; origY: number } | null>(null)
const resizing = ref<{
  id: string
  corner: ResizeCorner
  startX: number
  startY: number
  origX: number
  origY: number
  origW: number
  origH: number
} | null>(null)

function onBlockPointerDown(e: PointerEvent, block: CardBlock) {
  if (e.button !== 0) return
  selectedId.value = block.id
  const card = cardRef.value
  if (!card) return
  dragging.value = {
    id: block.id,
    startX: e.clientX,
    startY: e.clientY,
    origX: block.x,
    origY: block.y,
  }
  ;(e.currentTarget as HTMLElement).setPointerCapture(e.pointerId)
  e.preventDefault()
}

function onBlockPointerMove(e: PointerEvent, block: CardBlock) {
  const d = dragging.value
  const card = cardRef.value
  if (!d || d.id !== block.id || !card) return
  const rect = card.getBoundingClientRect()
  const dx = ((e.clientX - d.startX) / rect.width) * 100
  const dy = ((e.clientY - d.startY) / rect.height) * 100
  const maxX = Math.max(0, 100 - block.widthPct)
  const maxY = Math.max(0, 100 - block.heightPct)
  block.x = Math.round(Math.min(maxX, Math.max(0, d.origX + dx)) * 10) / 10
  block.y = Math.round(Math.min(maxY, Math.max(0, d.origY + dy)) * 10) / 10
}

function onBlockPointerUp(e: PointerEvent, block: CardBlock) {
  if (dragging.value?.id !== block.id) return
  dragging.value = null
  ;(e.currentTarget as HTMLElement).releasePointerCapture(e.pointerId)
}

function onResizePointerDown(e: PointerEvent, block: CardBlock, corner: ResizeCorner) {
  if (e.button !== 0) return
  e.stopPropagation()
  selectedId.value = block.id
  const card = cardRef.value
  if (!card) return
  resizing.value = {
    id: block.id,
    corner,
    startX: e.clientX,
    startY: e.clientY,
    origX: block.x,
    origY: block.y,
    origW: block.widthPct,
    origH: block.heightPct,
  }
  ;(e.currentTarget as HTMLElement).setPointerCapture(e.pointerId)
  e.preventDefault()
}

function applyResize(block: CardBlock, dx: number, dy: number) {
  const r = resizing.value
  if (!r || r.id !== block.id) return

  const ox = r.origX
  const oy = r.origY
  const ow = r.origW
  const oh = r.origH
  const right = ox + ow
  const bottom = oy + oh

  let x = ox
  let y = oy
  let w = ow
  let h = oh

  switch (r.corner) {
    case 'se': {
      w = Math.max(MIN_BOX_PCT, ow + dx)
      h = Math.max(MIN_BOX_PCT, oh + dy)
      w = Math.min(w, 100 - ox)
      h = Math.min(h, 100 - oy)
      x = ox
      y = oy
      break
    }
    case 'nw': {
      w = Math.max(MIN_BOX_PCT, ow - dx)
      h = Math.max(MIN_BOX_PCT, oh - dy)
      w = Math.min(w, right)
      h = Math.min(h, bottom)
      x = right - w
      y = bottom - h
      if (x < 0) {
        w = Math.max(MIN_BOX_PCT, right)
        x = 0
        w = Math.min(w, 100)
      }
      if (y < 0) {
        h = Math.max(MIN_BOX_PCT, bottom)
        y = 0
        h = Math.min(h, 100)
      }
      w = Math.min(w, 100 - x)
      h = Math.min(h, 100 - y)
      break
    }
    case 'ne': {
      w = Math.max(MIN_BOX_PCT, ow + dx)
      w = Math.min(w, 100 - ox)
      h = Math.max(MIN_BOX_PCT, oh - dy)
      h = Math.min(h, bottom)
      x = ox
      y = bottom - h
      break
    }
    case 'sw': {
      w = Math.max(MIN_BOX_PCT, ow - dx)
      h = Math.max(MIN_BOX_PCT, oh + dy)
      h = Math.min(h, 100 - oy)
      y = oy
      x = right - w
      if (x < 0) {
        w = Math.max(MIN_BOX_PCT, right)
        x = 0
        w = Math.min(w, 100)
      }
      w = Math.min(w, 100 - x)
      break
    }
  }

  w = Math.max(MIN_BOX_PCT, Math.min(w, 100 - x))
  h = Math.max(MIN_BOX_PCT, Math.min(h, 100 - y))

  block.x = Math.round(x * 10) / 10
  block.y = Math.round(y * 10) / 10
  block.widthPct = Math.round(w * 10) / 10
  block.heightPct = Math.round(h * 10) / 10
}

function onResizePointerMove(e: PointerEvent, block: CardBlock) {
  const r = resizing.value
  const card = cardRef.value
  if (!r || r.id !== block.id || !card) return
  const rect = card.getBoundingClientRect()
  const dx = ((e.clientX - r.startX) / rect.width) * 100
  const dy = ((e.clientY - r.startY) / rect.height) * 100
  applyResize(block, dx, dy)
}

function onResizePointerUp(e: PointerEvent, block: CardBlock) {
  if (resizing.value?.id !== block.id) return
  resizing.value = null
  ;(e.currentTarget as HTMLElement).releasePointerCapture(e.pointerId)
}

function onResizePointerCancel(e: PointerEvent, block: CardBlock) {
  if (resizing.value?.id !== block.id) return
  resizing.value = null
  ;(e.currentTarget as HTMLElement).releasePointerCapture(e.pointerId)
}

function clearSelectionOnCanvasBackground(e: PointerEvent) {
  if (e.button !== 0) return
  selectedId.value = null
}
</script>

<template>
  <div class="editor-root">
    <input
      ref="imageInputRef"
      type="file"
      class="sr-only"
      accept="image/*"
      aria-hidden="true"
      tabindex="-1"
      @change="onImageFileChange"
    />
    <input
      ref="projectInputRef"
      type="file"
      class="sr-only"
      accept="application/json,.json"
      aria-hidden="true"
      tabindex="-1"
      @change="onProjectFileChange"
    />

    <aside class="sidebar" aria-label="Tools and properties">
      <header class="sidebar-head">
        <h1 class="title">Business card</h1>
        <p class="subtitle">Two pages — thumbnails below switch which side you edit.</p>
      </header>
      <section class="project-bar" aria-label="Project file">
        <h2 class="section-label">Project</h2>
        <div class="btn-row">
          <button type="button" class="btn-secondary" :disabled="projectBusy" @click="saveProject">
            Save…
          </button>
          <button type="button" class="btn-secondary" :disabled="projectBusy" @click="openProjectPicker">
            Load…
          </button>
        </div>
        <button
          type="button"
          class="btn-secondary btn-full btn-export-pdf"
          :disabled="exportPdfBusy || projectBusy"
          @click="exportCardToPdf"
        >
          {{ exportPdfBusy ? 'Exporting PDF…' : 'Export PDF (3.5×2 in)' }}
        </button>
        <p v-if="projectMessage" class="project-msg" role="status">{{ projectMessage }}</p>
      </section>

      <div class="btn-row">
        <button type="button" class="btn-primary" @click="addTextBlock">+ Text</button>
        <button type="button" class="btn-secondary" @click="openImagePicker('add')">+ Image</button>
      </div>

      <section v-if="selected" class="panel" aria-label="Placement">
        <h2 class="section-label">Placement (%)</h2>
        <p class="placement-hint">Relative to the card. X/Y = top-left of the layer box.</p>
        <div class="placement-grid">
          <label class="placement-field">
            <span class="field-label">X</span>
            <input
              v-model.number="selected.x"
              type="number"
              min="0"
              max="100"
              step="0.5"
              class="placement-num"
            />
          </label>
          <label class="placement-field">
            <span class="field-label">Y</span>
            <input
              v-model.number="selected.y"
              type="number"
              min="0"
              max="100"
              step="0.5"
              class="placement-num"
            />
          </label>
          <label class="placement-field">
            <span class="field-label">Width</span>
            <input
              v-model.number="selected.widthPct"
              type="number"
              min="6"
              max="100"
              step="0.5"
              class="placement-num"
            />
          </label>
          <label class="placement-field">
            <span class="field-label">Height</span>
            <input
              v-model.number="selected.heightPct"
              type="number"
              min="6"
              max="100"
              step="0.5"
              class="placement-num"
            />
          </label>
        </div>
      </section>

      <section v-if="selectedText" class="panel" aria-label="Edit text">
        <h2 class="section-label">Edit text</h2>

        <label class="field">
          <span class="field-label">Content</span>
          <textarea
            v-model="selectedText.content"
            class="textarea"
            rows="4"
            placeholder="Type your text…"
          />
        </label>

        <label class="field">
          <span class="field-label">Font</span>
          <select v-model="selectedText.fontFamily" class="select font-select">
            <option
              v-for="opt in TEXT_FONT_OPTIONS"
              :key="opt.value"
              :value="opt.value"
              :style="{ fontFamily: opt.value }"
            >
              {{ opt.label }}
            </option>
          </select>
        </label>

        <label class="field">
          <span class="field-label">Size (px) — {{ selectedText.fontSize }}</span>
          <input v-model.number="selectedText.fontSize" type="range" min="8" max="48" step="1" class="range" />
        </label>

        <label class="field">
          <span class="field-label">Color</span>
          <input v-model="selectedText.color" type="color" class="color-input" />
        </label>

        <fieldset class="row">
          <label class="check">
            <input v-model="selectedText.fontWeight" type="checkbox" true-value="700" false-value="400" />
            Bold
          </label>
          <label class="check">
            <input
              v-model="selectedText.fontStyle"
              type="checkbox"
              true-value="italic"
              false-value="normal"
            />
            Italic
          </label>
        </fieldset>

        <label class="field">
          <span class="field-label">Alignment</span>
          <select v-model="selectedText.textAlign" class="select">
            <option value="left">Left</option>
            <option value="center">Center</option>
            <option value="right">Right</option>
          </select>
        </label>

        <button type="button" class="btn-danger" @click="removeSelected">Remove block</button>
      </section>

      <section v-else-if="selectedImage" class="panel" aria-label="Edit image">
        <h2 class="section-label">Edit image</h2>

        <div class="thumb-preview">
          <img :src="selectedImage.src" alt="Preview" />
        </div>

        <button type="button" class="btn-secondary btn-full" @click="openImagePicker('replace')">
          Replace image…
        </button>

        <label class="field">
          <span class="field-label">Opacity — {{ Math.round(selectedImage.opacity * 100) }}%</span>
          <input v-model.number="selectedImage.opacity" type="range" min="0.1" max="1" step="0.05" class="range" />
        </label>

        <label class="field">
          <span class="field-label">Corner radius — {{ selectedImage.borderRadius }}px</span>
          <input v-model.number="selectedImage.borderRadius" type="range" min="0" max="48" step="1" class="range" />
        </label>

        <label class="field">
          <span class="field-label">Brightness — {{ selectedImage.brightness }}%</span>
          <input v-model.number="selectedImage.brightness" type="range" min="40" max="160" step="5" class="range" />
        </label>

        <label class="field">
          <span class="field-label">Fit</span>
          <select v-model="selectedImage.objectFit" class="select">
            <option value="cover">Cover (crop to fill)</option>
            <option value="contain">Contain (letterbox)</option>
            <option value="fill">Stretch</option>
          </select>
        </label>
        <button type="button" class="btn-danger" @click="removeSelected">Remove block</button>
      </section>
      <p v-else class="empty-hint">Select a layer to edit its properties.</p>
    </aside>

    <main class="canvas-wrap">
      <div class="zoom-toolbar" role="toolbar" aria-label="Editor zoom">
        <button type="button" class="zoom-btn" aria-label="Zoom out" @click="zoomOut">−</button>
        <label class="zoom-slider-wrap">
          <span class="sr-only">Zoom level</span>
          <input
            v-model.number="zoomPercentUi"
            type="range"
            class="zoom-slider"
            min="25"
            max="400"
            step="5"
          />
        </label>
        <span class="zoom-label">{{ zoomPercentUi }}%</span>
        <button type="button" class="zoom-btn" aria-label="Zoom in" @click="zoomIn">+</button>
        <button type="button" class="zoom-reset" @click="zoomReset">Reset</button>
      </div>

      <div
        ref="canvasViewportRef"
        class="canvas-viewport"
        @pointerdown.self="clearSelectionOnCanvasBackground"
      >
        <div
          class="zoom-shell"
          :style="{
            width: `${CARD_BASE_W * editorZoom}px`,
            height: `${CARD_BASE_H * editorZoom}px`,
          }"
        >
          <div
            class="zoom-inner"
            :style="{
              width: `${CARD_BASE_W}px`,
              height: `${CARD_BASE_H}px`,
              transform: `scale(${editorZoom})`,
            }"
          >
            <div
              ref="cardRef"
              class="card"
              role="presentation"
              @pointerdown.self="clearSelectionOnCanvasBackground"
            >
        <template v-for="(b, i) in activeBlocksList" :key="b.id">
          <div
            v-if="b.kind === 'image'"
            class="card-item card-item--image"
            :class="{ selected: selectedId === b.id }"
            :style="{
              zIndex: i,
              left: `${b.x}%`,
              top: `${b.y}%`,
              width: `${b.widthPct}%`,
              height: `${b.heightPct}%`,
              opacity: b.opacity,
              borderRadius: `${b.borderRadius}px`,
            }"
            @pointerdown="onBlockPointerDown($event, b)"
            @pointermove="onBlockPointerMove($event, b)"
            @pointerup="onBlockPointerUp($event, b)"
          >
            <div class="card-item__media-clip">
              <img
                :src="b.src"
                alt=""
                class="card-item__img"
                :style="{
                  objectFit: b.objectFit,
                  filter: `brightness(${b.brightness}%)`,
                }"
                draggable="false"
              />
            </div>
            <template v-if="selectedId === b.id">
              <div
                class="resize-handle nw"
                aria-hidden="true"
                @pointerdown.stop="onResizePointerDown($event, b, 'nw')"
                @pointermove="onResizePointerMove($event, b)"
                @pointerup="onResizePointerUp($event, b)"
                @pointercancel="onResizePointerCancel($event, b)"
              />
              <div
                class="resize-handle ne"
                aria-hidden="true"
                @pointerdown.stop="onResizePointerDown($event, b, 'ne')"
                @pointermove="onResizePointerMove($event, b)"
                @pointerup="onResizePointerUp($event, b)"
                @pointercancel="onResizePointerCancel($event, b)"
              />
              <div
                class="resize-handle sw"
                aria-hidden="true"
                @pointerdown.stop="onResizePointerDown($event, b, 'sw')"
                @pointermove="onResizePointerMove($event, b)"
                @pointerup="onResizePointerUp($event, b)"
                @pointercancel="onResizePointerCancel($event, b)"
              />
              <div
                class="resize-handle se"
                aria-hidden="true"
                @pointerdown.stop="onResizePointerDown($event, b, 'se')"
                @pointermove="onResizePointerMove($event, b)"
                @pointerup="onResizePointerUp($event, b)"
                @pointercancel="onResizePointerCancel($event, b)"
              />
            </template>
          </div>
          <div
            v-else
            class="card-item card-item--text"
            :class="{ selected: selectedId === b.id }"
            :style="{
              zIndex: i,
              left: `${b.x}%`,
              top: `${b.y}%`,
              width: `${b.widthPct}%`,
              height: `${b.heightPct}%`,
            }"
            @pointerdown="onBlockPointerDown($event, b)"
            @pointermove="onBlockPointerMove($event, b)"
            @pointerup="onBlockPointerUp($event, b)"
          >
            <div
              class="card-item__text"
              :style="{
                fontFamily: b.fontFamily,
                fontSize: `${b.fontSize}px`,
                color: b.color,
                fontWeight: b.fontWeight,
                fontStyle: b.fontStyle,
                textAlign: b.textAlign,
              }"
            >
              {{ b.content }}
            </div>
            <template v-if="selectedId === b.id">
              <div
                class="resize-handle nw"
                aria-hidden="true"
                @pointerdown.stop="onResizePointerDown($event, b, 'nw')"
                @pointermove="onResizePointerMove($event, b)"
                @pointerup="onResizePointerUp($event, b)"
                @pointercancel="onResizePointerCancel($event, b)"
              />
              <div
                class="resize-handle ne"
                aria-hidden="true"
                @pointerdown.stop="onResizePointerDown($event, b, 'ne')"
                @pointermove="onResizePointerMove($event, b)"
                @pointerup="onResizePointerUp($event, b)"
                @pointercancel="onResizePointerCancel($event, b)"
              />
              <div
                class="resize-handle sw"
                aria-hidden="true"
                @pointerdown.stop="onResizePointerDown($event, b, 'sw')"
                @pointermove="onResizePointerMove($event, b)"
                @pointerup="onResizePointerUp($event, b)"
                @pointercancel="onResizePointerCancel($event, b)"
              />
              <div
                class="resize-handle se"
                aria-hidden="true"
                @pointerdown.stop="onResizePointerDown($event, b, 'se')"
                @pointermove="onResizePointerMove($event, b)"
                @pointerup="onResizePointerUp($event, b)"
                @pointercancel="onResizePointerCancel($event, b)"
              />
            </template>
          </div>
        </template>
            </div>
          </div>
        </div>
      </div>
      <nav class="page-strip" aria-label="Pages">
        <button
          v-for="(page, pi) in pages"
          :key="page.id"
          type="button"
          class="page-thumb-btn"
          :class="{ active: pi === activePageIndex }"
          :aria-current="pi === activePageIndex ? 'page' : undefined"
          @click="setActivePage(pi)"
        >
          <div
            class="page-thumb-viewport"
            :style="{ width: `${PAGE_THUMB_FRAME_W}px`, height: `${PAGE_THUMB_FRAME_H}px` }"
          >
            <div
              class="page-thumb-scaler"
              :style="{
                width: `${CARD_BASE_W}px`,
                height: `${CARD_BASE_H}px`,
                transform: `scale(${PAGE_THUMB_SCALE})`,
              }"
            >
              <div class="card card--thumb">
                <template v-for="(tb, ti) in page.blocks" :key="tb.id">
                  <div
                    v-if="tb.kind === 'image'"
                    class="card-item card-item--image card-item--thumb-preview"
                    :style="{
                      zIndex: ti,
                      left: `${tb.x}%`,
                      top: `${tb.y}%`,
                      width: `${tb.widthPct}%`,
                      height: `${tb.heightPct}%`,
                      opacity: tb.opacity,
                      borderRadius: `${tb.borderRadius}px`,
                    }"
                  >
                    <div class="card-item__media-clip">
                      <img
                        :src="tb.src"
                        alt=""
                        class="card-item__img"
                        :style="{
                          objectFit: tb.objectFit,
                          filter: `brightness(${tb.brightness}%)`,
                        }"
                        draggable="false"
                      />
                    </div>
                  </div>
                  <div
                    v-else
                    class="card-item card-item--text card-item--thumb-preview"
                    :style="{
                      zIndex: ti,
                      left: `${tb.x}%`,
                      top: `${tb.y}%`,
                      width: `${tb.widthPct}%`,
                      height: `${tb.heightPct}%`,
                    }"
                  >
                    <div
                      class="card-item__text"
                      :style="{
                        fontFamily: tb.fontFamily,
                        fontSize: `${tb.fontSize}px`,
                        color: tb.color,
                        fontWeight: tb.fontWeight,
                        fontStyle: tb.fontStyle,
                        textAlign: tb.textAlign,
                      }"
                    >
                      {{ tb.content }}
                    </div>
                  </div>
                </template>
              </div>
            </div>
          </div>
          <span class="page-thumb-label">Page {{ pi + 1 }}</span>
        </button>
      </nav>
      <p class="hint">
        3.5 × 2 in ratio — thumbnails below switch pages. Drag to move; corners resize. Zoom with the bar or
        <kbd class="kbd">Ctrl</kbd> / <kbd class="kbd">⌘</kbd> + scroll. Layer stack lists front-most at the top.
      </p>
    </main>

    <aside class="sidebar-right" aria-label="Layer stack">
      <header class="right-panel-head">
        <h2 class="title title-sm">Layer stack</h2>
        <p class="stack-hint">
          Top row is in front on the canvas; bottom is behind. ↑ / ↓ move the layer in the stack.
        </p>
      </header>

      <section class="right-section" aria-label="Stack order">
        <h3 class="section-label">Stack order</h3>
        <ul class="layer-pos-list">
          <li v-for="{ block: b, actualIndex } in layersFrontFirst" :key="b.id" class="layer-pos-row">
            <button
              type="button"
              class="layer-pos-name"
              :class="{ active: selectedId === b.id }"
              @click="selectedId = b.id"
            >
              <span v-if="b.kind === 'image'" class="layer-pos-thumb-wrap">
                <img :src="b.src" alt="" class="layer-pos-thumb" />
              </span>
              <span v-else class="layer-pos-type">T</span>
              <span class="layer-pos-label">{{ layerLabel(b) }}</span>
            </button>
            <div class="layer-pos-arrows" role="group" :aria-label="`Reorder ${layerLabel(b)}`">
              <button
                type="button"
                class="layer-arrow-btn"
                :disabled="actualIndex >= activeBlocksList.length - 1"
                title="In front (closer to viewer)"
                aria-label="Move layer in front of the next one down"
                @click="moveLayerAt(actualIndex, 1)"
              >
                ↑
              </button>
              <button
                type="button"
                class="layer-arrow-btn"
                :disabled="actualIndex === 0"
                title="Behind (under the next one up)"
                aria-label="Move layer behind the next one up"
                @click="moveLayerAt(actualIndex, -1)"
              >
                ↓
              </button>
            </div>
          </li>
        </ul>
        <p v-if="activeBlocksList.length === 0" class="empty-hint">No layers yet.</p>
      </section>
    </aside>
  </div>
</template>

<style scoped>
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

.editor-root {
  min-height: 100vh;
  display: flex;
  background: #e8eaef;
  font-family: system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif;
  color: #1a1a1a;
}

.sidebar {
  width: 300px;
  flex-shrink: 0;
  background: #f4f5f8;
  border-right: 1px solid #d8dce4;
  padding: 1.25rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
  overflow-y: auto;
  max-height: 100vh;
}

.sidebar-right {
  width: 280px;
  flex-shrink: 0;
  background: #f4f5f8;
  border-left: 1px solid #d8dce4;
  padding: 1.25rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
  overflow-y: auto;
  max-height: 100vh;
}

.right-panel-head {
  margin-bottom: 0;
}

.title-sm {
  font-size: 1rem;
  margin-bottom: 0.35rem;
}

.right-section {
  display: flex;
  flex-direction: column;
  gap: 0.65rem;
  padding-top: 0.25rem;
  border-top: 1px solid #e2e5eb;
}

.layer-pos-list {
  list-style: none;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
}

.layer-pos-row {
  display: flex;
  align-items: stretch;
  gap: 0.35rem;
}

.layer-pos-name {
  flex: 1;
  min-width: 0;
  display: flex;
  align-items: center;
  gap: 0.45rem;
  padding: 0.45rem 0.5rem;
  border: 1px solid #e2e5eb;
  border-radius: 8px;
  background: #fff;
  cursor: pointer;
  text-align: left;
  font-size: 0.75rem;
}

.layer-pos-name:hover {
  border-color: #c7ced9;
}

.layer-pos-name.active {
  border-color: #2563eb;
  box-shadow: 0 0 0 1px #2563eb;
}

.layer-pos-thumb-wrap {
  flex-shrink: 0;
  width: 28px;
  height: 22px;
  border-radius: 3px;
  overflow: hidden;
  background: #e5e7eb;
}

.layer-pos-thumb {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
}

.layer-pos-type {
  flex-shrink: 0;
  width: 22px;
  height: 22px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 4px;
  background: #e0e7ff;
  color: #3730a3;
  font-size: 0.65rem;
  font-weight: 700;
}

.layer-pos-label {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.layer-pos-arrows {
  display: flex;
  flex-direction: column;
  gap: 2px;
  flex-shrink: 0;
}

.layer-arrow-btn {
  width: 1.75rem;
  min-height: 1.35rem;
  padding: 0;
  font-size: 0.65rem;
  line-height: 1;
  border: 1px solid #d1d5db;
  border-radius: 4px;
  background: #fff;
  color: #374151;
  cursor: pointer;
}

.layer-arrow-btn:hover:not(:disabled) {
  background: #f3f4f6;
  border-color: #9ca3af;
}

.layer-arrow-btn:disabled {
  opacity: 0.35;
  cursor: not-allowed;
}

.placement-hint {
  margin: -0.35rem 0 0;
  font-size: 0.6875rem;
  color: #9ca3af;
  line-height: 1.35;
}

.placement-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.65rem 0.75rem;
}

.placement-field {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.placement-num {
  width: 100%;
  padding: 0.4rem 0.45rem;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 0.8125rem;
}

.sidebar-head {
  margin-bottom: 0.25rem;
}

.title {
  font-size: 1.125rem;
  font-weight: 700;
  margin: 0 0 0.35rem;
  letter-spacing: -0.02em;
}

.subtitle {
  margin: 0;
  font-size: 0.8125rem;
  color: #5c6370;
  line-height: 1.4;
}

.project-bar {
  padding-bottom: 0.75rem;
  margin-bottom: 0.25rem;
  border-bottom: 1px solid #e2e5eb;
}

.btn-export-pdf {
  margin-top: 0.5rem;
}

.project-msg {
  margin: 0.5rem 0 0;
  font-size: 0.75rem;
  color: #4b5563;
  line-height: 1.35;
}

.btn-row {
  display: flex;
  gap: 0.5rem;
}

.btn-primary {
  flex: 1;
  padding: 0.6rem 0.75rem;
  background: #2563eb;
  color: #fff;
  border: none;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
}

.btn-primary:hover {
  background: #1d4ed8;
}

.btn-secondary {
  flex: 1;
  padding: 0.6rem 0.75rem;
  background: #fff;
  color: #1e3a8a;
  border: 1px solid #bfdbfe;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
}

.btn-secondary:hover {
  background: #eff6ff;
}

.btn-full {
  width: 100%;
}

.btn-danger {
  margin-top: 0.5rem;
  padding: 0.5rem 0.75rem;
  background: transparent;
  color: #b91c1c;
  border: 1px solid #fecaca;
  border-radius: 8px;
  font-size: 0.8125rem;
  cursor: pointer;
}

.btn-danger:hover {
  background: #fef2f2;
}

.btn-ghost {
  flex: 1;
  padding: 0.45rem 0.5rem;
  font-size: 0.75rem;
  background: #fff;
  border: 1px solid #e2e5eb;
  border-radius: 6px;
  cursor: pointer;
  color: #374151;
}

.btn-ghost:hover {
  border-color: #c7ced9;
  background: #f9fafb;
}

.section-label {
  font-size: 0.6875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: #6b7280;
  margin: 0 0 0.5rem;
}

.stack-hint {
  margin: -0.25rem 0 0.5rem;
  font-size: 0.6875rem;
  color: #9ca3af;
  line-height: 1.3;
}

.block-list ul {
  list-style: none;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
}

.layer-btn {
  width: 100%;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  text-align: left;
  padding: 0.5rem 0.65rem;
  border: 1px solid #e2e5eb;
  border-radius: 8px;
  background: #fff;
  cursor: pointer;
  font-size: 0.8125rem;
}

.layer-btn:hover {
  border-color: #c7ced9;
}

.layer-btn.active {
  border-color: #2563eb;
  box-shadow: 0 0 0 1px #2563eb;
}

.layer-thumb-wrap {
  flex-shrink: 0;
  width: 36px;
  height: 28px;
  border-radius: 4px;
  overflow: hidden;
  background: #e5e7eb;
  border: 1px solid #d1d5db;
}

.layer-thumb {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
}

.layer-preview {
  flex: 1;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.panel {
  padding-top: 0.5rem;
  border-top: 1px solid #e2e5eb;
  display: flex;
  flex-direction: column;
  gap: 0.85rem;
}

.thumb-preview {
  border-radius: 8px;
  overflow: hidden;
  border: 1px solid #e2e5eb;
  background: repeating-conic-gradient(#f3f4f6 0% 25%, #e5e7eb 0% 50%) 50% / 12px 12px;
  max-height: 120px;
}

.thumb-preview img {
  display: block;
  width: 100%;
  max-height: 120px;
  object-fit: contain;
}

.layer-actions {
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
}

.stack-btns {
  display: flex;
  gap: 0.35rem;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
}

.field-label {
  font-size: 0.75rem;
  font-weight: 500;
  color: #4b5563;
}

.textarea {
  resize: vertical;
  min-height: 4.5rem;
  padding: 0.5rem 0.65rem;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  font-size: 0.875rem;
  font-family: inherit;
}

.textarea:focus {
  outline: none;
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.15);
}

.range {
  width: 100%;
  accent-color: #2563eb;
}

.color-input {
  width: 100%;
  height: 40px;
  padding: 2px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  cursor: pointer;
}

.row {
  border: none;
  margin: 0;
  padding: 0;
  display: flex;
  gap: 1rem;
}

.check {
  font-size: 0.8125rem;
  display: flex;
  align-items: center;
  gap: 0.4rem;
  cursor: pointer;
}

.select {
  padding: 0.45rem 0.65rem;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  font-size: 0.875rem;
  background: #fff;
}

.font-select {
  font-size: 0.9375rem;
}

.pos-row {
  display: flex;
  gap: 1rem;
  font-size: 0.8125rem;
  align-items: center;
}

.num {
  width: 4.5rem;
  margin-left: 0.25rem;
  padding: 0.35rem 0.5rem;
  border: 1px solid #d1d5db;
  border-radius: 6px;
}

.empty-hint {
  margin: 0;
  font-size: 0.8125rem;
  color: #6b7280;
}

.canvas-wrap {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: stretch;
  justify-content: flex-start;
  min-width: 0;
  min-height: 0;
  padding: 1.25rem 2rem 2rem;
  gap: 0;
}

.zoom-toolbar {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-bottom: 0.75rem;
}

.zoom-btn {
  width: 2rem;
  height: 2rem;
  padding: 0;
  font-size: 1.2rem;
  line-height: 1;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  background: #fff;
  cursor: pointer;
  color: #1f2937;
}

.zoom-btn:hover {
  background: #f3f4f6;
}

.zoom-reset {
  font-size: 0.75rem;
  font-weight: 600;
  padding: 0.4rem 0.65rem;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  background: #fff;
  cursor: pointer;
  color: #374151;
}

.zoom-reset:hover {
  background: #f3f4f6;
}

.zoom-slider-wrap {
  flex: 1;
  min-width: 100px;
  max-width: 200px;
}

.zoom-slider {
  width: 100%;
  vertical-align: middle;
  accent-color: #2563eb;
}

.zoom-label {
  font-size: 0.8125rem;
  font-weight: 600;
  color: #4b5563;
  min-width: 2.85rem;
  text-align: right;
}

.canvas-viewport {
  flex: 1;
  min-height: 0;
  overflow: auto;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1.5rem;
  background: #dfe3ea;
  border-radius: 12px;
  border: 1px solid #d1d5db;
}

.zoom-shell {
  position: relative;
  flex-shrink: 0;
}

.zoom-inner {
  position: absolute;
  top: 0;
  left: 0;
  transform-origin: 0 0;
}

.card {
  position: relative;
  width: 100%;
  height: 100%;
  background: #fff;
  border-radius: 4px;
  box-shadow:
    0 1px 3px rgba(0, 0, 0, 0.08),
    0 12px 40px rgba(0, 0, 0, 0.12);
  overflow: hidden;
  border: 1px solid #e5e7eb;
}

.card-item {
  position: absolute;
  cursor: grab;
  touch-action: none;
  user-select: none;
}

.card-item:active {
  cursor: grabbing;
}

.card-item.selected {
  outline: 2px dashed #2563eb;
  outline-offset: 2px;
}

.card-item--image {
  overflow: visible;
}

.card-item__media-clip {
  position: absolute;
  inset: 0;
  overflow: hidden;
  border-radius: inherit;
  pointer-events: none;
}

.card-item__img {
  display: block;
  width: 100%;
  height: 100%;
  pointer-events: none;
  user-select: none;
}

.card-item__text {
  width: 100%;
  height: 100%;
  overflow: hidden;
  line-height: 1.25;
  word-break: break-word;
  pointer-events: none;
}

.resize-handle {
  --resize-hit: 36px;
  --resize-knob: 11px;
  position: absolute;
  width: var(--resize-hit);
  height: var(--resize-hit);
  margin: 0;
  padding: 0;
  background: transparent;
  border: none;
  z-index: 5;
  touch-action: none;
}

.resize-handle::after {
  content: '';
  position: absolute;
  width: var(--resize-knob);
  height: var(--resize-knob);
  left: calc(var(--resize-hit) / 2 - var(--resize-knob));
  top: calc(var(--resize-hit) / 2 - var(--resize-knob));
  background: #fff;
  border: 2px solid #2563eb;
  border-radius: 2px;
  pointer-events: none;
}

.resize-handle.nw {
  top: calc(var(--resize-hit) / -2);
  left: calc(var(--resize-hit) / -2);
  cursor: nwse-resize;
}

.resize-handle.ne {
  top: calc(var(--resize-hit) / -2);
  right: calc(var(--resize-hit) / -2);
  cursor: nesw-resize;
}

.resize-handle.sw {
  bottom: calc(var(--resize-hit) / -2);
  left: calc(var(--resize-hit) / -2);
  cursor: nesw-resize;
}

.resize-handle.se {
  bottom: calc(var(--resize-hit) / -2);
  right: calc(var(--resize-hit) / -2);
  cursor: nwse-resize;
}

.page-strip {
  display: flex;
  flex-wrap: wrap;
  gap: 0.65rem;
  justify-content: center;
  align-items: flex-end;
  padding: 0.85rem 1rem 0;
  flex-shrink: 0;
}

.page-thumb-btn {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.35rem;
  padding: 0.4rem 0.5rem 0.35rem;
  border: 2px solid transparent;
  border-radius: 10px;
  background: transparent;
  cursor: pointer;
  font: inherit;
  color: #374151;
}

.page-thumb-btn:hover {
  background: rgba(37, 99, 235, 0.06);
}

.page-thumb-btn.active {
  border-color: #2563eb;
  background: #eef4ff;
  box-shadow: 0 0 0 1px rgba(37, 99, 235, 0.25);
}

.page-thumb-viewport {
  overflow: hidden;
  border-radius: 4px;
  border: 1px solid #cbd5e1;
  background: #fff;
  flex-shrink: 0;
}

.page-thumb-scaler {
  transform-origin: 0 0;
  pointer-events: none;
}

.card--thumb {
  position: relative;
  width: 100%;
  height: 100%;
  border-radius: 4px;
  overflow: hidden;
  border: 1px solid #e5e7eb;
  box-shadow: none;
}

.card-item--thumb-preview {
  cursor: default;
  pointer-events: none;
}

.page-thumb-label {
  font-size: 0.6875rem;
  font-weight: 600;
  color: #6b7280;
}

.page-thumb-btn.active .page-thumb-label {
  color: #1d4ed8;
}

.hint {
  margin: 0.75rem 0 0;
  font-size: 0.8125rem;
  color: #6b7280;
  line-height: 1.45;
}

.kbd {
  display: inline-block;
  padding: 0.1em 0.4em;
  font-size: 0.75em;
  font-family: inherit;
  background: #e5e7eb;
  border: 1px solid #d1d5db;
  border-radius: 4px;
  box-shadow: 0 1px 0 #cbd5e1;
}
</style>
