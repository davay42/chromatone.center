<script setup>
import { notes } from '#/use/theory'
import { defaultScheme, scheme, noteColor } from '#/use/colors'
import { getCircleCoord, rotateArray } from '#/use/calculations'
import { activeNotes, guessChords, activeChromaMidi } from '#/use/midi'
import { playNote, stopNote } from '#/use/midi'
import { globalScale } from '#/use/global'
import { useTuner } from '#/use/tuner'
import { colord } from "colord";
import { useClipboard, watchThrottled } from '@vueuse/core'
import { computed, onMounted, ref, shallowRef } from 'vue'
import { Note } from 'tonal'
import { useGesture } from '@vueuse/gesture'

const props = defineProps({
  letters: { type: Boolean, default: true },
  size: { type: Number, default: 1000 },
  pad: { type: Number, default: 50 },
})

const flower = computed(() => {
  return notes.map((note, n) => {
    return {
      note: note,
      pitch: n,
      midi: n + 57 + (n < globalScale.tonic ? 12 : 0),
      coord: coord(n, 0.42),
      middle: coord(n, 0.28),
      inside: coord(n, 0.15),
    }
  })
})


const tunr = shallowRef()
onMounted(() => {
  tunr.value = useTuner()
})

function getAmmount(ammount) {
  return ammount > tunr.value.tuner.chromaAvg ? tunr.value.tuner.note.silent ? 0 : ammount : 0
}

function coord(n = 0, q = 0.5) {
  return getCircleCoord(n % 12, 12, props.size * q, 0)
}

const pressed = ref()

function keyPlay(midiNote, event, off, velocity) {
  if (!off) {
    playNote(midiNote, velocity)
  } else {
    stopNote(midiNote)
  }
}

const { copy, copied } = useClipboard()

const loaded = ref('')
const err = ref(null)

watchThrottled(loaded, l => {
  try {
    let arr = JSON.parse(l)
    if (arr.length == 12 && arr.every(c => colord(c).isValid())) {
      scheme.custom = arr
      loaded.value = ''
      scheme.customize = false
    } else {
      throw ('Invalid scheme')
    }
  } catch (error) {
    err.value = true
    setTimeout(() => err.value = null, 1000)
    console.error('Loaded color scheme is not valid')
  }
})

const svg = ref()
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

    const keyElement = element.closest('.key')
    if (!keyElement) return

    const noteData = flower.value[parseInt(keyElement.dataset.pitch)]
    if (!noteData) return

    if (first) {
      playNote(noteData.midi)
      currentNote.value = noteData.midi
    } else if (last) {
      stopNote(currentNote.value)
      currentNote.value = null
    } else if (active && currentNote.value !== noteData.midi) {
      stopNote(currentNote.value)
      playNote(noteData.midi)
      currentNote.value = noteData.midi
    }
  },
  onDragEnd: () => {
    if (currentNote.value !== null) {
      stopNote(currentNote.value)
      currentNote.value = null
    }
  }
}, {
  domTarget: svg,
  eventOptions: { passive: false },
  triggerAllEvents: true,
  dragDelay: 0,
})

