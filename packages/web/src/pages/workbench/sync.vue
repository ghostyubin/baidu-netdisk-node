<template>
  <div class="flex-1 overflow-auto pt-16">
    <div
      class="mx-auto"
      :class="config.isMobile ? 'max-w-full px-8' : 'max-w-[1280px] px-16'"
    >
      <div v-if="folders.length">
        <div
          v-for="folder in folders"
          :key="folder.id"
          class="mb-16 overflow-hidden"
          style="border-radius: var(--radius-lg); background: var(--surface); box-shadow: var(--shadow-md); border: 1px solid var(--border)"
        >
          <div
            class="flex items-center justify-between p-10"
            style="background: linear-gradient(135deg, var(--card-hd-from), var(--card-hd-to))"
          >
            <div class="flex items-center gap-8 flex-wrap">
              <!-- 状态指示点 -->
              <span
                class="sync-status-dot"
                :class="(folder.uploadTasks?.length || folder.downloadTasks?.length) ? 'active' : 'idle'"
              ></span>

              <!-- 加密 badge -->
              <Tooltip>
                <template #trigger>
                  <span class="sync-badge" :class="folder.encrypt ? 'badge-encrypt' : 'badge-noencrypt'">
                    {{ folder.encrypt ? '🔒 加密' : '🔓 未加密' }}
                  </span>
                </template>
                <div>加密状态: {{ folder.encrypt ? '已加密' : '未加密' }}</div>
              </Tooltip>

              <!-- 方向 badge -->
              <span
                class="sync-badge"
                :class="folder.direction === 1 ? 'badge-up' : folder.direction === 2 ? 'badge-down' : 'badge-both'"
              >
                {{ folder.direction === 1 ? '↑ 上传' : folder.direction === 2 ? '↓ 下载' : '⇅ 双向' }}
              </span>

              <!-- 定时图标 -->
              <Tooltip v-if="folder.nextStart">
                <template #trigger>
                  <i class="iconfont icon-timerstart text-16 text-blue-300"></i>
                </template>
                <div>下次启动: {{ getNextTime(folder.nextStart) }}</div>
              </Tooltip>
              <Tooltip v-if="folder.nextStop">
                <template #trigger>
                  <i class="iconfont icon-timerstop text-16 text-orange-300"></i>
                </template>
                <div>下次停止: {{ getNextTime(folder.nextStop) }}</div>
              </Tooltip>
            </div>

            <div class="flex items-center gap-4">
              <div
                v-if="folder.checking"
                class="loader h-24 w-24"
              ></div>
              <button
                v-else
                class="card-icon-btn"
                title="手动检查"
                @click="manualCheck(folder.id)"
              >
                <i class="iconfont icon-manual text-14"></i>
              </button>
              <button
                class="card-icon-btn"
                title="编辑"
                @click="modFolderId = folder.id"
              >
                <i class="iconfont icon-edit text-14"></i>
              </button>
              <button
                class="card-icon-btn"
                title="删除"
                @click="onDeleteClick(folder.id)"
              >
                <i class="iconfont icon-close text-14"></i>
              </button>
            </div>
          </div>

          <div class="flex flex-col gap-8 p-12">
            <!-- 路径展示框 -->
            <div class="sync-paths-box">
              <!-- 移动端：竖排 -->
              <template v-if="config.isMobile">
                <div class="sync-path-row">
                  <i class="iconfont icon-computer sync-path-icon"></i>
                  <span class="sync-path-text">{{ folder.local }}</span>
                </div>
                <div class="sync-path-arrow-v">
                  <i
                    v-if="folder.direction === 1"
                    class="iconfont icon-arrow-up-long font-bold text-orange-500"
                  ></i>
                  <i
                    v-else-if="folder.direction === 2"
                    class="iconfont icon-arrow-up-long rotate-180 font-bold text-blue-500"
                  ></i>
                  <i
                    v-else
                    class="iconfont icon-arrow-up-down left-orange-right-blue font-bold"
                  ></i>
                </div>
                <div class="sync-path-row">
                  <i class="iconfont icon-cloud sync-path-icon"></i>
                  <span class="sync-path-text">{{ folder.remote }}</span>
                </div>
              </template>

              <!-- 桌面端：横排 -->
              <template v-else>
                <div class="sync-path-row flex-1">
                  <i class="iconfont icon-computer sync-path-icon"></i>
                  <span class="sync-path-text">{{ folder.local }}</span>
                </div>
                <div class="sync-path-arrow-h">
                  <i
                    v-if="folder.direction === 1"
                    class="iconfont icon-arrow-up-long rotate-90 text-orange-500"
                  ></i>
                  <i
                    v-else-if="folder.direction === 2"
                    class="iconfont icon-arrow-up-long -rotate-90 text-blue-500"
                  ></i>
                  <i
                    v-else
                    class="iconfont icon-arrow-up-down left-orange-right-blue rotate-90"
                  ></i>
                </div>
                <div class="sync-path-row flex-1">
                  <i class="iconfont icon-cloud sync-path-icon"></i>
                  <span class="sync-path-text">{{ folder.remote }}</span>
                </div>
              </template>
            </div>

            <div
              v-if="folder.uploadTasks.length || folder.downloadTasks.length"
              class="flex flex-col gap-8"
            >
              <WidgetTask
                v-for="task in folder.uploadTasks"
                :key="task.id"
                v-bind="task"
                type="upload"
              />
              <WidgetTask
                v-for="task in folder.downloadTasks"
                :key="task.id"
                v-bind="task"
                type="download"
              />
            </div>

            <div
              v-if="folder.uploadQueue || folder.downloadQueue"
              class="flex items-center gap-8"
            >
              <Tooltip v-if="folder.uploadQueue">
                <template #trigger>
                  <div class="queue-badge queue-badge-upload">
                    <i class="iconfont icon-upload text-14"></i>
                    <span>{{ folder.uploadQueue }}</span>
                  </div>
                </template>
                <div>上传队列: {{ folder.uploadQueue }}</div>
              </Tooltip>

              <Tooltip v-if="folder.downloadQueue">
                <template #trigger>
                  <div class="queue-badge queue-badge-download">
                    <i class="iconfont icon-download text-14"></i>
                    <span>{{ folder.downloadQueue }}</span>
                  </div>
                </template>
                <div>下载队列: {{ folder.downloadQueue }}</div>
              </Tooltip>
            </div>

            <div
              v-if="
                !folder.uploadTasks.length &&
                !folder.downloadTasks.length &&
                !folder.uploadQueue &&
                !folder.downloadQueue
              "
              class="text-center text-gray-400"
            >
              无任务
            </div>
          </div>
        </div>
      </div>

      <div
        v-else
        class="flex flex-col items-center justify-center py-48"
        style="color: var(--text-muted)"
      >
        <i class="iconfont icon-empty" style="font-size: 72px; margin-bottom: 12px; opacity: 0.4"></i>
        <div>暂无同步目录</div>
      </div>
    </div>

    <ModalFolder
      v-if="!!modFolderId"
      :id="modFolderId"
      @ok="modFolderId = ''"
      @cancel="modFolderId = ''"
    />
  </div>
