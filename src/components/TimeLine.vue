<script setup lang="ts">
import {computed, defineEmits, defineExpose, defineProps, onMounted, ref} from "vue"

const props=defineProps({
  totalTime:{
    type:Number,
    required:true
  },
  clips:{
    type:Array,
    required:true
  }
})

const emit=defineEmits(["timeCursorChange","clipDurationChange"])

//整个组件的高度
const componentHeight=ref(180)
//展示时间，放大缩小地方的高度
const headerHeight=ref(30)
//显示时间区域的高度
const timeAreaHeight=ref(32)
//操作区的高度
const operateAreaHeight=computed(()=> componentHeight.value - timeAreaHeight.value - headerHeight.value)

//缩放大小，即每个小单位多长时间，单位为毫秒
const scale=ref(1000)
const totalTime=ref(props.totalTime)

//最多可以选到多少个单位
const maxValidUnitNum=computed(()=> Math.ceil(totalTime.value / scale.value))
//总共要画多少个小单位
const unitNumToDraw=computed(()=> {
  if(timeUnitWidth.value * (maxValidUnitNum.value + 1) < window.innerWidth){
    //补充到window宽度
    return Math.floor(window.innerWidth / timeUnitWidth.value)
  }else{
    return maxValidUnitNum.value + 10
  }
})

//一个小单位有多少像素，这里默认设置成10
const timeUnitWidth=ref(10)

//页面的总宽度
//如果不足屏幕宽度，就绘制屏幕宽度大小
const pageWidth=computed(()=> {
  let width=timeUnitWidth.value * (unitNumToDraw.value + 1)
  if(width < window.innerWidth)
    width=window.innerWidth

  return width
})

//最大的有效的x值
const maxValidX=computed(()=> (maxValidUnitNum.value + 1) * timeUnitWidth.value)

//当前时间游标是否处在拖动模式
const dragMode=ref(false)
//当前时间游标的x偏移量
const cursorX=ref(timeUnitWidth.value)

//时间轴滚动条的偏移量
const scrollOffset=ref(0)

//像素值与时间转换的工具函数（时间单位为毫秒）
const timeToPx= (time:number) => (time / scale.value + 1) * timeUnitWidth.value
const pxToTime=(px:number) => ((px / timeUnitWidth.value) - 1) * scale.value

//测试用的切片信息
const clips=ref(props.clips)

//记录所有切片左边以及右边端点的像素值
const clipLeftRightPx=ref([

])

//挂载事件
//把clips转成clipLeftRightPx
onMounted(()=>{
  for(let i=0;i<clips.value.length;i++){
    let clip=clips.value[i]

    let startPx=timeToPx(clip["startTime"])
    let endPx=timeToPx(clip["endTime"])

    clipLeftRightPx.value.push([startPx,endPx])
  }
})

const changeScale = (newScale : number)=>{
  let timeCursorTime=pxToTime(cursorX.value)

  scale.value=newScale

  //Reposition clips
  clipLeftRightPx.value=[]
  for(let i=0;i<clips.value.length;i++){
    let clip=clips.value[i]

    let startPx=timeToPx(clip["startTime"])
    let endPx=timeToPx(clip["endTime"])

    clipLeftRightPx.value.push([startPx,endPx])
  }

  //Reposition time cursor
  cursorX.value = timeToPx(timeCursorTime)

  moveScrollerToTimeCursor()
}

type DraggableCursor={
  moveTo: (newX:number)=>void,
  getActualTime: ()=>number
}

let timeCursorDraggable : DraggableCursor = {
  moveTo: (newX: number)=>{
    cursorX.value = newX
    emit("timeCursorChange",pxToTime(newX))
  },
  getActualTime: () => {
    return pxToTime(cursorX.value)
  }
}

const getClipBarDraggable=(clipIndex:number,left:boolean)=>{
  let draggable:DraggableCursor = {
    moveTo(newX:number):void{
      if(left){
        //检查，是否左边大于右边
        if(newX > clipLeftRightPx.value[clipIndex][1]){
          newX=clipLeftRightPx.value[clipIndex][1]
        }
        clipLeftRightPx.value[clipIndex][0]=newX
      }else{
        //检查，是否右边小于左边
        if(newX < clipLeftRightPx.value[clipIndex][0]){
          newX=clipLeftRightPx.value[clipIndex][0]
        }
        clipLeftRightPx.value[clipIndex][1]=newX
      }

      emit("clipDurationChange",{
        "index":clipIndex,
        "clip":{
          "startTime":clipLeftRightPx.value[clipIndex][0],
          "endTime":clipLeftRightPx.value[clipIndex][1]
        }
      })

    },
    getActualTime():number{
      let currentPx=clipLeftRightPx.value[clipIndex][left? 0 : 1]
      return pxToTime(currentPx)
    }
  }

  return draggable
}

