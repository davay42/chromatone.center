<script setup>
//https://danigb.github.io/smplr/
//https://github.com/danigb/smplr

import { useSoundFont } from '#/use';

const { getSoundfontNames, inst, cached, loaded, instrument, synthEnabled, active, volume } = useSoundFont()

const capitalize = s => s && s[0].toUpperCase() + s.slice(1)
</script>

<template lang='pug'>
.flex.flex-col.bg-light-900.dark-bg-dark-100.rounded-lg.bg-opacity-60.backdrop-blur-md.dark-bg-opacity-60
  //- .flex.flex-wrap.gap-1.p-2.overflow-scroll.max-w-full.max-h-full
    button.transition.p-1.text-sm.cursor-pointer.border-1.border-dark-900.border-opacity-10.rounded.shadow.flex-auto.capitalize.dark-border-light-200.dark-border-opacity-20(
      v-for="name in getSoundfontNames()" :key="name"
      @click="instrument = name"
      :class="{ ['border-opacity-100 dark-border-opacity-90 filter filter-invert']: name == instrument, ['opacity-50']: name == instrument && !loaded, ['bg-light-100 dark-bg-dark-900']: !cached[name], ['bg-light-900 dark-bg-dark-100']: cached?.[name] }"
      ) {{ name.replaceAll('_', ' ') }}

  .flex.flex.gap-4.items-center.justify-start.p-2
    button.flex.text-lg.transition.text-button.items-center.justify-center(@click="synthEnabled = !synthEnabled"
      :class="{ 'active': synthEnabled }"
      )
      .i-la-spinner.animate-pulse(v-if="!loaded")
      .i-la-power-off(v-else)
      .p-2px.rounded-full.bg-current.absolute(:style="{ opacity: active ? 1 : 0 }")

    select.flex-1.min-w-8.p-1.bg-light-500.dark-bg-dark-500(v-model="instrument")
      option( v-for="name in getSoundfontNames()" :key="name" :value="name") {{ capitalize(name.replaceAll('_', ' ')) }} {{ cached[name] ? '✔' : '' }}

    ControlRotary.scale-80.-m-4.-ml-2(v-model="volume" :max="1" :step="0.01" param="VOL")


</template>