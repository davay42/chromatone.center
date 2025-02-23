<script setup>
import { noteColor } from '#/use/colors';
import { useMidi } from '#/use/midi';
import { notes } from '#/use/theory';
import { computed, ref, watch } from 'vue';
import { useRange } from './useRange.js'
import { useGesture } from '@vueuse/gesture';
import { globalScale } from '#/use/global';
import { useClamp } from '@vueuse/math'
import { useStorage } from '@vueuse/core'
import { intervals } from '#/use/theory';
import { ScaleType } from 'tonal'
import { playNote, stopNote } from '#/use/midi'

const props = defineProps({
  width: { type: Number, default: 900 },
  height: { type: Number, default: 400 },
  controlOffset: { type: Number, default: 100 },
  slotOffset: { type: Number, default: 0 }
})

const { activeNotes } = useMidi()

const { roundBegin: begin, roundEnd: end, beginControl, endControl, keys: rawKeys } = useRange()

const svg = ref()
const area = ref()

const pressed = ref(false)

useGesture({
  onPointerdown(ev) { pressed.value = true },
  onPointerleave(ev) { pressed.value = false },
  onPointercancel(ev) { pressed.value = false },
  onPointerup(ev) { pressed.value = false }
}, {
  domTarget: svg,
})

const tonicControl = ref()
const tonicCents = ref(globalScale.tonic * 100)

watch(tonicCents, cents => {
  globalScale.tonic = Math.round(((cents / 100) % 12 + 12) % 12)
})

useGesture({
  onDrag(ev) {
    ev.event.preventDefault()
    tonicCents.value += ev.delta[0]
  },
  onWheel(ev) {
    ev.event.preventDefault()
    tonicCents.value -= ev.velocities[0]
  }
}, {
  domTarget: tonicControl,
  eventOptions: { passive: false }
})

const scaleControl = ref()
const scaleCurrent = useClamp(useStorage('midikeys-scale', 1), 1, ScaleType.all().length - 1)

useGesture({
  onDrag(ev) {
    ev.event.preventDefault()
    scaleCurrent.value += ev.delta[0] / 5
  },
  onWheel(ev) {
    ev.event.preventDefault()
    scaleCurrent.value -= ev.velocities[0] / 5
  }
}, {
  domTarget: scaleControl,
  eventOptions: { passive: false }
})

const scaleChroma = computed(() => {
  let num = Math.round(scaleCurrent.value)
  return ScaleType.all()[num]
})

watch(scaleChroma, scale => {
  globalScale.chroma = scale.chroma
})

const filterScale = useStorage('filter-keys', true)

const keys = computed(() => filterScale.value ? rawKeys.value.filter(key => {
  return globalScale.isIn(notes[(key + 3) % 12])
}) : rawKeys.value)

const currentNote = ref(null)
const touchPoints = new Map()

useGesture({
  onTouchstart: handleTouchStart,
  onTouchmove: handleTouchMove,
  onTouchend: handleTouchEnd,
  onTouchcancel: handleTouchEnd,
  onDrag: ({ event, first, last, active }) => {
    event.preventDefault()
    const element = document.elementFromPoint(event.clientX, event.clientY)
    if (!element) return

    const noteElement = element.closest('.note-cell')
    if (!noteElement) return

    const note = parseInt(noteElement.dataset.note)
    if (isNaN(note)) return

    if (first) {
      playNote(note)
      currentNote.value = note
    } else if (last) {
      stopNote(currentNote.value)
      currentNote.value = null
    } else if (active && currentNote.value !== note) {
      stopNote(currentNote.value)
      playNote(note)
      currentNote.value = note
    }
  },
  onDragEnd: () => {
    if (currentNote.value !== null) {
      stopNote(currentNote.value)
      currentNote.value = null
    }
  }
}, {
  domTarget: area,
  eventOptions: { passive: false },
  triggerAllEvents: true,
  dragDelay: 0,
})

function handleTouchStart({ event }) {
  event.preventDefault()
  Array.from(event.changedTouches).forEach(touch => {
    const element = document.elementFromPoint(touch.clientX, touch.clientY)
    if (!element) return

    const noteElement = element.closest('.note-cell')
    if (!noteElement) return

    const note = parseInt(noteElement.dataset.note)
    if (isNaN(note)) return

    touchPoints.set(touch.identifier, note)
    playNote(note)
  })
}

function handleTouchMove({ event }) {
  event.preventDefault()
  Array.from(event.changedTouches).forEach(touch => {
    const oldNote = touchPoints.get(touch.identifier)
    const element = document.elementFromPoint(touch.clientX, touch.clientY)
    if (!element) {
      if (oldNote !== undefined) {
        stopNote(oldNote)
        touchPoints.delete(touch.identifier)
      }
      return
    }

    const noteElement = element.closest('.note-cell')
    if (!noteElement) {
      if (oldNote !== undefined) {
        stopNote(oldNote)
        touchPoints.delete(touch.identifier)
      }
      return
    }

    const newNote = parseInt(noteElement.dataset.note)
    if (newNote !== oldNote) {
      if (oldNote !== undefined) {
        stopNote(oldNote)
      }
      touchPoints.set(touch.identifier, newNote)
      playNote(newNote)
    }
  })
}

function handleTouchEnd({ event }) {
  event.preventDefault()
  Array.from(event.changedTouches).forEach(touch => {
    const note = touchPoints.get(touch.identifier)
    if (note !== undefined) {
      stopNote(note)
      touchPoints.delete(touch.identifier)
    }
  })
}

