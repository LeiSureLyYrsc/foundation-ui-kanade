<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, provide, reactive, ref, watch } from 'vue'
import MIcon from './MIcon.vue'
import MTrayRow from './MTrayRow.vue'
import { useTrayPointer } from '../tray/trayPointer.js'
import { alignedScrollLeft, nearestScrollTop } from '../tray/trayPager.js'
import { ITEMS, createNav, isEnabled, isVisible, labelOf as labelFor, openingDensity, pageTitle as titleFor, shortOf as shortFor, tx } from '../tray/trayModel.js'

// The tray menu (docs/tray-menu.md): the three layouts of src/tray/trayModel.js drawn for a state. One component, one surface (the card of the Panel's menu window); the pages of a
// layout slide inside it. A stack (V1, V2) drills in with a shared-axis slide and backs out; a pager (V3) moves along its track and shows page dots. Everything it does goes out as `act(id)`:
// the host runs the item, the component keeps no function and no state of the host.
//   emits: act(id)   an item was chosen        page(name)   the page changed        close()   Escape on the first page        size({ w, h })   the card's size (CSS px) once the page that is on has
//   its height - the host sizes its window to it BEFORE the card grows (the card waits for `sized` to answer) and after it has shrunk: one resize per page change, never one per frame.
// The card changes height with transforms only, so a page change costs the compositor a few layers and the main thread nothing per frame. The card is laid out at the larger of the two heights at
// once, inside a fixed clip (.m-tray__clip); its surface (.m-tray__surface: colour, radius) is moved by translateY from the old height to the new one, so its edge away from the taskbar is the moving
// edge and its own rounded corners come with it; the page rides that edge (or stays, when the taskbar is at the top and the edge is the bottom one), the dots and the footer stay at the taskbar side,
// and the shadow and the 1 px edge (.m-tray__shade) are scaled to the visible height from the taskbar side. No layout, paint or raster per frame.
const props = defineProps({
  model: { type: Object, required: true },
  state: { type: Object, required: true },
  page: { type: String, default: '' }, // the page to start on
  motion: { type: Boolean, default: true },
  focus: { type: Boolean, default: false }, // take the keyboard focus on the first item when mounted (the popup window)
  edit: { type: Boolean, default: false }, // start in edit mode (the gallery; the quick page only)
  apply: { type: Function, default: null }, // async (key, value): a slider was released; resolves when the host has set it, throws the host's words when it could not
  edge: { type: String, default: 'bottom' }, // where the taskbar is ('bottom' | 'top' | 'left' | 'right'): the card is anchored to that side, the other side moves when the height changes
  maxHeight: { type: Number, default: 0 }, // CSS px the window can have at most (the host's `maxHeight`, the work area less its margins); a page taller than this scrolls inside the card; 0 = no limit
  density: { type: String, default: '' }, // 'auto' | 'standard' | 'touch': the person's choice; empty = state.density, else auto
  sized: { type: Function, default: null }, // async ({ w, h }): the host sizes its window to the card; a card that grows waits for it (the window is resized once, before anything moves)
})
const emit = defineEmits(['act', 'page', 'close', 'size', 'layout', 'drag'])