</template>

<script setup lang="ts">
import { httpDelFolder, httpFoldersInfo, httpManualCheck } from '@src/common/api'
import { config } from '@src/common/config'
import ModalFolder from '@src/components/modal-folder.vue'
import WidgetTask from '@src/components/widget-task.vue'
import Dialog from '@src/ui-components/dialog'
import Message from '@src/ui-components/message'
import Tooltip from '@src/ui-components/tooltip.vue'
import { type IHttpFoldersInfoRes } from 'baidu-netdisk-srv/types'
import dayjs from 'dayjs'
import { onMounted, onUnmounted, ref } from 'vue'

const folders = ref<IHttpFoldersInfoRes['folders']>([])
const modFolderId = ref('')
const timer = ref<number | undefined>(void 0)

onMounted(() => {
  getFoldersInfo()
})

onUnmounted(() => {
  if (timer.value) {
    clearTimeout(timer.value)
    timer.value = void 0
  }
})

async function getFoldersInfo() {
  try {
    const realTimeInfo = await httpFoldersInfo()
    folders.value = realTimeInfo.folders
  } catch (inErr) {
    Message.error(`获取目录失败: ${(inErr as Error).message}`)
  }

  timer.value = window.setTimeout(() => getFoldersInfo(), 3000)
}

async function onDeleteClick(inFolderId: string) {
  if (!(await Dialog.confirm({ title: '删除同步目录', okText: '删除', okType: 'error' }))) {
    return
  }

  try {
    await httpDelFolder({ id: inFolderId })
    Message.success('删除成功')
  } catch (inErr) {
    Message.error(`删除失败: ${(inErr as Error).message}`)
  }
}