//处理拖动事件
let currentDraggableCursor = timeCursorDraggable

const cursorMouseDown=(evt:MouseEvent,draggable:DraggableCursor)=>{
  dragMode.value=true
  currentDraggableCursor=draggable
}

const cursorMouseUp=()=>{
  dragMode.value=false
}

//处理区鼠标移动事件
const cursorMouseMove=(evt:MouseEvent)=>{
  if(dragMode.value){
    let newX : number = evt.clientX + scrollOffset.value
    if(newX < timeUnitWidth.value){
      //左越界
      dragMode.value=false
      currentDraggableCursor.moveTo(timeUnitWidth.value)
    }else if(newX > maxValidX.value + 20){
      //右越界
      dragMode.value=false
      currentDraggableCursor.moveTo(maxValidX.value)
    }else if(evt.offsetY < 2 || evt.offsetY > operateAreaHeight.value - 2){
      //上下越界
      dragMode.value=false
    }else
      currentDraggableCursor.moveTo(evt.clientX + scrollOffset.value)
    }
}

//滚动条滚动时的事件
const scrollMove=(evt: {scrollLeft: number, scrollTop: number})=>{
  scrollOffset.value = evt.scrollLeft
}

//获取当前时间游标的时间
const currentTime=computed(()=> currentDraggableCursor.getActualTime())

//阻止默认拖拽事件（防止禁止拖拽图标）
const preventDragEvent=(evt:DragEvent)=>{
  evt.preventDefault()
  evt.stopPropagation()
}

//只针对时间游标的改变，点击时间轴改变时间游标位置
const timeAreaInteract=(evt:PointerEvent)=>{
  let newX : number = evt.clientX + scrollOffset.value
  if(newX < timeUnitWidth.value){
    cursorX.value = timeUnitWidth.value
  }else if(newX > maxValidX.value){
    cursorX.value = maxValidX.value
  }else{
    cursorX.value=evt.clientX + scrollOffset.value
  }
}

//把时间游标移动到屏幕中心
const timeLineScroller = ref<any>()
const moveScrollerToTimeCursor=()=>{
  let left=cursorX.value - window.innerWidth / 2
  if(left < 0)
    left = 0

  timeLineScroller.value!.setScrollLeft(left)
}

const setTimeCursorTime=(newTime : number)=>{
  cursorX.value = timeToPx(newTime)
  moveScrollerToTimeCursor()
}

//计算格式化后的总时间
const formattedTotalTime=computed(()=> convertTimestamp(totalTime.value))
const formattedCurrentCursorTime=computed(()=> convertTimestamp(timeCursorDraggable.getActualTime()))

function convertTimestamp(timestamp) {
  // 创建一个新日期对象，用毫秒数初始化
  let date = new Date(timestamp);

  // 使用国际标准格式获取时间字符串
  let timeStr = date.toISOString();

  // 将时间字符串拆分为日期部分和时间部分，只保留时间部分
  let timeParts = timeStr.split("T")[1].split(".");

  // 获取小时，分钟，秒和毫秒
  let hours = date.getUTCHours();
  let minutes = date.getUTCMinutes();
  let seconds = date.getUTCSeconds();
  let milliseconds = timeParts[1];

  // 如果小时为0，就不显示；如果分钟为0，就显示为00
  let timeDisplay = (hours > 0 ? hours + ':' : '')
      + (minutes < 10 ? '0' + minutes : minutes)
      + ':' + (seconds < 10 ? '0' + seconds : seconds)
      + '.' + milliseconds.replace("Z","");

  return timeDisplay;
}

defineExpose({
  setTimeCursorTime
})

</script>

<template>

  <div :style="{height: headerHeight + 'px'}" class="header-area">
    <div style="color: white; font-size: .875rem">
<!--      展示时间-->
      <span style="color: white">{{formattedCurrentCursorTime}}</span>
      <span style="color: hsla(0,0%,100%,.6)"> / </span>
      <span style="color: hsla(0,0%,100%,.6)">{{formattedTotalTime}}</span>
    </div>

    <div style="position:absolute; right: 0; display: flex;">