const nav = reactive(createNav(props.model, props.page))
// Density: decided once, when the menu opens (the host's facts about the press that opened it, then the person's choice); with 'auto', the first pointer of the page corrects the guess after its press.
// `input` is what the last pointer was: a touch has no hover and no sticky hover state, a mouse is as it always was.
const densityPref = props.density || props.state?.density || 'auto'
const touch = ref(openingDensity(densityPref, props.state?.touch) === 'touch')
const input = ref(touch.value ? 'touch' : 'mouse')
const densifying = ref(false)
let densifyTimer = 0
provide('trayTouch', touch)
function setDensity(next) {
  if (touch.value === next) return
  densifying.value = true; clearTimeout(densifyTimer); densifyTimer = setTimeout(() => { densifying.value = false }, 520)
  touch.value = next
}
let corrected = densityPref !== 'auto' // a choice of the person is not corrected
const root = ref(null)
const surface = ref(null)
const shade = ref(null)
const viewport = ref(null)
const chrome = ref(null) // pager: the dots and the footer
const isPager = computed(() => props.model.kind === 'pager')
const t = computed(() => tx(props.state))
// the layout of the quick page: what the host stored (state.layout), and what the person has just set until the host's state says the same
const mine = ref(null)
const viewState = computed(() => (mine.value ? { ...props.state, layout: mine.value } : props.state))
watch(() => JSON.stringify(props.state?.layout ?? null), (now) => { if (mine.value && JSON.stringify(mine.value) === now) mine.value = null })
function setLayout(record) { mine.value = record; emit('layout', record) }
// edit mode: the quick page shows the layout editor in place of its rows
const editing = ref(props.edit && !!props.model.edit)
const canEdit = computed(() => !!props.model.edit && nav.current === props.model.edit)
function toggleEdit() { editing.value = !editing.value; nextTick(() => { if (props.focus) requestAnimationFrame(focusFirst) }) }
watch(() => nav.current, (page) => { if (page !== props.model.edit) editing.value = false })
const pageOf = (id) => props.model.pages[id]
const rowsOf = (id) => (editing.value && id === props.model.edit ? [{ t: 'editor' }] : pageOf(id).rows.filter((r) => isVisible(r, viewState.value)))
// a row keeps its key while the rows around it come and go, so the ones that stay move to their new place instead of being redrawn
const rowKey = (row, i) => (row.t === 'item' ? `item:${row.id}` : row.t === 'page' ? `page:${row.to}` : row.t === 'divider' ? `divider${i}` : row.t)
const labelOf = (id) => labelFor(id, props.state)
const shortOf = (id) => shortFor(id, props.state)
const iconOf = (id) => ITEMS[id]?.icon || 'circle'
const pageTitle = (id) => titleFor(id, props.state, props.model.variant)
const enabled = (id) => isEnabled(id, props.state)

function act(id) {
  if (!enabled(id)) return
  emit('act', id)
}
function open(to) { if (nav.open(to)) emit('page', nav.current) }
function back() { if (nav.back()) emit('page', nav.current) }
function go(i) { if (nav.go(i)) emit('page', nav.current) }

