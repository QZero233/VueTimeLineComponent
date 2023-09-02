<script setup lang="ts">
import TimeLine from "./components/TimeLine.vue"
import {ref} from "vue";


const clip=ref([
  {
    "startTime":5*1000,
    "endTime":20*1000
  },
  {
    "startTime":30*1000,
    "endTime":50*1000
  }
])

const timeChange=(newTime : number)=>{
  console.log(newTime)
}

const clipDurationChange=(evt)=>{
  console.log(evt)
}

const timeLine=ref<InstanceType<typeof TimeLine>>()

const clk = ()=>{
  timeLine.value.setTimeCursorTime(200*1000)
}

</script>

<template>
  <button @click="clk">233</button>

  <div style="position: relative; top: 500px; height: 180px">
    <TimeLine ref="timeLine" :clips="clip" :totalTime="400*1000" @timeCursorChange="timeChange" @clipDurationChange="clipDurationChange" />
  </div>
</template>

<style scoped>

</style>
