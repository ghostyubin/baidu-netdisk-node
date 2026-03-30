<template>
  <div
    style="
      border: 1px solid var(--border);
      border-radius: var(--radius-md);
      background: var(--surface);
      padding: 10px 12px;
      box-shadow: var(--shadow-sm);
    "
  >
    <div class="direction mb-8 flex items-center">
      <i
        v-if="type === 'upload'"
        class="iconfont icon-arrow-up-long text-16 mr-4 text-orange-600"
      ></i>
      <i
        v-if="type === 'download'"
        class="iconfont icon-arrow-up-long text-16 mr-4 rotate-180 text-blue-600"
      ></i>
      <div class="flex flex-1 items-center gap-8">
        <div
          :class="['h-24 w-24 bg-contain bg-center bg-no-repeat', getFileIconClass()].join(' ')"
        ></div>
        <div class="flex-1 overflow-hidden text-ellipsis whitespace-nowrap">{{ filename }}</div>
      </div>
      <div class="flex items-center">
        <IconButton
          v-if="props.stepStatus === EStepStatus.STOPPED"
          icon-class="icon-play"
          @click="onPlayClick"
        ></IconButton>

        <IconButton
          v-if="props.stepStatus === EStepStatus.RUNNING"
          icon-class="icon-pause"
          @click="onPauseClick"
        ></IconButton>

        <IconButton
          icon-class="icon-close"
          @click="onDeleteClick"
        ></IconButton>
      </div>
    </div>

    <Progress
      :percentage="percentage"
      :type="props.type"
      class="mb-8"
    ></Progress>

    <div class="flex items-center gap-8">
      <div
        v-if="props.stepStatus !== EStepStatus.STOPPED"
        class="loader h-24 w-24"
      ></div>
      <div
        :class="getStatusClass()"
        class="mr-8"
      >
        {{ getStatusText() }}
      </div>
      <div class="flex-1">
        <div
          v-if="props.stepStatus === EStepStatus.RUNNING"
          class="flex items-center justify-end gap-12"
        >
          <span
            v-if="speedText"
            class="task-speed-badge"
          >{{ speedText }}</span>
          <span class="task-pct">{{ percentage.toFixed(2) }}%</span>
        </div>
        <div
          v-if="props.stepStatus === EStepStatus.STOPPED"
          class="truncate text-red-600"
        >
          {{ props.stepErrMsg }}
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { httpActTask } from '@src/common/api'
import { __FILEICONS__, getDownloadStepName, getUploadStepName } from '@src/common/const'
import Dialog from '@src/ui-components/dialog'
import IconButton from '@src/ui-components/icon-button.vue'
import Message from '@src/ui-components/message'
import Progress from '@src/ui-components/progress.vue'
import { EDownloadSteps, EStepStatus, EUploadSteps } from 'baidu-netdisk-sdk/types'
import { type IHttpTaskInfoItem } from 'baidu-netdisk-srv/types'
import { computed, ref, watch } from 'vue'

type IProps = IHttpTaskInfoItem & {
  type: 'upload' | 'download'
}

const props = defineProps<IProps>()

function getStatusClass() {
  if (props.stepStatus === EStepStatus.STOPPED) {
    return 'text-red-600'
  }

  return ''
}

function getStatusText() {
  if (props.stepStatus === EStepStatus.CREATED) {
    return '已创建'
  }

  if (props.stepStatus === EStepStatus.RUNNING) {
    if (props.type === 'upload') {
      return getUploadStepName(props.stepId as EUploadSteps)
    }

    if (props.type === 'download') {
      return getDownloadStepName(props.stepId as EDownloadSteps)
    }

    return '运行中'
  }

  if (props.stepStatus === EStepStatus.STOPPED) {
    return '已停止:'
  }

  if (props.stepStatus === EStepStatus.FINISHED) {
    return '已完成'
  }

  return '无状态'
}

