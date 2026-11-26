<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref } from 'vue'
import MTrayMenu from './MTrayMenu.vue'
import { installModality } from '../a11y/modality.js'
import { POPUP_PAD, applyThemeSnapshot } from '../contextmenu/popup.js'
import { VARIANTS } from '../tray/trayModel.js'

// The tray menu in the Panel's menu window (src-tauri/src/ctx_popup.rs + tray/popup_input.rs): the page of that window shows this next to MMenuPopup. The host opens it from the tray icon
// with the variant, the state (host state, streams, display, update, notification, DS5, the words of its native menu) and where the taskbar is; this page draws the menu, tells the host its size
// so that it sizes and places the window against the taskbar and shows it, and gives back the item that was chosen (the host runs it through the handler of its native menu: same ids).
//
//   transport = {
//     onOpen(cb)   cb({ token, variant, state, theme, edge }) for every open; returns a function that stops listening      edge: 'bottom' | 'top' | 'left' | 'right' (where the taskbar is)
//     onState(cb)  cb({ token, state }) the state changed while the menu is on
//     onClose(cb)  cb({ token }) the host closed the menu
//     ready(token, { w, h })  the card's size in CSS px; the host sizes the window to it (+ the room for the shadow), places it and shows it (the first time)
//     act(token, id)          an item was chosen
//     set(token, key, value)  a slider was released (key: resolution | refresh | dpi | bitrate): the menu stays; resolves when the host has set it, rejects with the host's words when it could not
//     layout(token, { tiles, sliders })  the person edited the quick panel (edit mode): the host stores it and sends the state back with it (null = the default)
//     cancel(token)           the menu ended without a choice (Escape, a press outside)
//     loaded()
//   }
// The optional start-page slot receives the current { state, token } for the popup. The menu owns sizing, focus and dismissal.
const props = defineProps({ transport: { type: Object, required: true } })

const gen = ref(0)
const variant = ref('v1')
const state = ref({})
const edge = ref('bottom')
const maxHeight = ref(0)
const shown = ref(false)
const model = computed(() => VARIANTS[variant.value] || VARIANTS.v1)
let token = 0
let open = false
let reported = null // the size the host's window has
const stops = []

function report(size) {
  reported = size
  return Promise.resolve(props.transport.ready(token, { w: size.w, h: size.h })).catch(() => {})
}
// The menu says when its card needs another window size: before a card grows (it waits for the answer, so the window is resized once, before anything moves) and after one has shrunk. Never per frame.
function onSize(size) {
  if (!open || !size.w || !size.h) return Promise.resolve()
  if (reported && reported.w === size.w && reported.h === size.h) return Promise.resolve()
  return report(size)
}
async function onOpen(msg) {
  if (open) open = false
  applyThemeSnapshot(document, msg.theme)
  token = msg.token
  variant.value = VARIANTS[msg.variant] ? msg.variant : 'v1'
  state.value = msg.state || {}
  edge.value = msg.edge || 'bottom'
  maxHeight.value = Number(msg.maxHeight) || 0
  reported = null
  open = true
  shown.value = false
  gen.value++ // a menu of its own for every open: nothing of the last (page, focus) is left
  await nextTick()
  shown.value = true
}
function onClose() { open = false; shown.value = false }
function onState(msg) { if (open && msg.token === token) state.value = msg.state || {} }

const layout = (record) => { if (open) props.transport.layout?.(token, record) }
const apply = (key, value) => (props.transport.set ? props.transport.set(token, key, value) : Promise.resolve())

function answer(id) {
  if (!open) return
  open = false
  if (id) props.transport.act(token, id)
  else props.transport.cancel(token)
}
const onPress = (event) => { if (open && !event.target?.closest?.('.m-tray')) answer(null) }
const onKey = (event) => { if (open && event.key === 'Escape') { event.preventDefault(); answer(null) } }
const onContext = (event) => event.preventDefault()

onMounted(async () => {
  installModality(document)
  stops.push(await props.transport.onOpen(onOpen), await props.transport.onClose(onClose), await props.transport.onState?.(onState))
  document.addEventListener('pointerdown', onPress, true)
  document.addEventListener('keydown', onKey)
  document.addEventListener('contextmenu', onContext, true)
  props.transport.loaded?.()
})
onBeforeUnmount(() => {
  for (const stop of stops.splice(0)) { try { stop?.() } catch { /* already gone */ } }
  document.removeEventListener('pointerdown', onPress, true)
  document.removeEventListener('keydown', onKey)
  document.removeEventListener('contextmenu', onContext, true)
})
defineExpose({ onOpen, onClose, onSize, state })
</script>

<template>
  <div v-if="shown" class="m-tray-host" :class="{ 'is-top': edge === 'top' }" :style="{ inset: `${POPUP_PAD}px` }">
    <MTrayMenu :key="gen" :model="model" :state="state" :edge="edge" :max-height="maxHeight" focus :apply="apply" :sized="onSize" @layout="layout" @act="answer" @close="answer(null)">
      <template v-if="$slots['start-page']" #start-page>
        <slot name="start-page" :state="state" :token="token" />
      </template>
    </MTrayMenu>
  </div>
</template>

<style>
html:not([data-modality='keyboard']) .m-tray button:focus-visible { outline-color: transparent; }
</style>
