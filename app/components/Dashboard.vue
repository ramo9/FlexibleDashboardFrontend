<template>
  <div ref="mainRef" :style="{
    position: 'relative',
    width: '100%',
    minWidth: `${containerMinWidth}px`,
    height: `${containerHeight}px`,
    boxSizing: 'border-box',
  }">
    <!-- Background Grid with SVG Lines -->
    <svg v-if="isEdit" :style="{
      position: 'absolute',
      top: 0,
      left: 0,
      width: '100%',
      height: '100%',
      pointerEvents: 'none',
      zIndex: 0
    }" :width="gridDimensions.width" :height="containerHeight">
      <line v-for="col in gridColumns + 1" :key="`v-cell-${col}`"
        :x1="(col - 1) * gridDimensions.cellWidth + (col - 1) * gap" :y1="0"
        :x2="(col - 1) * gridDimensions.cellWidth + (col - 1) * gap" :y2="containerHeight" :stroke="stokeColor"
        stroke-width="0.5" />
      <line v-for="col in gridColumns" :key="`v-gap-${col}`" :x1="col * gridDimensions.cellWidth + (col - 1) * gap"
        :y1="0" :x2="col * gridDimensions.cellWidth + (col - 1) * gap" :y2="containerHeight" :stroke="stokeColor"
        stroke-width="0.5" />
      <line v-for="row in Math.ceil(containerHeight / (cellHeight + gap)) + 1" :key="`h-cell-${row}`" :x1="0"
        :y1="(row - 1) * cellHeight + (row - 1) * gap" :x2="gridDimensions.width"
        :y2="(row - 1) * cellHeight + (row - 1) * gap" :stroke="stokeColor" stroke-width="0.5" />

      <line v-for="row in Math.ceil(containerHeight / (cellHeight + gap))" :key="`h-gap-${row}`" :x1="0"
        :y1="row * cellHeight + (row - 1) * gap" :x2="gridDimensions.width" :y2="row * cellHeight + (row - 1) * gap"
        :stroke="stokeColor" stroke-width="0.5" />
    </svg>

    <!-- Draggable Cards -->
    <div v-for="item in (items || [])" :key="item[pk]" :style="{
      position: 'absolute',
      left: `${item.x}px`,
      top: `${item.y}px`,
      width: `${item.width}px`,
      height: `${item.height}px`,
      minWidth: `${(gridDimensions.cellWidth * 2) + gap}px`,
      minHeight: `${(cellHeight * 2) + gap}px`,
      boxSizing: 'border-box',
      transition: isEdit ? dragState.dragIndex === item[pk] ? 'none' : 'all 0.2s ease' : undefined,
      zIndex: dragState.dragIndex === item[pk] ? 1000 : 1,
      cursor: isEdit ? 'move' : undefined,
      userSelect: isEdit ? 'none' : undefined,
      overflow: isEdit ? 'hidden' : undefined,
    }" @mousedown="!isEdit ? () => { } : startDrag($event, item[pk])" @mouseover="hoveredItem = item[pk]"
      @mouseleave="hoveredItem = null">
      <!-- Slot -->
      <slot :data="item" />

      <!-- Resize Handle -->
      <div v-if="isEdit && (hoveredItem === item[pk] || resizeState.resizeIndex === item[pk])" style="
            position: absolute; 
            bottom: -2px; 
            right: -2px; 
            width: 12px; 
            height: 12px; 
            background: #1976d2; 
            cursor: se-resize; 
            border-radius: 2px;
          " @mousedown.stop="startResize($event, item[pk])" />
    </div>

    <!-- Drag Preview -->
    <div v-if="dragState.isDragging && dragState.previewPosition && isEdit" :class="[`oc-drag-preview-style`]" :style="{
      position: 'absolute',
      left: `${dragState.previewPosition.x}px`,
      top: `${dragState.previewPosition.y}px`,
      width: `${dragState.previewPosition.width}px`,
      height: `${dragState.previewPosition.height}px`,
      minWidth: `${(gridDimensions.cellWidth * 2) + gap}px`,
      minHeight: `${(cellHeight * 2) + gap}px`,
      backgroundColor: dragState.previewPosition.isValid ? 'rgba(76, 175, 80, 0.3)' : 'rgba(244, 67, 54, 0.3)',
      border: `2px solid ${dragState.previewPosition.isValid ? '#4caf50' : '#f44336'}`,
      zIndex: 999,
      pointerEvents: 'none'
    }" />
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick } from 'vue'