</script>

<template lang='pug'>
svg.w-full.cursor-pointer.fullscreen-container.overflow-hidden.select-none.touch-none(
  :viewBox="`0 -${controlOffset + slotOffset} ${width} ${height + controlOffset}`"
  version="1.1",
  baseProfile="full",
  xmlns="http://www.w3.org/2000/svg",
  style="touch-action:none"
  ref="svg"
  )
  g.slot(
    :transform="`translate(0,${-slotOffset - controlOffset})`"
    )
    slot
  g.offset(  
    :transform="`translate(0,${-controlOffset})`"
    )
    g.tonic(
      ref="tonicControl"
      )
      rect(
        :height="controlOffset"
        :width="width / 5"
        :fill="noteColor(globalScale.tonic, 2, .8)"
        )
      text.font-bold.text-5xl(
        :x="width / 10"
        text-anchor="middle"
        :y="controlOffset * .75"
        ) {{ notes[(globalScale.tonic) % 12] }}
      line(
        v-for="(note, n) in notes"
        :transform="`translate(${(n / 12 + 1 / 24) * width / 5},0)`"
        :y2="16"
        stroke-width="6"
        stroke-linecap="round"
        :stroke="noteColor(n, 5, .4)"
        )
      line.transition(
        :y2="20"
        stroke-width="8"
        stroke-linecap="round"
        :transform="`translate(${(((tonicCents / 1200) % 1 + 1) % 1) * width / 5 + width / 120},0)`"
        stroke="white"
        )
    g.scale(
      ref="scaleControl"
      :transform="`translate(${width / 5},0)`"
      )
      rect(
        :width="width * 2 / 5"
        :height="controlOffset"
        fill="#aaa"
        )
      text.text-4xl(
        :x="30"
        :y="controlOffset * .75"
        ) {{ scaleChroma.name }}
      line(
        v-for="(note, n) in ScaleType.all().length"
        :transform="`translate(${(n / ScaleType.all().length + 1 / (ScaleType.all().length)) * width * 2 / 5},0)`"
        :y2="36"
        :opacity="note == Math.round(scaleCurrent) ? 1 : .2"
        stroke-width="2"
        stroke-linecap="round"
        :stroke="`currentColor`"
        )
      line(
        :y2="20"
        stroke-width="8"
        stroke-linecap="round"
        :transform="`translate(${scaleCurrent / ScaleType.all().length * width * 2 / 5},0)`"
        stroke="white"
        )

      g.show(
        :transform="`translate(${width * 2 / 5 - 50},10)`"
        )
        circle(
          @click="filterScale = !filterScale"
          :cy="controlOffset * .5"
          r="20"
          :fill="filterScale ? 'black' : 'transparent'"
          :stroke="'black'"
          :stroke-width="4"
          )


    g.begin(
      ref="beginControl"
      :transform="`translate(${width * 3 / 5},0)`"
      )
      rect(
        :height="controlOffset"
        :width="width / 5"
        :fill="noteColor(begin + 3, null, activeNotes[begin] ? 1 : 0.1)"
        )
      text.font-bold.text-5xl.pointer-events-none(
        :x="10"
        :y="controlOffset * .75"
        ) {{ notes[(begin + 3) % 12] }}{{ Math.floor((Math.round(begin) + 3) / 12) - 1 }}

    g.end(
      ref="endControl"
      :transform="`translate(${width * 4 / 5},0)`"
      )
      rect(
        :height="controlOffset"
        :width="width / 5"
        :fill="noteColor(end + 3, null, activeNotes[end] ? 1 : 0.1)"
        )
      text.font-bold.text-5xl.pointer-events-none(
        :x="width / 5 - 20"
        text-anchor="end"
        :y="controlOffset * .75"
        ) {{ notes[(end + 3) % 12] }}{{ Math.floor((end + 3) / 12) - 1 }}


    g.range.pointer-events-none(
      :transform="`translate(${width * 3 / 5},0)`"
      )
      line(
        v-for="(note, n) in 127"
        :transform="`translate(${(n / 127) * width * 2 / 5},0)`"
        :y2="16"
        opacity=".3"
        stroke-width="2"
        stroke-linecap="round"
        :stroke="(n + 3) % 12 == globalScale.tonic ? noteColor((n + 3) % 12) : `white`"
        )
      rect.ranger(
        :x="(begin / 127) * width * 2 / 5"
        :width="((end - begin) / 127) * width * 2 / 5"
        height="20"
        fill="gray"
        :opacity=".8"
        )
      line.begin(
        :y2="20"
        stroke-width="8"
        stroke-linecap="round"
        :transform="`translate(${(begin / 127) * width * 2 / 5},0)`"
        stroke="white"
        )
      line.end(
        :y2="20"
        stroke-width="8"
        stroke-linecap="round"
        stroke="white"
        :transform="`translate(${(end / 127) * width * 2 / 5},0)`"
        )

  g.keys(
    ref="area"
    )
    MidiKeysNote(
      v-for="(key, k) in keys" :key="key"
      :note="key"
      :step="k"
      :width="width / keys.length"
      :height="height"
      :pressed="pressed"
      )
</template>

<style scoped>
svg {
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
  -o-user-select: none;
  user-zoom: none;
  -webkit-user-drag: none;
  -webkit-touch-callout: none;
  -webkit-user-callout: none;
  -webkit-user-select: none;
  -webkit-user-modify: none;
  -webkit-highlight: none;
}

.note-cell {
  touch-action: none;
}
</style>