// The viewport takes the height of the page that is on. A change of page (and of the page's own content) moves the card's edge on the compositor (see the head): the card is laid out at the larger of the
// two heights, and how much of it is hidden on the moving side runs from the old value to the new one. The edge away from the taskbar moves; the page rides it and
// the dots and footer of the pager stay at the taskbar side, or ride the bottom edge when the taskbar is at the top.
const anchorTop = computed(() => props.edge === 'top')
let ro = null
let boxH = 0 // the height of the viewport as laid out now
let idle = true // no reveal is running
let wanted = -1 // the page height the card is going to
let timer = 0
let seq = 0
let last = ''
const pause = (ms) => new Promise((resolve) => setTimeout(resolve, ms))
// laid-out heights are fractions at a display scale of 125 or 150 %: offsetHeight rounds them, and a page measured a fraction short is clipped by its last row's edge; the card takes them rounded UP
const heightOf = (el) => (el ? Math.ceil(el.getBoundingClientRect().height - 0.01) : 0)
const chromeH = () => heightOf(chrome.value)
const motionMs = () => {
  if (!props.motion || typeof document === 'undefined') return 0
  const html = document.documentElement
  const reduced = html.dataset.motion === 'reduced' || (html.dataset.motion !== 'full' && window.matchMedia?.('(prefers-reduced-motion: reduce)').matches)
  if (reduced) return 0
  const raw = getComputedStyle(root.value).getPropertyValue('--m-spring-default-spatial-duration').trim()
  const n = parseFloat(raw)
  return Number.isFinite(n) ? (raw.endsWith('ms') ? n : n * 1000) : 0
}
const setBox = (h) => { boxH = h; if (viewport.value) viewport.value.style.height = `${h}px` }
// `px` is how much of the laid-out card is not there (yet, or any more): the surface is moved that far away from the taskbar side, out of the clip, and the shade is scaled to what is left. The numbers go to
// the four elements as inline transforms, not as custom properties on the root: a custom property inherits, and changing one on the root restyles the whole card on every page change.
function setCut(px, box) {
  const total = box + chromeH()
  const shift = anchorTop.value ? -px : px
  const put = (el, value) => { if (el) el.style.transform = value }
  put(surface.value, `translateY(${shift}px)`)
  put(shade.value, `scaleY(${total > 0 ? Math.max(0, total - px) / total : 1})`)
  put(anchorTop.value ? viewport.value : chrome.value, `translateY(${-shift}px)`) // the part that stays put against the surface's move
}
const moveWith = (on) => { for (const el of [surface.value, shade.value, viewport.value, chrome.value]) if (el) el.style.transition = on ? 'transform var(--m-spring-default-spatial-duration) var(--m-spring-default-spatial)' : 'none' }
// how much is hidden right now (the transform the compositor is at; getComputedStyle reads the running transition)
function cutNow() {
  const m = new DOMMatrixReadOnly(getComputedStyle(surface.value).transform)
  return Math.max(0, Math.abs(m.m42) || 0)
}
function announce(h) {
  const size = { w: root.value?.offsetWidth || 0, h: h + chromeH() }
  const key = `${size.w}x${size.h}`
  if (key === last) return Promise.resolve()
  last = key
  emit('size', size)
  return props.sized ? Promise.resolve(props.sized(size)).catch(() => {}) : Promise.resolve()
}
function settle(h) {
  clearTimeout(timer)
  moveWith(false)
  setBox(h); setCut(0, h); idle = true; wanted = h
  return announce(h)
}
async function resize(next) {
  if (!root.value || !viewport.value) return
  if (!boxH) { settle(next); return }
  if (next === wanted) return
  const moving = !idle
  const ms = motionMs()
  if (!ms) { settle(next); return }
  const mine = ++seq
  const visible = moving ? boxH - cutNow() : boxH
  wanted = next
  // a card that grows has its window first: the native resize is done before anything moves, and the reveal starts on the frame after it
  if (next > boxH) await Promise.race([announce(next), pause(150)])
  if (mine !== seq) return
  const box = Math.max(boxH, next)
  moveWith(false)
  setBox(box); setCut(Math.max(0, box - visible), box)
  void surface.value.getBoundingClientRect() // the start state is on screen before the transition is switched on
  moveWith(true); idle = false
  setCut(box - next, box)
  clearTimeout(timer)
  timer = setTimeout(() => { if (mine === seq) settle(next) }, ms + 80)
}
const scrolls = ref(false) // the page is taller than the window can be: it scrolls inside the card (a 125 % display or a touch density on a small one)
// The card of a pager keeps ONE size: the height of its tallest page. A page change then moves the track and nothing else (no resize of the card, none of the host's window); a stack (V1, V2) takes the height of
// the page that is on, as it always did. A page that is clamped (the scrolling case: its neighbours are held to the viewport) reports its content height by scrollHeight, so the measure does not depend on the clamp.
const naturalOf = (el) => Math.max(heightOf(el), el.scrollHeight || 0)
// What a pager page is when nothing is open on it: a group's choices (.m-tray__sheet, under its pill, one grid gap of 8) and the sponsors (.m-tray__supchips--sponsors, one gap of 6) are opened in place on a page that
// is already as full as the card, so they are not part of the card's size: the page scrolls inside the card for as long as they are open and the card (and the host's window) keep their size.
const OPENED = [['.m-tray__sheet:not(.m-tray-sheet-leave-active)', 8], ['.m-tray__supchips--sponsors', 6]]
const closedOf = (el) => OPENED.reduce((h, [sel, gap]) => h - Array.from(el.querySelectorAll(sel), (x) => heightOf(x) + gap).reduce((a, b) => a + b, 0), naturalOf(el))
function measure() {
  const el = viewport.value?.querySelector(`[data-page="${nav.current}"]`)
  if (!el) return
  const cap = props.maxHeight > 0 ? Math.max(200, props.maxHeight - chromeH()) : Infinity
  const natural = isPager.value ? Math.max(...Array.from(viewport.value.querySelectorAll('[data-page]'), closedOf)) : heightOf(el)
  const box = Math.min(natural, cap)
  scrolls.value = natural > cap || (isPager.value && naturalOf(el) > box)
  resize(box)
}
watch(() => nav.current, () => { if (viewport.value) viewport.value.scrollTop = 0 })
onMounted(() => {
  measure()
  if (typeof ResizeObserver !== 'undefined') { ro = new ResizeObserver(measure); watchPageEls() }
  if (props.focus) nextTick(focusFirst)
})
function watchPageEls() {
  if (!ro) return
  ro.disconnect()
  viewport.value?.querySelectorAll('[data-page]').forEach((el) => ro.observe(el))
}
watch(() => nav.current, async () => { await nextTick(); watchPageEls(); measure() })
watch(() => nav.index, () => { keepAligned(); placeTrack(0) }, { flush: 'post' })
onMounted(() => placeTrack(0))
onBeforeUnmount(() => { ro?.disconnect(); clearTimeout(timer); clearTimeout(densifyTimer) })