const props = defineProps(['pk', 'gap'])
const emit = defineEmits(['update:modelValue','onResizeMove','onResizeEnd'])

defineExpose({
  add: addNewItem,
  remove: removeItem,
})

// Refs
const stokeColor = ref('#e5e7eb')
const mainRef = ref()
const gridDimensions = ref({ width: 0, cellWidth: 0 })
const hoveredItem = ref(null)
const isInitialized = ref(false)
const items = defineModel()
const isEdit = defineModel('isEdit')

// Constants
const gap = computed(() => props.gap || gridColumns.value)
const cellHeight = 40

// Computed
const pk = computed(() => props.pk ?? 'Id')
const gridColumns = computed(() => {
  return 12
})
const containerHeight = computed(() => {
  if (!items.value || items.value.length === 0) return 400
  const maxBottom = Math.max(...items.value.map(item => item.y + item.height), 400)
  const fullHeight = document.documentElement.clientHeight;
  const height = fullHeight - mainRef.value?.offsetTop;

  return isEdit.value ? (maxBottom < height ? height : maxBottom + 150) : maxBottom
})
const containerMinWidth = computed(() => {
  return (cellHeight * gridColumns.value) + (gap.value * (gridColumns.value - 1));
})

// State
const dragState = ref({
  isDragging: false,
  dragIndex: null,
  startX: 0,
  startY: 0,
  offsetX: 0,
  offsetY: 0,
  previewPosition: null
})

const resizeState = ref({
  isResizing: false,
  resizeIndex: null,
  startX: 0,
  startY: 0,
  startWidth: 0,
  startHeight: 0
})

// Helper functions
const gridToPixel = (gridX, gridY) => {
  const x = gridX * (gridDimensions.value.cellWidth + gap.value)
  const y = gridY * (cellHeight + gap.value)
  return { x, y }
}

const pixelToGrid = (x, y) => {
  const gridX = Math.round(x / (gridDimensions.value.cellWidth + gap.value))
  const gridY = Math.round(y / (cellHeight + gap.value))
  return { gridX, gridY }
}

const snapToGrid = (x, y) => {
  const snappedX = Math.round(x / (gridDimensions.value.cellWidth + gap.value)) * (gridDimensions.value.cellWidth + gap.value)
  const snappedY = Math.round(y / (cellHeight + gap.value)) * (cellHeight + gap.value)
  return { x: snappedX, y: snappedY }
}

const isValidPosition = (x, y, width, height, excludeId = null) => {
  if (x < 0 || y < 0 || x + width > gridDimensions.value.width) {
    return false
  }

  if (!items.value || items.value.length === 0) {
    return true
  }

  for (const item of items.value) {
    if (item[pk.value] === excludeId) continue

    const overlap = !(
      x >= item.x + item.width ||
      x + width <= item.x ||
      y >= item.y + item.height ||
      y + height <= item.y
    )

    if (overlap) return false
  }

  return true
}

const findAvailablePosition = (width, height) => {
  const maxCols = gridColumns.value
  const maxRows = 50

  for (let row = 0; row < maxRows; row++) {
    for (let col = 0; col <= maxCols - width; col++) {
      const x = col * (gridDimensions.value.cellWidth + gap.value)
      const y = row * (cellHeight + gap.value)
      const itemWidth = width * gridDimensions.value.cellWidth + (width - 1) * gap.value
      const itemHeight = height * cellHeight + (height - 1) * gap.value

      if (isValidPosition(x, y, itemWidth, itemHeight)) {
        return { gridX: col, gridY: row }
      }
    }
  }

  // If no position found, place at the bottom
  const maxY = items.value && items.value.length > 0
    ? Math.max(...items.value.map(item => item.gridY + item.rowSpan), 0)
    : 0
  return { gridX: 0, gridY: maxY }
}

