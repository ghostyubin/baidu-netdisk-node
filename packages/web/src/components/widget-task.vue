<template>
  <div class="task-card">
    <!-- 顶部：文件信息 + 操作按钮 -->
    <div class="task-top">
      <!-- 左：文件图标 + 文件名 -->
      <div class="task-file-info">
        <div
          :class="['task-file-icon', 'bg-contain bg-center bg-no-repeat', getFileIconClass()]"
        ></div>
        <div class="task-filename">{{ filename }}</div>
      </div>

      <!-- 右：控制按钮 -->
      <div class="task-actions">
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
        <IconButton icon-class="icon-close" @click="onDeleteClick"></IconButton>
      </div>
    </div>

    <!-- 进度条 -->
    <Progress
      :percentage="percentage"
      :type="props.type"
      class="task-progress"
    ></Progress>

    <!-- 底部：状态 + 速度 + 百分比 -->
    <div class="task-bottom">
      <!-- 左：加载动画 + 状态文字 -->
      <div class="task-status">
        <div
          v-if="props.stepStatus !== EStepStatus.STOPPED"
          class="loader h-16 w-16"
          style="flex-shrink: 0"
        ></div>
        <span :class="getStatusClass()">{{ getStatusText() }}</span>
      </div>

      <!-- 右：上传/下载方向标签 + 速度 + 百分比 -->
      <div class="task-right-info">
        <!-- 错误信息 -->
        <span
          v-if="props.stepStatus === EStepStatus.STOPPED && props.stepErrMsg"
          class="task-err"
        >{{ props.stepErrMsg }}</span>

        <!-- 速度 -->
        <span v-if="speedText && props.stepStatus === EStepStatus.RUNNING" class="task-speed">
          {{ speedText }}
        </span>

        <!-- 方向标签 -->
        <span
          v-if="props.stepStatus === EStepStatus.RUNNING || props.stepStatus === EStepStatus.STOPPED"
          class="task-dir-tag"
          :class="props.type === 'upload' ? 'task-dir-tag-up' : 'task-dir-tag-down'"
        >
          {{ props.type === 'upload' ? '↑ 上传' : '↓ 下载' }}
        </span>
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
  if (props.stepStatus === EStepStatus.STOPPED) return 'text-red-500'
  return 'task-status-text'
}

function getStatusText() {
  if (props.stepStatus === EStepStatus.CREATED) return '已创建'
  if (props.stepStatus === EStepStatus.RUNNING) {
    if (props.type === 'upload') return getUploadStepName(props.stepId as EUploadSteps)
    if (props.type === 'download') return getDownloadStepName(props.stepId as EDownloadSteps)
    return '运行中'
  }
  if (props.stepStatus === EStepStatus.STOPPED) return '已停止'
  if (props.stepStatus === EStepStatus.FINISHED) return '已完成'
  return '无状态'
}

async function onPlayClick() {
  try {
    await httpActTask({ id: props.id, type: props.type, action: 'play' })
    Message.success('启动任务成功')
  } catch (inErr) {
    Message.error(`启动任务失败: ${(inErr as Error).message}`)
  }
}

async function onPauseClick() {
  try {
    await httpActTask({ id: props.id, type: props.type, action: 'pause' })
    Message.success('暂停任务成功')
  } catch (inErr) {
    Message.error(`暂停任务失败: ${(inErr as Error).message}`)
  }
}

async function onDeleteClick() {
  if (!(await Dialog.confirm({ title: '删除任务', okText: '删除', okType: 'error' }))) return
  try {
    await httpActTask({ id: props.id, type: props.type, action: 'del' })
    Message.success('删除任务成功')
  } catch (inErr) {
    Message.error(`删除任务失败: ${(inErr as Error).message}`)
  }
}

const filename = computed(() => props.local.split('/').pop() || props.local.split('\\').pop() || props.local)

const fileSuffix = computed(() => (props.local.split('.').pop() || '').toLowerCase())

