<template>
  <div class="rounded-8 h-6 w-full" style="background: var(--surface-2)">
    <div
      class="rounded-8 h-6 transition duration-200 ease-in-out"
      :style="{
        width: `${Math.min(props.percentage ?? 0, 100)}%`,
        background: fillColor,
      }"
    ></div>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'

interface IProps {
  percentage?: number
  /** 'upload' → orange, 'download' → accent blue, default → accent */
  type?: 'upload' | 'download' | 'storage'
}

const props = defineProps<IProps>()

const fillColor = computed(() => {
  if (props.type === 'upload') return '#f97316'
  if (props.type === 'storage') return '#4f6ef7'
  return 'var(--accent)'
})
</script>