<!--      放大和缩小-->
      <div @click="changeScale(scale / 2)">
        <svg t="1693634897326" class="icon" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" p-id="4046" width="32" height="32"><path d="M672.495448 771.063111C536.288206 861.520472 350.802077 846.716896 230.724063 726.652386 93.758646 589.702371 93.758646 367.662526 230.724063 230.712511 367.68948 93.762496 589.754297 93.762496 726.719714 230.712511 846.795912 350.775203 861.60252 536.236675 771.139544 672.42803 773.082599 674.008086 774.96705 675.706322 776.783972 677.52304L875.05372 775.781733C902.70344 803.428352 903.102472 847.853384 875.518408 875.434352 848.125328 902.824352 803.92872 903.040688 775.854586 874.969704L677.584842 776.711015C675.771042 774.897418 674.074508 773.01162 672.495448 771.063111L672.495448 771.063111ZM677.120149 677.058399C786.692482 567.498387 786.692482 389.86651 677.120149 280.306498 567.547815 170.746486 389.895962 170.746486 280.323628 280.306498 170.751294 389.86651 170.751294 567.498387 280.323628 677.058399 389.895962 786.61841 567.547815 786.61841 677.120149 677.058399ZM448 448 320 448 320 512 448 512 448 640 512 640 512 512 640 512 640 448 512 448 512 320 448 320 448 448Z" fill="#ffffff" p-id="4047"></path></svg>
      </div>

      <div @click="changeScale(scale * 2)" style="padding-left: 5px; padding-right: 5px">
        <svg t="1693634968619" class="icon" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" p-id="5182" width="32" height="32"><path d="M672.495448 771.063111C536.288206 861.520472 350.802077 846.716896 230.724063 726.652386 93.758646 589.702371 93.758646 367.662526 230.724063 230.712511 367.68948 93.762496 589.754297 93.762496 726.719714 230.712511 846.795912 350.775203 861.60252 536.236675 771.139544 672.42803 773.082599 674.008086 774.96705 675.706322 776.783972 677.52304L875.05372 775.781733C902.70344 803.428352 903.102472 847.853384 875.518408 875.434352 848.125328 902.824352 803.92872 903.040688 775.854586 874.969704L677.584842 776.711015C675.771042 774.897418 674.074508 773.01162 672.495448 771.063111L672.495448 771.063111ZM677.120149 677.058399C786.692482 567.498387 786.692482 389.86651 677.120149 280.306498 567.547815 170.746486 389.895962 170.746486 280.323628 280.306498 170.751294 389.86651 170.751294 567.498387 280.323628 677.058399 389.895962 786.61841 567.547815 786.61841 677.120149 677.058399ZM320 448 320 512 640 512 640 448 320 448Z" fill="#ffffff" p-id="5183"></path></svg>
      </div>
    </div>
  </div>

    <el-scrollbar ref="timeLineScroller" @scroll="scrollMove" :style="{height: (timeAreaHeight + operateAreaHeight) + 'px' }">
      <div style="position: relative" :style="{height: (timeAreaHeight + operateAreaHeight) + 'px' }">
        <div class="time-area">
          <!--    这里展示时间轴-->

          <div v-for="(n,i) in unitNumToDraw">
            <!--      大刻度-->
            <div class="no-select-text" v-if="i % 10 == 0" :style="{left: (i+1)*10 + 'px'}" style="color: rgba(255, 255, 255, 0.6);position: absolute;font-size: 12px;top: 0">
              {{i * (scale / 1000)}}
            </div>
            <div v-if="i % 10 == 0"  :style="{left: (i+1)*10 + 'px'}" class="timeline-unit" style="height: 16px; position: absolute; top: 16px; width: 10px;" />
            <!--        小刻度-->
            <div v-if="i % 10 != 0" :style="{left: (i+1)*10 + 'px'}" class="timeline-unit" style="height: 8px; position: absolute; top: 24px; width: 10px;" />
          </div>
        </div>

        <div class="time-area-interact"  @click="timeAreaInteract">
          <!--    时间轴的交互区域-->
        </div>

        <div class="time-edit-lower-part" draggable="false"
             @dragenter="preventDragEvent"
             @dragstart="preventDragEvent"
             @dragover="preventDragEvent"
             @mouseup="cursorMouseUp" @mousemove="cursorMouseMove" >
          <!--    这里展示高亮片段-->

          <div v-for="(pos,i) in clipLeftRightPx" class="time-line-highlight-area" :style="{left:pos[0]+'px',
                width:(pos[1]-pos[0])+'px'}"
               @dragenter="preventDragEvent"
               @dragstart="preventDragEvent"
               @dragover="preventDragEvent">
            <div class="highlight-left-bracket"
                 @mousedown="(evt:MouseEvent)=>cursorMouseDown(evt,getClipBarDraggable(i,true))"
                 @mousemove="cursorMouseMove"
                 @dragenter="preventDragEvent"
                 @dragstart="preventDragEvent"
                 @dragover="preventDragEvent"></div>
            <div class="highlight-right-bracket"
                 @mousedown="(evt:MouseEvent)=>cursorMouseDown(evt,getClipBarDraggable(i,false))"
                 @mousemove="cursorMouseMove"
                 @dragenter="preventDragEvent"
                 @dragstart="preventDragEvent"
                 @dragover="preventDragEvent"></div>
          </div>
        </div>

        <div class="time-edit-cursor" draggable="false" :style="{left:cursorX + 'px'}">
          <svg class="cursor-top" width="8" height="12" viewBox="0 0 8 12" fill="none">
            <path d="M0 1C0 0.447715 0.447715 0 1 0H7C7.55228 0 8 0.447715 8 1V9.38197C8 9.76074 7.786 10.107 7.44721 10.2764L4.44721 11.7764C4.16569 11.9172 3.83431 11.9172 3.55279 11.7764L0.552786 10.2764C0.214002 10.107 0 9.76074 0 9.38197V1Z" fill="#5297FF">
            </path>
          </svg>

          <div class="cursor-area" draggable="false" @drag="(evt:DragEvent)=>{evt.preventDefault()}"
               @mouseup="cursorMouseUp" @mousedown="(evt:MouseEvent)=>cursorMouseDown(evt,timeCursorDraggable)" @mousemove="cursorMouseMove">

          </div>
        </div>
      </div>
    </el-scrollbar>
