<script setup>
import { reactive, ref, watch, onMounted } from 'vue'
import { useTuner } from '#/use/tuner'
import { onKeyStroke, useWindowSize } from '@vueuse/core'
import { useClamp } from '@vueuse/math';
import { useStorage } from '@vueuse/core';
import { colord } from 'colord';
import { noteColor } from '#/use';

const { width, height } = useWindowSize()

const draw = reactive({
  running: true,
  speed: useClamp(useStorage('pitch-roll-speed', 4), 4, 20),
  runnerWidth: 2,
  runnerAhead: 4,
  maxPlotFrequency: 880,
  maxPlotNote: 97,
  minPlotNote: 24,
  prevBeat: 0,
  step: 0,
})

onKeyStroke(' ', (event) => {
  event.preventDefault()
  draw.running = !draw.running
})

onKeyStroke('Enter', () => {
  clear()
})

function dragSpeed(drag) {
  if (drag.tap) {
    draw.running = !draw.running
  }
  draw.speed = draw.speed + drag.delta[0] / 5
}

const octaves = [0, 1, 2, 3, 4, 5, 6, 7, 8]

const { init, tuner } = useTuner();



const roll = ref()
let ctx

onMounted(() => {
  ctx = roll.value.getContext('2d')
  // roll.value.width = window.innerWidth
  // roll.value.height = window.innerHeight
})

watch(() => tuner?.frame, frame => {
  if (!draw.running) return
  let x = (frame * draw.speed) % roll.value.width

  // let imageData = ctx.getImageData(0, 0, roll.value.width, roll.value.height);
  // ctx.clearRect(0, 0, roll.value.width, roll.value.height)
  // ctx.globalAlpha = 0.7
  // ctx.putImageData(imageData, 0, 0);

  ctx.globalAlpha = 1
  //runner

  ctx.clearRect(x, 0, draw.speed, roll.value.height)

  ctx.beginPath()
  ctx.fillStyle = noteColor(tuner.note.value - 9, 0)
  ctx.fillRect(x + draw.speed, 0, draw.runnerWidth, roll.value.height)

  //notes
  if (!tuner.note.silent) {
    const y = roll.value.height - (tuner.note.value - draw.minPlotNote) / (draw.maxPlotNote - draw.minPlotNote) * roll.value.height
    ctx.arc(x - 8, y, 8, 0, 4 * Math.PI)
    ctx.fillStyle = noteColor(tuner.note.value - 9, 0)
    ctx.fill()
  }

  if (frame % 50 == 0) {
    // green octave lines
    ctx.beginPath()
    ctx.strokeStyle = "hsla(90,50%,50%,0.1)"
    ctx.lineWidth = "1";
    for (let oct of octaves) {
      let y = roll.value.height - (oct * 12 - draw.minPlotNote) / (draw.maxPlotNote - draw.minPlotNote) * roll.value.height
      ctx.moveTo(0, y)
      ctx.lineTo(roll.value.width, y)
    }
    ctx.stroke()
  }

  ctx.globalAlpha = 0.25
  //beat
  if (tuner.beat > draw.prevBeat) {
    draw.prevBeat = tuner.beat
    ctx.fillStyle = tuner.note.color
    ctx.fillRect(x - 5, 0, 1, roll.value.height)
  }
  // if (frame % 5) {
  //   ctx.globalAlpha = 0.01
  //   ctx.fillStyle = "hsl(0,0%,100%)"
  //   ctx.fillRect(0, 0, roll.value.width, roll.value.height)
  // }

})

function start() {
  init();
  draw.running = true
}

function clear() {
  ctx.clearRect(0, 0, roll.value.width, roll.value.height)
}

</script>

<template lang="pug">
.flex.flex-col.mb-8.relative
  .fullscreen-container.cursor-pointer#screen
    control-start.absolute(
      v-if="!tuner.running", 
      @click="start()") Start rolling 
    .p-1.absolute.top-2.left-2.flex.items-center
      .i-la-angle-double-right
      span {{ draw.speed.toFixed(1) }}
    canvas.w-full.h-full(    
      ref="roll"
      v-drag="dragSpeed"
      :width
      :height
      @dblclick="clear()"
      )
  .flex.gap-2.w-full.p-2.items-center.absolute.bottom-4(v-if="tuner.note")
    .btn(@click="draw.running = !draw.running")
      .i-la-play(v-if="!draw.running")
      .i-la-pause(v-else)
    .btn(@click="clear()")
      .i-la-times
    .flex-1
    .flex-1.text-center.font-bold.text-4xl.transition-all.duration-200.flex.items-center(:style="{ color: tuner.note.color }") 
      .p-1.w-2em {{ tuner.note?.name }}
      .p-1.w-1em {{ tuner.note?.octave }} 
      .p-1.mt-2.w-6em.text-sm {{ tuner.note.cents > 0 ? '+' : '' }}{{ tuner.note.cents }} cents

    .flex-1
    .flex-0.text-center {{ tuner.bpm.toFixed(1) }} BPM

</template>

<style lang="postcss" scoped>
.button {
  @apply p-4 m-8 shadow-lg cursor-pointer transition-all duration-300 rounded-3xl text-4xl bg-light-700 text-center font-bold dark-(bg-dark-400);
}

.btn {
  @apply p-2 cursor-pointer border-1 rounded;
}
</style>