// keyboard: Up / Down / Home / End move along the page's buttons, Tab leaves them in order; stack: Left / Backspace / Escape go back; pager: Left / Right change page; Escape on the first page closes
const pageEl = () => viewport.value?.querySelector(`[data-page="${nav.current}"]`)
const focusables = () => Array.from(pageEl()?.querySelectorAll('button:not(:disabled), input[type=range]:not(:disabled)') || []).filter((b) => b.offsetParent !== null || b === document.activeElement)
// Focus never scrolls the viewport sideways (see trayPager.js): it is given with preventScroll, and the page is scrolled up or down only, as far as the control needs.
function focusIn(el) {
  if (!el) return
  el.focus({ focusVisible: true, preventScroll: true })
  const vp = viewport.value
  if (!vp) return
  const v = vp.getBoundingClientRect(), r = el.getBoundingClientRect()
  const to = nearestScrollTop({ top: r.top, bottom: r.bottom, viewTop: v.top, viewBottom: v.bottom, scrollTop: vp.scrollTop })
  if (to !== vp.scrollTop) vp.scrollTop = to
}
// Whatever else scrolls the viewport sideways (Tab onto a control, a trackpad's sideways pan on a page that scrolls, a script) is undone before the frame is drawn: the scroll event runs ahead of the paint.
function keepAligned() { const vp = viewport.value; if (vp && vp.scrollLeft !== alignedScrollLeft) vp.scrollLeft = alignedScrollLeft }
function focusFirst() { focusIn(focusables()[0]) }
function moveFocus(delta, edge) {
  const list = focusables()
  if (!list.length) return
  const at = list.indexOf(document.activeElement)
  const to = edge === 'start' ? 0 : edge === 'end' ? list.length - 1 : at < 0 ? (delta > 0 ? 0 : list.length - 1) : (at + delta + list.length) % list.length
  focusIn(list[to])
}
// pointers: presses, ripples, swipes and holds are src/tray/trayPointer.js
const track = ref(null)
const { onDown, onMove, onUp, onCancel, onClickCapture, placeTrack } = useTrayPointer({
  root, viewport, track, nav, props, emit, isPager, canEdit, editing, toggleEdit, pageEl, go, back, input,
  corrected: () => corrected, firstPressDone: () => { corrected = true }, densityFix: setDensity,
})
function onKey(e) {
  const k = e.key
  // Escape in edit mode ends it (a picked-up item has already taken the key); the menu stays
  if (editing.value && k === 'Escape') { editing.value = false; e.stopPropagation(); e.preventDefault(); return }
  // on a slider Left / Right / Home / End move the value (the page keeps still); Up / Down still move along the controls
  if (e.target.matches?.('input[type=range]') && (k === 'ArrowLeft' || k === 'ArrowRight' || k === 'Home' || k === 'End')) return
  if (k === 'ArrowDown') { moveFocus(1); e.preventDefault(); return }
  if (k === 'ArrowUp') { moveFocus(-1); e.preventDefault(); return }
  if (k === 'Home') { moveFocus(0, 'start'); e.preventDefault(); return }
  if (k === 'End') { moveFocus(0, 'end'); e.preventDefault(); return }
  if (isPager.value) {
    const inSeg = !!e.target.closest?.('.m-tray__seg')
    if (k === 'ArrowRight' && !inSeg) { go(nav.index + 1); e.preventDefault() }
    else if (k === 'ArrowLeft' && !inSeg) { go(nav.index - 1); e.preventDefault() }
    else if (k === 'Escape') { if (nav.index > 0) { go(0); e.stopPropagation() } else emit('close') }
  } else if ((k === 'ArrowLeft' || k === 'Backspace') && nav.canBack) { back(); e.preventDefault() }
  else if (k === 'Escape') { if (nav.canBack) { back(); e.stopPropagation() } else emit('close') }
}
// after a page change the focus goes to the new page's first button (the old page's button is gone / inert)
watch(() => nav.current, async () => { if (!props.focus) return; await nextTick(); requestAnimationFrame(focusFirst) })
defineExpose({ nav, back, open, go, focusFirst, editing, touch })
</script>