const updateItemPositions = () => {
  if (gridDimensions.value.cellWidth === 0 || !items.value) return

  items.value.forEach(item => {
    const currentWidth = gridColumns.value === 1 ? 1 : item.colSpan

    item.width = currentWidth * gridDimensions.value.cellWidth + (currentWidth - 1) * gap.value
    item.height = item.rowSpan * cellHeight + (item.rowSpan - 1) * gap.value

    const pixelPos = gridToPixel(item.gridX || 0, item.gridY || 0)
    item.x = pixelPos.x
    item.y = pixelPos.y
  })
}

const updateGridDimensions = async () => {
  if (!mainRef.value) return

  await nextTick()

  const gridWidth = mainRef.value.offsetWidth
  const currentGridColumns = gridColumns.value
  const totalGapWidth = gap.value * (currentGridColumns - 1)
  const cellWidth = (gridWidth - totalGapWidth) / currentGridColumns

  gridDimensions.value = { width: gridWidth, cellWidth }
  updateItemPositions()

  if (!isInitialized.value) {
    isInitialized.value = true
  }
}

// Item management
function addNewItem(item) {
  if (!isEdit.value || !item || !item[pk.value]) {
    console.error('Invalid item to add.');
    return;
  }

  if (items.value.some(existingItem => existingItem[pk.value] === item[pk.value])) {
    console.error('Item pk already exists');
    return;
  }

  const [w, h] = (item.size ?? '2x2').split('x').map(Number)
  const position = findAvailablePosition(w, h)
  const newItem = {
    ...{
      colSpan: w,
      rowSpan: h,
      gridX: position.gridX,
      gridY: position.gridY,
      x: 0,
      y: 0,
      width: 0,
      height: 0
    }, ...item
  }

  if (!items.value) {
    items.value = []
  }

  items.value.push(newItem)
  updateItemPositions()
}

function removeItem(itemId) {
  if (!items.value) return

  const index = items.value.findIndex(item => item[pk.value] === itemId)
  if (index !== -1) {
    items.value.splice(index, 1)
    updateItemPositions()
  }

  if (hoveredItem.value === itemId) {
    hoveredItem.value = null
  }
}

// Drag functionality
const startDrag = (event, itemId) => {
  if (resizeState.value.isResizing || !items.value) return

  const item = items.value.find(i => i[pk.value] === itemId)
  if (!item) return

  const rect = mainRef.value.getBoundingClientRect()
  const offsetX = event.clientX - rect.left - item.x
  const offsetY = event.clientY - rect.top - item.y

  dragState.value = {
    isDragging: true,
    dragIndex: itemId,
    startX: event.clientX,
    startY: event.clientY,
    offsetX: offsetX,
    offsetY: offsetY,
    previewPosition: null
  }

  document.addEventListener('mousemove', onDragMove)
  document.addEventListener('mouseup', onDragEnd)
  event.preventDefault()
}

const onDragMove = (event) => {
  if (!dragState.value.isDragging || !items.value) return

  const draggedItem = items.value.find(i => i[pk.value] === dragState.value.dragIndex)
  if (!draggedItem) return

  const rect = mainRef.value.getBoundingClientRect()
  const newX = event.clientX - rect.left - dragState.value.offsetX
  const newY = event.clientY - rect.top - dragState.value.offsetY
  const snapped = snapToGrid(newX, newY)
  const isValid = isValidPosition(snapped.x, snapped.y, draggedItem.width, draggedItem.height, dragState.value.dragIndex)

  dragState.value.previewPosition = {
    x: snapped.x,
    y: snapped.y,
    width: draggedItem.width,
    height: draggedItem.height,
    isValid
  }

  // Update visual position during drag
  draggedItem.x = newX
  draggedItem.y = newY
}

