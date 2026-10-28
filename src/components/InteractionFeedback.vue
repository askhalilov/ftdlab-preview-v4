<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { interactionConfig as config } from '../motion/interactionConfig'
import { createPortfolioCursorMotion } from '../motion/portfolioCursorMotion'

const props = withDefaults(defineProps<{ paused?: boolean }>(), { paused: false })
const surface = ref<HTMLElement>()
const dot = ref<HTMLElement>()
const ring = ref<HTMLElement>()

const interactiveSelector = 'a[href],button,[role="button"],[role="link"],summary,[tabindex]:not([tabindex="-1"]),[data-cursor-interactive]'
const nativeSelector = 'input,textarea,select,option,[contenteditable]:not([contenteditable="false"]),[data-native-cursor],iframe,video[controls],audio[controls]'
const navigationKeys = new Set(['Tab', 'Escape', 'Enter', ' ', 'ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight', 'Home', 'End', 'PageUp', 'PageDown'])

let capability: MediaQueryList | undefined
let motion: MediaQueryList | undefined
let observer: MutationObserver | undefined
let mounted = false
let visible = false
let lastHideReason = 'initial'
let loaderVisible = false
let mouseDown = false
let selecting = false
let keyboardActive = false
let pointer = 'none'
let cursorMotion: ReturnType<typeof createPortfolioCursorMotion> | undefined
let touch: { element: HTMLElement; id: number; x: number; y: number; started: number; original: string | null } | undefined
let touchTimer: ReturnType<typeof setTimeout> | undefined

function attribute(name: string, value: string) {
  surface.value?.setAttribute(`data-cursor-${name}`, value)
}

function hide(reason: string) {
  visible = false
  lastHideReason = reason
  document.documentElement.classList.remove('ftd-cursor-active')
  if (surface.value) surface.value.hidden = true
  attribute('state', 'hidden')
  attribute('reason', reason)
  cursorMotion?.stop()
}

function clearTouch() {
  if (touchTimer) clearTimeout(touchTimer)
  touchTimer = undefined
  if (touch) {
    if (touch.original === null) touch.element.removeAttribute('data-ftd-touch')
    else touch.element.setAttribute('data-ftd-touch', touch.original)
  }
  touch = undefined
}

function hasOpenModal() {
  // A body overlay cannot paint above a native top-layer dialog/popover.
  if (document.querySelector('dialog[open]')) return true
  try { return Boolean(document.querySelector('[popover]:popover-open')) } catch { return false }
}

function unavailableReason() {
  if (loaderVisible) return 'loader'
  if (props.paused) return 'paused'
  if (motion?.matches) return 'reduced-motion'
  if (!capability?.matches) return 'pointer-capability'
  if (document.hidden) return 'hidden'
  if (document.fullscreenElement) return 'fullscreen'
  if (hasOpenModal()) return 'dialog'
  return ''
}

function elementTarget(target: EventTarget | null) {
  return target instanceof Element ? target : undefined
}

function disabled(element: Element | undefined) {
  return Boolean(element?.closest(':disabled,[aria-disabled="true"],[inert]'))
}

function interactive(target: Element | undefined) {
  const control = target?.closest<HTMLElement>(interactiveSelector)
  return control && !disabled(control) && !control.closest(nativeSelector) ? control : undefined
}

function needsNative(target: Element | undefined) {
  if (!target || disabled(target) || target.closest(nativeSelector)) return true
  if (interactive(target)) return false
  const cursor = getComputedStyle(target).cursor
  return cursor === 'text' || cursor === 'vertical-text' || cursor === 'grab' || cursor === 'grabbing'
    || cursor.endsWith('-resize') || cursor === 'wait' || cursor === 'progress' || cursor === 'crosshair'
}

function show(event: PointerEvent) {
  const reason = unavailableReason()
  if (reason) { hide(reason); return }
  if (event.clientX < 0 || event.clientY < 0 || event.clientX >= innerWidth || event.clientY >= innerHeight) { hide('viewport'); return }
  if (selecting) { hide('selection'); return }
  const target = elementTarget(event.target)
  if (needsNative(target)) { hide('native'); return }
  // The portfolio only changes scale over buttons/anchors, including descendants.
  const hovering = Boolean(target?.closest('button,a'))
  // Match original mount/re-entry. Additional accessibility/native-UI gates
  // resume at the pointer to avoid a flight across selected or editable text.
  const initial = visible ? undefined : ['initial', 'viewport', 'loader'].includes(lastHideReason) ? 'origin' : 'snap'
  cursorMotion?.update(event.clientX, event.clientY, hovering, initial)
  visible = true
  if (surface.value) surface.value.hidden = false
  document.documentElement.classList.add('ftd-cursor-active')
  attribute('state', hovering ? 'hover' : 'default')
  attribute('reason', '')
}