<template>
  <div ref="root" class="m-tray" :class="[`m-tray--${model.variant}`, { 'is-still': !motion, 'is-dense': model.dense, 'is-top': edge === 'top', 'is-touch': touch, 'is-densifying': densifying }]" :data-input="input" role="menu" @keydown="onKey" @pointerdown="onDown" @pointermove="onMove" @pointerup="onUp" @pointercancel="onCancel" @click.capture="onClickCapture">
    <div ref="shade" class="m-tray__shade" aria-hidden="true" />
    <div class="m-tray__clip"><div ref="surface" class="m-tray__surface">
      <div ref="viewport" class="m-tray__viewport" :class="{ 'is-scroll': scrolls }" @scroll.passive="keepAligned">
        <!-- stack: the page that is on, a shared-axis slide between two -->
        <Transition v-if="!isPager" :name="nav.direction > 0 ? 'm-tray-fwd' : 'm-tray-bck'">
          <div :key="nav.current" class="m-tray__page" :data-page="nav.current">
            <div v-if="nav.canBack" class="m-tray__bar" :class="{ 'is-row': model.variant === 'v1' }">
              <button class="m-tray__back" type="button" :aria-label="t.back" @click="back"><MIcon name="arrow_back" /><span v-if="model.variant === 'v1'" class="m-tray__bar-title">{{ pageTitle(nav.current) }}</span></button>
              <span v-if="model.variant !== 'v1'" class="m-tray__bar-title">{{ pageTitle(nav.current) }}</span>
            </div>
            <slot v-if="nav.current === model.root" name="start-page" />
            <TransitionGroup name="m-tray-row">
              <MTrayRow v-for="(row, ri) in rowsOf(nav.current)" :key="rowKey(row, ri)" :row="row" :state="viewState" @act="act" @open="open" @layout="setLayout" @done="toggleEdit" :apply="apply" />
            </TransitionGroup>
          </div>
        </Transition>
        <!-- pager: every page side by side on a track; it is placed by `placeTrack` (the page's index, and the finger's offset while one moves it) -->
        <div v-else ref="track" class="m-tray__track">
          <div v-for="name in model.order" :key="name" class="m-tray__page" :data-page="name" :inert="name !== nav.current || undefined">
            <slot v-if="name === model.order[0] && !editing" name="start-page" />
            <TransitionGroup name="m-tray-row">
              <MTrayRow v-for="(row, ri) in rowsOf(name)" :key="rowKey(row, ri)" :row="row" :state="viewState" @act="act" @open="open" @layout="setLayout" @done="toggleEdit" :apply="apply" />
            </TransitionGroup>
          </div>
        </div>
      </div>
      <div v-if="isPager" ref="chrome" class="m-tray__chrome">
        <div class="m-tray__dots" role="tablist">
          <button v-for="(name, i) in model.order" :key="name" class="m-tray__dot" :class="{ 'is-on': i === nav.index }" type="button" role="tab" :aria-selected="i === nav.index" :aria-label="pageTitle(name)" @click="go(i)" />
        </div>
        <div class="m-tray__foot">
          <span class="m-tray__foot-title">{{ pageTitle(nav.current) }}</span>
          <button v-if="canEdit" class="m-tray__ibtn" :class="{ 'is-on': editing }" type="button" :aria-pressed="editing" :title="editing ? t.edit.done : t.edit.open" :aria-label="editing ? t.edit.done : t.edit.open" @click="toggleEdit"><MIcon :name="editing ? 'check' : 'edit'" /><span v-if="touch" class="m-tray__ibtn-t">{{ editing ? t.edit.done : t.edit.open }}</span></button>
          <button v-for="id in model.footer.ids" :key="id" class="m-tray__ibtn" :class="{ 'is-danger': ITEMS[id].danger }" type="button" :disabled="!enabled(id)" :title="labelOf(id)" :aria-label="labelOf(id)" @click="act(id)"><MIcon :name="iconOf(id)" /><span v-if="touch" class="m-tray__ibtn-t">{{ shortOf(id) }}</span></button>
        </div>
      </div>
    </div></div>
  </div>
</template>
