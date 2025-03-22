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
import { Ref, computed, nextTick, onBeforeMount, onBeforeUnmount, onMounted, onUnmounted, ref, render, useTemplateRef } from "vue";

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
let wasDomChanged = false;

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

      const isFirstCoupleRows = rowIndex <= 1;
      const isLastCoupleRows = rowIndex >= rendered.value.length - 2;
      const isFirstRow = rowIndex === 0;
      const isLastRow = rowIndex === rendered.value.length - 1;

      if (!isFirstCoupleRows && !isLastCoupleRows) {
        return;
      }

      const rowHeight = heights.value[rowIndex][0];

      /* // If a row just scrolled out of view, un-render it and increase the height of the scroll container accordingly to compensate
      if (!entry.isIntersecting) {

        if (isFirstCoupleRows && currentStartingRow < heights.value.length - 1) {
          rendered.value.splice(0, rowIndex + 1);
          currentStartingRow++;
          startPadding += rowHeight;
          wasDomChanged = true;
        }
        else if (isLastCoupleRows && currentStartingRow > 0) {
          rendered.value.splice(rowIndex);
          currentStartingRow--;
          endPadding += rowHeight;
          wasDomChanged = true;
        }
      } */

      // scrolling down, last row comes fully into view, render next row - decrease end padding
      if (isLastRow && entry.isIntersecting && entry.intersectionRatio === 1 && currentEndingRow < heights.value.length - 1) {
        rendered.value.push(heights.value[currentEndingRow]);
        currentEndingRow++;

        if (endPadding > 0) {
          endPadding -= heights.value[currentEndingRow][0];
        }
        
        wasDomChanged = true;
      }

      // scrolling up, last row (or second last, if just rendered a new one) goes out of view, remove current row to the end - increase end padding
      if (isLastCoupleRows && !entry.isIntersecting && entry.intersectionRatio === 0 && currentEndingRow > 0) {
        const removedRows = rendered.value.splice(rowIndex);
        currentEndingRow -= removedRows.length;
        let removedRowHeight = 0;

        for (let row of removedRows) {
          removedRowHeight += row[0];
        }

        // add height of padding between removed rows as well
        removedRowHeight += (removedRows.length - 1) * 20;

        endPadding += removedRowHeight;
        wasDomChanged = true;
      }

      // scrolling up, first row comes fully into view, render the row before - decrease start padding
      if (isFirstRow && entry.isIntersecting && entry.intersectionRatio === 1 && currentStartingRow > 0) {
        currentStartingRow--;
        rendered.value.unshift(heights.value[currentStartingRow]);

        if (startPadding > 0) {
          startPadding -= heights.value[currentStartingRow][0];
        }
        
        wasDomChanged = true;
      }

      // scrolling down, first row (or second, if just rendered a new one) goes out of view, remove start to current row - increase start padding
      if (isFirstCoupleRows && !entry.isIntersecting && entry.intersectionRatio === 0 && currentStartingRow > 0) {
        const removedRows = rendered.value.splice(0, rowIndex);
        currentStartingRow += removedRows.length;
        let removedRowHeight = 0;

        for (let row of removedRows) {
          removedRowHeight += row[0];
        }

        // add height of padding between removed rows as well
        removedRowHeight += (removedRows.length - 1) * 20;

        startPadding += removedRowHeight;        
        wasDomChanged = true;
      }


      /* // If a row just scrolled into view, render the next row and decrease the height of the scroll container accordingly to compensate
      if (entry.isIntersecting && entry.intersectionRatio >= .95) {
        if (isFirstRow && currentStartingRow > 0) {
          currentStartingRow--;
          rendered.value.unshift(heights.value[currentStartingRow]);
          if (startPadding > 0) {
            startPadding -= rowHeight;
          }
          wasDomChanged = true;
        }
 
        if (isLastRow && currentEndingRow < heights.value.length-1) {
          currentEndingRow++;
          rendered.value.push(heights.value[currentEndingRow]);
          if (endPadding > 0) {
            endPadding -= rowHeight;
          }
          wasDomChanged = true;
        }
      } */
    });

    if (wasDomChanged) {
      wasDomChanged = false;
      await reobserve();
    }
  }, { 
    threshold: [0, 1] 
    //rootMargin: '20px 0px 20px 0px' 
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