function pointerKind(event: PointerEvent) {
  pointer = event.pointerType || 'mouse'
  attribute('pointer', pointer)
  if (pointer !== 'mouse') hide(pointer === 'touch' ? 'touch' : 'pen')
  return pointer
}

function onMove(event: PointerEvent) {
  if (pointerKind(event) !== 'mouse') {
    if (touch && touch.id === event.pointerId && Math.hypot(event.clientX - touch.x, event.clientY - touch.y) > config.touchMoveTolerance) clearTouch()
    return
  }
  keyboardActive = false
  if (!event.buttons) { mouseDown = false; selecting = false }
  show(event)
}

function onOver(event: PointerEvent) {
  // Source mouseover updates hover only; visibility begins on actual movement.
  if (pointerKind(event) === 'mouse' && visible && !selecting && !keyboardActive) show(event)
}

function onDown(event: PointerEvent) {
  if (pointerKind(event) === 'mouse') {
    keyboardActive = false
    mouseDown = event.button === 0
    // Original portfolio has no pointer-down scaling or added click effect.
    if (visible) show(event)
    return
  }
  clearTouch()
  if (!event.isPrimary || loaderVisible || props.paused || document.hidden) return
  const element = interactive(elementTarget(event.target))
  if (!element) return
  touch = { element, id: event.pointerId, x: event.clientX, y: event.clientY, started: performance.now(), original: element.getAttribute('data-ftd-touch') }
  element.setAttribute('data-ftd-touch', 'pressed')
  touchTimer = setTimeout(clearTouch, config.touchSafetyMs)
}

function onUp(event: PointerEvent) {
  if (pointerKind(event) === 'mouse') {
    const wasSelecting = selecting
    mouseDown = false
    selecting = false
    if (!wasSelecting && visible) show(event)
    return
  }
  if (touch?.id !== event.pointerId) return
  if (touchTimer) clearTimeout(touchTimer)
  touchTimer = setTimeout(clearTouch, Math.max(0, config.touchMinimumMs - (performance.now() - touch.started)))
}

function onCancel() { mouseDown = false; selecting = false; clearTouch(); hide('cancelled') }
function onExit(event: PointerEvent) { if (!event.relatedTarget) { mouseDown = false; selecting = false; hide('viewport') } }
function onBlur() { mouseDown = false; selecting = false; clearTouch(); hide('blur') }
function onVisibility() { if (document.hidden) { clearTouch(); hide('hidden') } }
function onResize() { clearTouch(); hide('resize') }
function onFullscreen() { clearTouch(); hide('fullscreen') }
function onCapabilityChange() { clearTouch(); hide(unavailableReason() || 'pointer-change') }
function onKeyboard(event: KeyboardEvent) { if (navigationKeys.has(event.key) || event.key.length === 1) { keyboardActive = true; clearTouch(); hide('keyboard') } }
function onContextMenu() { mouseDown = false; selecting = false; clearTouch(); hide('native') }
function onSelectStart() { if (mouseDown) { selecting = true; hide('selection') } }
function onFocus(event: FocusEvent) { if (needsNative(elementTarget(event.target))) hide('native') }
function onTopLayer(event: Event) {
  const target = elementTarget(event.target)
  if (target?.matches('dialog,[popover]')) hide('dialog')
}
function observeLoader() {
  loaderVisible = Boolean(document.querySelector('.ftd-prehero-loader'))
  if (loaderVisible) { clearTouch(); hide('loader') }
}

watch(() => props.paused, (paused) => { if (mounted && paused) { clearTouch(); hide('paused') } })