const onDragEnd = () => {
  if (!dragState.value.isDragging || !items.value) return

  const draggedItem = items.value.find(i => i[pk.value] === dragState.value.dragIndex)
  if (!draggedItem) return

  // Snap to valid position or revert
  if (dragState.value.previewPosition && dragState.value.previewPosition.isValid) {
    draggedItem.x = dragState.value.previewPosition.x
    draggedItem.y = dragState.value.previewPosition.y

    // Update grid position
    const gridPos = pixelToGrid(draggedItem.x, draggedItem.y)
    draggedItem.gridX = gridPos.gridX
    draggedItem.gridY = gridPos.gridY
  } else {
    // Revert to original position
    const pixelPos = gridToPixel(draggedItem.gridX, draggedItem.gridY)
    draggedItem.x = pixelPos.x
    draggedItem.y = pixelPos.y
  }

  // Reset drag state
  dragState.value = {
    isDragging: false,
    dragIndex: null,
    startX: 0,
    startY: 0,
    offsetX: 0,
    offsetY: 0,
    previewPosition: null
  }

  document.removeEventListener('mousemove', onDragMove)
  document.removeEventListener('mouseup', onDragEnd)
}

// Resize functionality
const startResize = (event, itemId) => {
  if (!items.value) return

  const item = items.value.find(i => i[pk.value] === itemId)
  if (!item) return

  resizeState.value = {
    isResizing: true,
    resizeIndex: itemId,
    startX: event.clientX,
    startY: event.clientY,
    startWidth: item.colSpan,
    startHeight: item.rowSpan
  }

  document.addEventListener('mousemove', onResizeMove)
  document.addEventListener('mouseup', onResizeEnd)
  event.preventDefault()
}

const onResizeMove = (event) => {
  if (!resizeState.value.isResizing || !items.value) return

  const item = items.value.find(i => i[pk.value] === resizeState.value.resizeIndex)
  if (!item) return

  const deltaX = event.clientX - resizeState.value.startX
  const deltaY = event.clientY - resizeState.value.startY
  const cellsX = Math.round(deltaX / (gridDimensions.value.cellWidth + gap.value))
  const cellsY = Math.round(deltaY / (cellHeight + gap.value))
  const maxWidth = gridColumns.value === 1 ? 1 : gridColumns.value
  const newWidth = Math.max(1, Math.min(resizeState.value.startWidth + cellsX, maxWidth))
  const newHeight = Math.max(1, resizeState.value.startHeight + cellsY)

  item.colSpan = newWidth
  item.rowSpan = newHeight

  const currentWidth = gridColumns.value === 1 ? 1 : newWidth
  item.width = currentWidth * gridDimensions.value.cellWidth + (currentWidth - 1) * gap.value
  item.height = newHeight * cellHeight + (newHeight - 1) * gap.value
  emit('onResizeMove',{...resizeState.value})
}

const onResizeEnd = () => {
  emit('onResizeEnd',{...resizeState.value})
  resizeState.value = {
    isResizing: false,
    resizeIndex: null,
    startX: 0,
    startY: 0,
    startWidth: 0,
    startHeight: 0
  }


  document.removeEventListener('mousemove', onResizeMove)
  document.removeEventListener('mouseup', onResizeEnd)
}

// Lifecycle
let resizeTimeout = null
const debouncedResize = () => {
  if (resizeTimeout) clearTimeout(resizeTimeout)
  resizeTimeout = setTimeout(() => {
    updateGridDimensions()
  }, 100)
}

onMounted(async () => {
  await updateGridDimensions()
  window.addEventListener('resize', debouncedResize)
})

onUnmounted(() => {
  window.removeEventListener('resize', debouncedResize)
  document.removeEventListener('mousemove', onDragMove)
  document.removeEventListener('mouseup', onDragEnd)
  document.removeEventListener('mousemove', onResizeMove)
  document.removeEventListener('mouseup', onResizeEnd)
  if (resizeTimeout) clearTimeout(resizeTimeout)
})
</script>

<style scoped>
.oc-drag-preview-style {
  position: absolute;
}
</style>