function getFileIconClass() {
  for (const item of __FILEICONS__) {
    if (item.matches.includes(fileSuffix.value)) return item.className
  }
  return 'png-unknown'
}

const percentage = computed(() => {
  if (props.type === 'upload') {
    if (props.stepId === EUploadSteps.VERIFY_DOWNLOAD) return (props.downBytes / props.comSize) * 100
    return (props.upBytes / props.comSize) * 100
  }
  return (props.downBytes / props.comSize) * 100
})

// ── 速度计算 ──
const POLL_INTERVAL = 1000
const _prevBytes = ref(-1)
const _prevTime = ref(Date.now() - POLL_INTERVAL)
const speedText = ref('')

const activeBytes = computed(() => {
  if (props.type === 'upload') {
    return props.stepId === EUploadSteps.VERIFY_DOWNLOAD ? props.downBytes : props.upBytes
  }
  return props.downBytes
})

watch(activeBytes, (newVal) => {
  const now = Date.now()
  if (_prevBytes.value === -1) {
    _prevBytes.value = newVal
    _prevTime.value = now
    return
  }
  const dt = (now - _prevTime.value) / 1000
  if (dt > 0.1 && props.stepStatus === EStepStatus.RUNNING) {
    const diff = newVal - _prevBytes.value
    if (diff > 0) speedText.value = formatSpeed(diff / dt)
  }
  _prevBytes.value = newVal
  _prevTime.value = now
}, { immediate: true })

watch(() => props.stepStatus, (s) => {
  if (s !== EStepStatus.RUNNING) {
    speedText.value = ''
    _prevBytes.value = -1
    _prevTime.value = Date.now() - POLL_INTERVAL
  }
})

function formatSpeed(bytesPerSec: number): string {
  if (bytesPerSec >= 1024 ** 3) return (bytesPerSec / 1024 ** 3).toFixed(1) + ' GB/s'
  if (bytesPerSec >= 1024 ** 2) return (bytesPerSec / 1024 ** 2).toFixed(1) + ' MB/s'
  if (bytesPerSec >= 1024) return (bytesPerSec / 1024).toFixed(1) + ' KB/s'
  return bytesPerSec.toFixed(0) + ' B/s'
}
</script>

<style scoped>
.task-card {
  background: var(--bg, #f5f6fa);
  border: 1px solid var(--border);
  border-radius: var(--radius-md);
  padding: 12px 14px 10px;
}

/* 顶部 */
.task-top {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 10px;
}
.task-file-info {
  display: flex;
  align-items: center;
  gap: 8px;
  flex: 1;
  min-width: 0;
}
.task-file-icon {
  width: 28px;
  height: 28px;
  flex-shrink: 0;
}
.task-filename {
  flex: 1;
  font-size: 14px;
  font-weight: 500;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  color: var(--text-primary);
}
.task-actions {
  display: flex;
  align-items: center;
  flex-shrink: 0;
}

/* 进度条 */
.task-progress {
  margin-bottom: 8px;
}

/* 底部 */
.task-bottom {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
}
.task-status {
  display: flex;
  align-items: center;
  gap: 6px;
  min-width: 0;
}
.task-status-text {
  font-size: 12px;
  color: var(--text-secondary);
  white-space: nowrap;
}
.task-right-info {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-shrink: 0;
}
.task-err {
  font-size: 12px;
  color: #ef4444;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  max-width: 180px;
}
.task-speed {
  font-size: 13px;
  font-weight: 600;
  color: #3b82f6;
}

/* 方向标签 */
.task-dir-tag {
  font-size: 12px;
  font-weight: 600;
  padding: 2px 8px;
  border-radius: 100px;
  white-space: nowrap;
}
.task-dir-tag-up {
  background: rgba(249, 115, 22, 0.10);
  color: #f97316;
}
.task-dir-tag-down {
  background: color-mix(in srgb, var(--accent) 10%, transparent);
  color: var(--accent);
}
</style>