function handleTouchStart({ event }) {
  event.preventDefault()
  Array.from(event.changedTouches).forEach(touch => {
    const element = document.elementFromPoint(touch.clientX, touch.clientY)
    if (!element) return

    const keyElement = element.closest('.key')
    if (!keyElement) return

    const noteData = flower.value[parseInt(keyElement.dataset.pitch)]
    if (!noteData) return

    touchPoints.set(touch.identifier, noteData.midi)
    playNote(noteData.midi)
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

    const keyElement = element.closest('.key')
    if (!keyElement) {
      if (oldNote !== undefined) {
        stopNote(oldNote)
        touchPoints.delete(touch.identifier)
      }
      return
    }

    const noteData = flower.value[parseInt(keyElement.dataset.pitch)]
    if (!noteData) return

    const newNote = noteData.midi
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

<template lang="pug">
.max-h-80vh.relative.flex.items-center.flex-col.justify-center

  button.absolute.opacity-50.hover-opacity-100.transition.cursor-pointer.bottom-0.left-5.flex.items-center.gap-1.bg-light-100.dark-bg-dark-100.rounded-xl.p-2.shadow-lg(
    v-tooltip.bottom="'Copy custom schema'"
    aria-label="Copy custom schema"
    @click="copy(JSON.stringify(scheme.custom))"
    :class="{ customize: copied }"
    v-if="scheme.customize"
    )
    .i-la-copy.text-xl
    .p-0 COPY

  .absolute.opacity-50.hover-opacity-100.transition.bottom-0.right-5.flex.items-center.gap-1.bg-light-100.dark-bg-dark-100.rounded-xl.shadow-lg(
    v-tooltip.bottom="'Paste custom schema'"
    aria-label="Paste custom schema"
    v-if="scheme.customize"
    )
    .i-la-paste.text-xl.absolute.ml-1
    input.w-25.p-2.pl-8.rounded-xl(
      v-model="loaded" placeholder="PASTE"
      :style="{ backgroundColor: err ? 'red' : '' }"
      )

  svg.w-full.min-w-full(
    ref="svg"
    version="1.1",
    baseProfile="full",
    :viewBox="`${-pad} ${-pad} ${size + 2 * pad} ${size + 2 * pad}`",
    xmlns="http://www.w3.org/2000/svg",
    text-anchor="middle" 
    dominant-baseline="middle"
    style="touch-action: none"
    @touchstart="pressed = true"
    @touchend="pressed = false"
    @mousedown="pressed = true"
    @mouseup="pressed = false"
    @mouseleave="pressed = false"
    )

    defs
      filter#shadow
        feDropShadow(
          dx="0"
          dy="0"
          stdDeviation="8"
          flood-opacity="0.3"
        )
      filter#blur(
        x="-1" 
        y="-1" 
        width="300" 
        height="300")
        feGaussianBlur(i
        n="SourceGraphic" 
        :stdDeviation="20")
      filter#blur-less(
        x="-1" 
        y="-1" 
        width="300" 
        height="300")
        feGaussianBlur(i
        n="SourceGraphic" 
        :stdDeviation="15")

    g(:transform="`translate(${size / 2}, ${size / 2}) `")
      g.keys(v-for="(note, pitch) in flower" :key="note")
        g.key.cursor-pointer(
          :data-pitch="pitch"
          )
          g.petal(
            :transform="`rotate(${pitch * 30}) translate(2,-120) `"
            :fill="note.note.length > 1 ? '#222' : '#eee'"
            :opacity="activeChromaMidi[pitch] ? 1 : 0.9"
            style="transition: all 100ms ease-out"
            filter="url(#shadow)"
            ) 
            path(
              :d="`M 0,0 a 30,30,0,0,0,25,-20 l 70 -260 a 120,120,0,0,0,2,-20 a 100,100,0,0,0,-200,0 a 120,120,0,0,0,2,20 l 70,260 a 30,30,0,0,0,25,20z`"
              )
          g()
            g.note.select-none(
              stroke-width="4"
              stroke-linecap="round"
              v-if="scheme.custom[pitch] != defaultScheme[pitch]"
              :transform="`translate(${note.inside.x}, ${note.inside.y})`"
              @click="scheme.custom[pitch] = defaultScheme[pitch]"
              )
              circle.transition(:fill="defaultScheme[pitch]" r="20" :opacity="scheme.customize ? 1 : 0.1" )
              g(
                stroke-width="4"
                stroke-linecap="round"
                )
                line(stroke="black" x1="-10" y1="-10" x2="10" y2="10")
                line(stroke="black" x1="10" y1="-10" x2="-10" y2="10")
            g.note.select-none(
              v-if="scheme.customize"
              :transform="`translate(${note.middle.x - 36}, ${note.middle.y - 36})`"
              )
              foreignObject(x="10" y="10" width="100" height="100")
                input.h-30.w-30.rounded-xl(type="color" v-model="scheme.custom[pitch]")
          g.note.select-none(
            :transform="`translate(${note.coord.x}, ${note.coord.y})`"
            )
            circle.transition(
              style="transition: all 100ms ease-out"
              :fill="noteColor(pitch, activeChromaMidi[pitch] ? 4 : 3)"
              :r="pitch == globalScale.tonic ? size / 11 : size / 12"
              )
            g(v-if="tunr?.tuner?.initiated")
              circle(
                style="transition: all 80ms ease-out"
                :fill="noteColor(pitch, 7, 2, getAmmount(tunr.tuner.aChroma[pitch]))"
                :r="size / 12"
                filter="url(#blur)"
                )
            text.transition(
              v-if="letters"
              :font-size="size / 20"
              font-weight="bold"
              :fill="!activeChromaMidi[pitch] ? 'white' : 'black'"
              )
              tspan(
                dy="5"
                text-anchor="middle" 
                dominant-baseline="middle" 
                ) {{ note.note }}

      g.spiral.pointer-events-none
        g.interval(v-for="(bool, note) in activeNotes" :key="note")
          transition-group(name="fade")
            line(
              v-for="(bool2, note2) in activeNotes" :key="note2"
              :x1="coord((note - 9) % 12, note / 700 + 0.145).x" 
              :y1="coord((note - 9) % 12, note / 700 + 0.145).y" 
              :x2="coord((note2 - 9) % 12, note2 / 700 + 0.145).x" 
              :y2="coord((note2 - 9) % 12, note2 / 700 + 0.145).y" 
              :stroke="colord(noteColor(note - 9, 2)).mix(noteColor(note2 - 9, 2)).toHex()"
              stroke-width="20"
              stroke-linecap="round"
              :style="{ filter: `drop-shadow(0px 0px 4px ${colord(noteColor(note - 9, 3)).mix(noteColor(note2 - 9, 3)).alpha(0.5).toHex()}` }"
              )
        transition-group(name="fade")
          g.note(v-for="(bool, note) in activeNotes" :key="note")
            circle(
              style="transition: all 100ms ease-out"
              :cx="coord((note - 9) % 12, note / 700 + 0.145).x" 
              :cy="coord((note - 9) % 12, note / 700 + 0.145).y" 
              :fill="scheme.custom[(note - 9) % 12]"
              :r="12" 
              )          
      g.controls
        g.mic.transition.cursor-pointer.opacity-40.hover-opacity-100(
          v-if="tunr?.tuner && !tunr.tuner.initiated"
          v-tooltip.top="'Start input audio analysis'"
          aria-label="Start input audio analysis"
          @click="tunr.init()"
          )
          circle.transition(
            r='32' 
            cy="-68"
            fill="#3331")
          i-la-microphone(
            font-size="42"
            x="-25"
            y="-90")
        g.tuner.transition(
          v-if="tunr?.tuner?.initiated"
          aria-label="Calculated fundamental frequency"
          )
          circle.transition(
            filter="url(#blur)"
            style="transition:all 500ms ease"
            r='32' 
            cy="0"
            :fill="!tunr.tuner.note?.silent && tunr.tuner.initiated ? noteColor((tunr.tuner.note.value + 3) % 12, 4) : 'transparent'")
          text.opacity-50.select-none(
            v-if="tunr?.tuner?.initiated"
            y="-67"
            v-show="!tunr.tuner.note?.silent"
            font-size="28"
            fill="currentColor"
            ) {{ tunr.tuner.note.name }}
        g.customize.opacity-20.hover-opacity-100.transition.cursor-pointer(
          @click="scheme.customize = !scheme.customize"
          v-tooltip.top="'Customize colors'"
          aria-label="Customize colors sitewide"
          )
          circle(r='32' cy="70" fill="#3333")
          i-la-cog(
            font-size="42"
            x="-25"
            y="44"
            :class="{ customize: scheme.customize }"
            )
      g.chord.cursor-pointer(
        v-tooltip.top="guessChords.length > 1 && 'or ' + guessChords.slice(1).join(', ') || copied ? 'Copied!' : 'Click to copy'"
        :aria-label="'Guessed chord: ' + guessChords[0]"
        @click="copy(guessChords[0])"
        v-if="guessChords[0]"
        )
        rect(
          fill="#0001"
          x="-100" 
          y="-32" 
          width="200" 
          height="70"
          rx="10"
          )
        text(
          y="6"
          font-size="40"
          fill="currentColor"
        ) {{ guessChords[0] }}
      g.center(v-else)
        circle(
          r="3" 
          fill="currentColor"
          )
</template>

<style scoped lang="postcss">
input[type="color"] {
  width: 3rem;
  height: 3rem;
  padding: .5rem;
  background-color: transparent;
  border: 0;
  border-radius: 100%;
}

input[type="color"]::-webkit-color-swatch {
  border: none;
  border-radius: 50%;
}

svg {
  touch-action: none;
}

.key {
  touch-action: none;
}
</style>