onMounted(() => {
  mounted = true
  if (dot.value && ring.value) {
    cursorMotion = createPortfolioCursorMotion(dot.value, ring.value, active => attribute('loop', active ? 'running' : 'idle'))
  }
  capability = matchMedia('(any-pointer: fine) and (any-hover: hover)')
  motion = matchMedia('(prefers-reduced-motion: reduce)')
  capability.addEventListener('change', onCapabilityChange)
  motion.addEventListener('change', onCapabilityChange)
  observeLoader()
  // The loader teleports directly to body. Avoid observing mascot/style changes.
  observer = new MutationObserver(observeLoader)
  observer.observe(document.body, { childList: true })
  document.addEventListener('pointermove', onMove, { passive: true })
  document.addEventListener('pointerover', onOver, { passive: true })
  document.addEventListener('pointerdown', onDown, { passive: true })
  document.addEventListener('pointerup', onUp, { passive: true })
  document.addEventListener('pointercancel', onCancel, { passive: true })
  document.addEventListener('pointerout', onExit, { passive: true })
  document.addEventListener('keydown', onKeyboard)
  document.addEventListener('contextmenu', onContextMenu)
  document.addEventListener('selectstart', onSelectStart)
  document.addEventListener('focusin', onFocus)
  document.addEventListener('beforetoggle', onTopLayer, true)
  document.addEventListener('toggle', onTopLayer, true)
  document.addEventListener('visibilitychange', onVisibility)
  document.addEventListener('fullscreenchange', onFullscreen)
  window.addEventListener('blur', onBlur)
  window.addEventListener('resize', onResize, { passive: true })
  window.addEventListener('scroll', clearTouch, { passive: true })
})

onBeforeUnmount(() => {
  mounted = false
  clearTouch()
  hide('unmounted')
  cursorMotion?.destroy()
  cursorMotion = undefined
  observer?.disconnect()
  capability?.removeEventListener('change', onCapabilityChange)
  motion?.removeEventListener('change', onCapabilityChange)
  document.removeEventListener('pointermove', onMove)
  document.removeEventListener('pointerover', onOver)
  document.removeEventListener('pointerdown', onDown)
  document.removeEventListener('pointerup', onUp)
  document.removeEventListener('pointercancel', onCancel)
  document.removeEventListener('pointerout', onExit)
  document.removeEventListener('keydown', onKeyboard)
  document.removeEventListener('contextmenu', onContextMenu)
  document.removeEventListener('selectstart', onSelectStart)
  document.removeEventListener('focusin', onFocus)
  document.removeEventListener('beforetoggle', onTopLayer, true)
  document.removeEventListener('toggle', onTopLayer, true)
  document.removeEventListener('visibilitychange', onVisibility)
  document.removeEventListener('fullscreenchange', onFullscreen)
  window.removeEventListener('blur', onBlur)
  window.removeEventListener('resize', onResize)
  window.removeEventListener('scroll', clearTouch)
})
</script>

<template>
  <Teleport to="body">
    <div ref="surface" class="ftd-interaction-feedback" :style="{ '--ftd-cursor-dot-size': `${config.dotDiameter}px`, '--ftd-cursor-ring-size': `${config.ringDiameter}px` }" hidden aria-hidden="true" data-cursor-state="hidden" data-cursor-loop="idle" data-cursor-pointer="none" data-cursor-reason="initial">
      <span ref="dot" class="ftd-cursor-dot" />
      <span ref="ring" class="ftd-cursor-ring" />
    </div>
  </Teleport>
</template>

<style>
/* This component changes only its own overlay and a temporary pressed target. */
html.ftd-cursor-active, html.ftd-cursor-active * { cursor: none !important; }
html.ftd-cursor-active :is(input,textarea,select,option,[contenteditable]:not([contenteditable="false"]),[data-native-cursor],iframe,video[controls],audio[controls]),
html.ftd-cursor-active :is([contenteditable]:not([contenteditable="false"]),[data-native-cursor]) * { cursor: auto !important; }
.ftd-interaction-feedback { display: contents; pointer-events: none; user-select: none; }
.ftd-interaction-feedback[hidden] { display: none !important; }
.ftd-interaction-feedback * { position: fixed; left: 0; top: 0; z-index: 9999; pointer-events: none; border-radius: 50%; box-sizing: border-box; mix-blend-mode: difference; }
.ftd-cursor-dot { width: var(--ftd-cursor-dot-size); height: var(--ftd-cursor-dot-size); background: #fff; }
.ftd-cursor-ring { width: var(--ftd-cursor-ring-size); height: var(--ftd-cursor-ring-size); border: 1px solid rgb(255 255 255 / 98%); }
[data-ftd-touch="pressed"] { filter: brightness(1.1); }
@media (prefers-reduced-motion: reduce) {
  .ftd-interaction-feedback { display: none !important; }
}
</style>