async function manualCheck(inId: string) {
  try {
    await httpManualCheck({ id: inId })
    Message.success('手动检查成功')
  } catch (inErr) {
    Message.error(`手动检查失败: ${(inErr as Error).message}`)
  }
}

function getNextTime(inTime: number) {
  if (!inTime) {
    return '无'
  }

  const nextDate = dayjs(inTime)
  const nowDate = dayjs(Date.now())

  if (nextDate.isBefore(nowDate)) {
    return '已过'
  }

  if (nextDate.isSame(nowDate, 'day')) {
    return nextDate.format('今天 HH:mm')
  }

  if (nextDate.isSame(nowDate.add(1, 'day'), 'day')) {
    return nextDate.format('明天 HH:mm')
  }

  if (nextDate.isSame(nowDate.add(2, 'day'), 'day')) {
    return nextDate.format('后天 HH:mm')
  }

  return nextDate.format('MM-DD HH:mm')
}
</script>

<style scoped>
/* ── 状态指示点 ── */
.sync-status-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  flex-shrink: 0;
}
.sync-status-dot.active {
  background: #4ade80;
  box-shadow: 0 0 0 3px rgba(74, 222, 128, 0.25);
}
.sync-status-dot.idle {
  background: rgba(255, 255, 255, 0.35);
}

/* ── Badge pill ── */
.sync-badge {
  display: inline-flex;
  align-items: center;
  padding: 2px 8px;
  border-radius: 100px;
  font-size: 11px;
  font-weight: 600;
  white-space: nowrap;
  line-height: 1.6;
}
.badge-encrypt  { background: rgba(34, 197, 94, 0.2);  color: #86efac; }
.badge-noencrypt{ background: rgba(239, 68, 68, 0.2);  color: #fca5a5; }
.badge-up       { background: rgba(249, 115, 22, 0.2); color: #fdba74; }
.badge-down     { background: rgba(59, 130, 246, 0.2); color: #93c5fd; }
.badge-both     { background: rgba(168, 85, 247, 0.2); color: #d8b4fe; }

/* ── 卡片头图标按钮 ── */
.card-icon-btn {
  width: 28px;
  height: 28px;
  border-radius: var(--radius-sm);
  border: none;
  background: rgba(255, 255, 255, 0.1);
  color: rgba(255, 255, 255, 0.8);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.15s, color 0.15s;
}
.card-icon-btn:hover {
  background: rgba(255, 255, 255, 0.25);
  color: #fff;
}

/* ── 路径展示框 ── */
.sync-paths-box {
  display: flex;
  align-items: center;
  gap: 8px;
  background: var(--surface-2);
  border: 1px solid var(--border);
  border-radius: var(--radius-md);
  padding: 10px 12px;
  margin-bottom: 8px;
}
.sync-path-row {
  display: flex;
  align-items: center;
  gap: 6px;
  min-width: 0;
}
.sync-path-icon {
  font-size: 15px;
  flex-shrink: 0;
  color: var(--text-secondary);
}
.sync-path-text {
  font-size: 12px;
  color: var(--text-secondary);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  flex: 1;
  min-width: 0;
}
.sync-path-arrow-h {
  font-size: 16px;
  flex-shrink: 0;
}
.sync-path-arrow-v {
  display: flex;
  justify-content: center;
  font-size: 16px;
  padding: 2px 0;
}

/* ── 队列 badge ── */
.queue-badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 3px 10px;
  border-radius: 100px;
  font-size: 12px;
  font-weight: 600;
}
.queue-badge-upload {
  background: rgba(249, 115, 22, 0.12);
  color: #ea580c;
}
.queue-badge-download {
  background: rgba(59, 130, 246, 0.12);
  color: #2563eb;
}
</style>
