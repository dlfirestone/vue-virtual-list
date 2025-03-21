<template>
  <div class="wrapper" ref="wrapper">
    <p>itemCount: {{ itemCount }}</p>
    <p>rowCount: {{ rowCount }}</p>
    <p>renderedCount: {{ renderedCount }}</p>
    <p>heights: {{ heights.slice(0, 5) }}...</p>
    <p>rendered: {{ rendered }}</p>
    <p>scrollEnd: {{ scrollEnd }}</p>
    <p>items: {{ items }}</p>
    <div class="pad-start" :style="{height: `${startPadding}px`}"></div>
    <div class="grid">
      <template v-for="(row, rowIndex) in rendered">
        <div class="item" 
          v-for="(renderedItem, colIndex) in row"
          data-item-element
          :key="`row${rowIndex}-col${colIndex}`"
          :data-item-row="currentStartingRow + rowIndex"
          :data-item-col="colIndex">
          <div class="item__content"
            :style="{'height': `${renderedItem}px`}"></div>
        </div>
      </template>
    </div>
    <div class="grid__pad-end" :style="{height: `${endPadding}px`}"></div>
    <div class="scroll-end" ref="scrollEnd"></div>
  </div>
</template>

<script setup lang="ts">
import { Ref, computed, nextTick, onBeforeMount, onBeforeUnmount, onMounted, onUnmounted, ref, useTemplateRef } from "vue";

type Matrix = Array<Array<number>>;

defineProps<{  }>();

const itemCount = 50;
const rowCount = 3;
const gap = 20;
const maxRenderedRows = 4;

const scrollEnd = useTemplateRef('scrollEnd');
const wrapper = useTemplateRef('wrapper');
let items: Ref<Array<HTMLElement>> = ref([]);

let renderedCount = ref(0);
const heights: Ref<Matrix> = ref([]);

renderedCount.value = rowCount * maxRenderedRows;

let scrollObserver: IntersectionObserver;
let scrollEndObserver: IntersectionObserver;


const rendered: Ref<Matrix> = ref([]);
let currentStartingRow = 0;
let currentEndingRow = 0;
let startPadding = 0;
let endPadding = 0;

let isInitialObserve = true;

onMounted(async () => {
  for (let i=0; i<itemCount; i+=rowCount){
    const row: Array<number> = [];
    for (let j=0; j<rowCount; j++){
      row.push((Math.ceil(Math.random() * 200)));
    }
    heights.value.push(row);
  }

  rendered.value = heights.value.slice(0, 1);

  currentEndingRow = 0;

  await nextTick();

  //addScrollEndObserver();

  // not using ref here to prevent items from getting out of order - ref does not guarantee items stay in original DOM order
  items.value = Array.from(document.querySelectorAll('[data-item-element]'));
  addScrollObserver();
});

function getItemIndex(rowIndex: number, colIndex: number) {
  const itemRow = currentStartingRow + rowIndex;
  return (itemRow * rowCount) + colIndex;
}

function addScrollObserver(){
  scrollObserver = new IntersectionObserver(async (entries) => {

    entries.forEach(async (entry, index) => {
      if (index === entries.length - 1) {
        isInitialObserve = false;
        return;
      }

      if (isInitialObserve) {
        return;
      }

      let rowIndex = 0;
      let colIndex = 0;
      let isFirstInRow = false;

      const rowIndexRaw = (entry.target as HTMLElement).getAttribute('data-item-row');
      const colIndexRaw = (entry.target as HTMLElement).getAttribute('data-item-col');

      if (!rowIndexRaw || !colIndexRaw) {
        return;
      }

      rowIndex = parseInt(rowIndexRaw);
      colIndex = parseInt(colIndexRaw);

      if (isNaN(rowIndex) || isNaN(colIndex)){
        return;
      }

      isFirstInRow = colIndex === 0;

      // Only detect for the first item in the row so it's not firing multiple times
      if (!isFirstInRow) {
        return;
      }

      const isFirstRow = rowIndex === 0;
      const isLastRow = rowIndex === rendered.value.length - 1;

      if (!isFirstRow && !isLastRow) {
        return;
      }

      const rowHeight = heights.value[rowIndex][0]

      // If a row just scrolled out of view, un-render it and increase the height of the scroll container accordingly to compensate
      if (entry.intersectionRatio === 0) {
        rendered.value.splice(rowIndex, 1);

        if (isFirstRow) {
          currentStartingRow++;
          startPadding += rowHeight;
        }
        else if (isLastRow) {
          currentStartingRow--;
          endPadding += rowHeight;
        }
        
        await reobserve();

        return;
      }


      // If a row just scrolled into view, render the next row and decrease the height of the scroll container accordingly to compensate
      if (entry.intersectionRatio === 1) {
        if (isFirstRow && currentStartingRow > 0) {
          currentStartingRow--;
          rendered.value.unshift(heights.value[currentStartingRow]);
          if (startPadding > 0) {
            startPadding -= rowHeight;
          }
        
          await reobserve();
          return;
        }
 
        if (isLastRow && currentEndingRow < heights.value.length-1) {
          currentEndingRow++;
          rendered.value.push(heights.value[currentEndingRow]);
          if (endPadding > 0) {
            endPadding -= rowHeight;
          }
        
          await reobserve();
          return;
        }
      }
    });

    await nextTick();

    

  }, { threshold: [0, 1] });

  if (items.value) {
    items.value.forEach((item) => {
      scrollObserver?.observe(item);
    });    
  }
}

function addScrollEndObserver() {
  scrollEndObserver = new IntersectionObserver(async (entries) => {
    const scrollEnd = entries[0];

    if (renderedCount.value >= itemCount) {
      scrollEndObserver.disconnect();
      return;
    }
    else if (itemCount - renderedCount.value < rowCount) {
      renderedCount.value = itemCount;
    }
    else {
      if (scrollEnd.isIntersecting){
        renderedCount.value += rowCount;
      }
    }
    
    await nextTick();

    if (scrollEnd.isIntersecting){
      scrollEndObserver.disconnect();
      scrollEndObserver.observe(scrollEnd.target);
    }
  });

  if (scrollEnd.value) {
    scrollEndObserver.observe(scrollEnd.value);
  }  
}

async function reobserve() {
  await nextTick(); 

  isInitialObserve = true;
  items.value = Array.from(document.querySelectorAll('[data-item-element]'));
  items.value.forEach((item) => {
    scrollObserver.observe(item);
  });    
}

onBeforeUnmount(() => {
  scrollObserver.disconnect();
});
</script>

<style scoped lang="scss">

.pad-start,
.pad-end {
  display: inline-block;
  grid-column: 1 / -1;
}

.pad-start {
  grid-row: 1;
}

.pad-end {
  grid-row: -1;
}

.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  align-items: stretch;
  gap: 20px;
  grid-auto-flow: row;
}

.item {
  background-color: #ddd;
}

.scroll-end {
  height: 1px;
}
</style>