async function onPlayClick() {
  try {
    await httpActTask({
      id: props.id,
      type: props.type,
      action: 'play',
    })
    Message.success('启动任务成功')
  } catch (inErr) {
    Message.error(`启动任务失败: ${(inErr as Error).message}`)
  }
}

async function onPauseClick() {
  try {
    await httpActTask({
      id: props.id,
      type: props.type,
      action: 'pause',
    })
    Message.success('暂停任务成功')
  } catch (inErr) {
    Message.error(`暂停任务失败: ${(inErr as Error).message}`)
  }
}

async function onDeleteClick() {
  if (!(await Dialog.confirm({ title: '删除任务', okText: '删除', okType: 'error' }))) {
    return
  }

  try {
    await httpActTask({
      id: props.id,
      type: props.type,
      action: 'del',
    })
    Message.success('删除任务成功')
  } catch (inErr) {
    Message.error(`删除任务失败: ${(inErr as Error).message}`)
  }
}

const filename = computed(() => {
  return props.local.split('/').pop()
})

const fileSuffix = computed(() => {
  return (props.local.split('.').pop() || '').toLowerCase()
})

function getFileIconClass() {
  for (const item of __FILEICONS__) {
    if (item.matches.includes(fileSuffix.value)) {
      return item.className
    }
  }

  return 'png-unknown'
}

const percentage = computed(() => {
  if (props.type === 'upload') {
    if (props.stepId === EUploadSteps.VERIFY_DOWNLOAD) {
      return (props.downBytes / props.comSize) * 100
    }

    return (props.upBytes / props.comSize) * 100
  } else {
    return (props.downBytes / props.comSize) * 100
  }
})

// ── 速度计算 ──
// 记录上一次轮询时的 bytes 值和时间戳，diff / dt = 速度
const _prevBytes = ref(0)
const _prevTime = ref(Date.now())
const speedText = ref('')

// 当前任务的有效字节数（根据类型和步骤取 upBytes 或 downBytes）
const activeBytes = computed(() => {
  if (props.type === 'upload') {
    return props.stepId === EUploadSteps.VERIFY_DOWNLOAD ? props.downBytes : props.upBytes
  }
  return props.downBytes
})

watch(activeBytes, (newVal) => {
  const now = Date.now()
  const dt = (now - _prevTime.value) / 1000  // 秒

  if (dt > 0.2 && props.stepStatus === EStepStatus.RUNNING) {
    const diff = newVal - _prevBytes.value
    if (diff > 0) {
      speedText.value = formatSpeed(diff / dt)
    } else if (diff === 0) {
      // bytes 没变化时保留上次速度，不清零（避免闪烁）
    }
  }

  _prevBytes.value = newVal
  _prevTime.value = now
})

// 状态变为非运行时清空速度
watch(() => props.stepStatus, (s) => {
  if (s !== EStepStatus.RUNNING) {
    speedText.value = ''
    _prevBytes.value = 0
    _prevTime.value = Date.now()
  }
})

function formatSpeed(bytesPerSec: number): string {
  if (bytesPerSec >= 1024 * 1024 * 1024) {
    return (bytesPerSec / (1024 * 1024 * 1024)).toFixed(1) + ' GB/s'
  }
  if (bytesPerSec >= 1024 * 1024) {
    return (bytesPerSec / (1024 * 1024)).toFixed(1) + ' MB/s'
  }
  if (bytesPerSec >= 1024) {
    return (bytesPerSec / 1024).toFixed(1) + ' KB/s'
  }
  return bytesPerSec.toFixed(0) + ' B/s'
}
</script>

<style scoped>
.task-speed-badge {
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.02em;
  color: var(--accent, #4f6ef7);
  background: color-mix(in srgb, var(--accent, #4f6ef7) 12%, transparent);
  border: 1px solid color-mix(in srgb, var(--accent, #4f6ef7) 25%, transparent);
  border-radius: 4px;
  padding: 1px 6px;
  white-space: nowrap;
}

.task-pct {
  font-size: 12px;
  color: var(--text-muted, #9ca3af);
  white-space: nowrap;
}
</style>