</template>

<style scoped>

.header-area{
  display: flex;
  flex-direction: row;
  position: relative;
  justify-content: center;
  background-color: #020617;
  width: 100%;
}

.highlight-right-bracket{
  background-color: #a855f7;
  left: calc(100% - 4px);
  top: 0;
  width: 4px;
  height: 100%;
  border-radius: 4px;
  position: absolute;
  cursor: ew-resize;
  z-index: 999;
}

.highlight-left-bracket{
  background-color: #a855f7;
  left: 0;
  width: 4px;
  height: 100%;
  border-radius: 4px;
  position: absolute;
  cursor: ew-resize;
  z-index: 999;
}

.time-line-highlight-area{
  height: 30%;
  top: 20%;
  position: absolute;
  background-color: #6b21a8;
}

.no-select-text {
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  -khtml-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
}

.timeline-unit{
  border-left: 1px solid rgba(255, 255, 255, 0.2);
  box-sizing: content-box;
  height: 4px;
  width: 10px;
  position: absolute;
  bottom: 0;
  top: auto;
}

.time-area{
  height: v-bind(timeAreaHeight + 'px');
  background-color: #020617;
  width: v-bind(pageWidth + 'px');
  top: v-bind(headerHeight + 'px');
}

.time-area-interact{
  height: v-bind(timeAreaHeight + 'px');
  width: v-bind(pageWidth + 'px');
  position: absolute;
  top: 0;
}

.time-edit-lower-part{
  background-color: #020617;
  position: relative;
  top: 0;
  height: v-bind(operateAreaHeight + 'px');
  width: v-bind(pageWidth + 'px');
  z-index: 99;
}

.time-edit-cursor{
  width: 1px;
  position: absolute;
  border-left: 1px solid #5297FF;
  border-right: 1px solid #5297FF;
  transform: translateX(-25%) scaleX(0.5);
  top: 32px;
  cursor: ew-resize;
  box-sizing: border-box;
  height: v-bind(operateAreaHeight + 'px');
  z-index: 999;
}

.cursor-top{
  transform: translate(-50%, 0) scaleX(2);
}

.cursor-area{
  width: 32px;
  cursor: ew-resize;
  position: absolute;
  top: 0;
  left: 50%;
  transform: translateX(-50%);
  height: v-bind(operateAreaHeight + 'px')
}
</style>