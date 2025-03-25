<template>
  <div class="wrapper" ref="wrapper">
    <p>itemCount: {{ itemCount }}</p>
    <p>rowCount: {{ rowCount }}</p>
    <p>renderedCount: {{ renderedCount }}</p>
    <p>heights: {{ heights.slice(0, 5) }}...</p>
    <!-- <p>rendered: {{ rendered }}</p> -->
    <p>scrollEnd: {{ scrollEnd }}</p>
    <!-- <p>items: {{ items }}</p> -->
    <div class="pad-start" :style="{height: `${startPadding}px`}"></div>
    <div class="grid">
      <template v-for="(row, rowIndex) in rendered">
        <div class="item" 
          v-for="(renderedItem, colIndex) in row"
          data-item-element
          :key="`row${rowIndex}-col${colIndex}`"
          :data-item-row="currentStartingRow + rowIndex"
          :data-rendered-row="rowIndex"
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
import { Ref, computed, nextTick, onBeforeMount, onBeforeUnmount, onMounted, onUnmounted, ref, render, useTemplateRef } from "vue";

type Matrix = Array<Array<number>>;
enum ScrollDirection {
  Up,
  Down
} 

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
let wasDomChanged = false;

let lastScrollY = 0;
let currentScrollY = 0;
let futureScrollY = 0;
let currentScrollDirection = ScrollDirection.Down;

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
  const thresholdIncrement = .1;
  const observerThresholds = [];

  for (let i=0; i<=1; i+=thresholdIncrement) {
    observerThresholds.push(i);
  }

  scrollObserver = new IntersectionObserver(async (entries) => {

    entries.forEach(async (entry, index) => {

      currentScrollY = window.scrollY;
      if (currentScrollY != lastScrollY) {
        currentScrollDirection = currentScrollY > lastScrollY ? ScrollDirection.Down : ScrollDirection.Up;
      }
      lastScrollY = currentScrollY;

      if (index === entries.length - 1) {
        isInitialObserve = false;
        return;
      }

      if (isInitialObserve) {
        return;
      }

      let rowIndex = 0;
      let renderedRowIndex = 0;
      let colIndex = 0;
      let isFirstInRow = false;

      const rowIndexRaw = (entry.target as HTMLElement).getAttribute('data-item-row');
      const renderedRowIndexRaw = (entry.target as HTMLElement).getAttribute('data-rendered-row');
      const colIndexRaw = (entry.target as HTMLElement).getAttribute('data-item-col');

      if (!rowIndexRaw || !renderedRowIndexRaw || !colIndexRaw) {
        return;
      }

      rowIndex = parseInt(rowIndexRaw);
      renderedRowIndex = parseInt(renderedRowIndexRaw);
      colIndex = parseInt(colIndexRaw);

      if (isNaN(rowIndex) || isNaN(colIndex)){
        return;
      }

      isFirstInRow = colIndex === 0;

      // Only detect for the first item in the row so it's not firing multiple times
      if (!isFirstInRow) {
        return;
      }

      if (entry.intersectionRatio >= .1 && entry.intersectionRatio <= .9) {
        return;
      } 

      const isFirstCoupleRows = renderedRowIndex <= 1;
      const isLastCoupleRows = renderedRowIndex >= rendered.value.length - 2;
      const isFirstRow = renderedRowIndex === 0;
      const isLastRow = renderedRowIndex === rendered.value.length - 1;

      if (!isFirstCoupleRows && !isLastCoupleRows) {
        return;
      }

      const rowHeight = heights.value[rowIndex][0];

      // scrolling down, last row comes fully into view, render next row - decrease end padding
      if (currentScrollDirection === ScrollDirection.Down
        && isLastRow 
        && entry.isIntersecting 
        && entry.intersectionRatio >= .901 
        && currentEndingRow < heights.value.length - 1
      ) {
        rendered.value.push(heights.value[currentEndingRow + 1]);
        currentEndingRow++;

        if (endPadding > 0) {
          endPadding -= heights.value[currentEndingRow][0] + gap;
        }
        
        wasDomChanged = true;
        return;
      }

      // scrolling down, first row (or second, if just rendered a new one) goes out of view, remove start to current row - increase start padding, add new row below
      if (currentScrollDirection === ScrollDirection.Down 
        && isFirstCoupleRows 
        && !entry.isIntersecting 
        && entry.intersectionRatio <= .099 
        && currentStartingRow < heights.value.length - 1
      ) {
        const removedRows = rendered.value.splice(0, renderedRowIndex + 1);
        currentStartingRow += removedRows.length;
        let removedRowHeight = 0;

        for (let row of removedRows) {
          removedRowHeight += Math.max(...row);
        }

        // add height of padding between removed rows as well
        removedRowHeight += removedRows.length * gap;

        startPadding += removedRowHeight;  

        // window.scrollTo(0, currentScrollY + removedRowHeight);
        currentScrollY = window.scrollY;
        
        // add new row below
        if (currentEndingRow < heights.value.length - 1) {
          currentEndingRow++;
          rendered.value.push(heights.value[currentEndingRow]);

          if (endPadding > 0) {
            endPadding -= heights.value[currentEndingRow][0] + gap;
          }
        }

        wasDomChanged = true;
        return;
      }

      // scrolling up, first row comes fully into view, render the row before - decrease start padding
      if (currentScrollDirection === ScrollDirection.Up 
        && isFirstRow 
        && entry.isIntersecting 
        && entry.intersectionRatio >= .901 
        && currentStartingRow > 0
      ) {
        currentStartingRow--;
        rendered.value.unshift(heights.value[currentStartingRow]);

        let removedPadding = Math.max(...heights.value[currentStartingRow]) + gap;

        if (startPadding > 0) {
          startPadding -= removedPadding;
        }

        // window.scrollTo(0, currentScrollY - removedPadding);
        currentScrollY = window.scrollY;
        
        wasDomChanged = true;
        return;
      }

      // scrolling up, last row (or second last, if just rendered a new one) goes out of view, remove current row to the end - increase end padding, add new row above
      if (currentScrollDirection === ScrollDirection.Up
        && isLastCoupleRows 
        && !entry.isIntersecting 
        && entry.intersectionRatio <= .099 
        && currentEndingRow > 0
      ) {
        const removedRows = rendered.value.splice(renderedRowIndex);
        currentEndingRow -= removedRows.length;
        let removedRowHeight = 0;

        for (let row of removedRows) {
          removedRowHeight += Math.max(...row);
        }

        // add height of padding between removed rows as well
        removedRowHeight += removedRows.length * 20;

        endPadding += removedRowHeight;

        // add new row above so scrollbar doesn't get stuck
        if (currentStartingRow > 0) {
          currentStartingRow--;
          rendered.value.unshift(heights.value[currentStartingRow]);

          if (startPadding > 0) {
            let rowHeight = Math.max(...heights.value[currentStartingRow]) + gap;
            startPadding -= rowHeight;
          }

          // window.scrollTo(0, currentScrollY + rowHeight);
        }

        wasDomChanged = true;
        return;
      }

    });

    if (wasDomChanged) {
      wasDomChanged = false;
      await reobserve();
    }
  }, { 
    threshold: observerThresholds
    //rootMargin: '50px 0px 50px 0px' 
  });

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

  window.scrollTo(0, currentScrollY);
}

onBeforeUnmount(() => {
  scrollObserver.disconnect();
});
</script>

<style scoped lang="scss">

.pad-start,
.pad-end {
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
