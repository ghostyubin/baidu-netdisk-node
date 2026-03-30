<template>
  <div class="sync-page">
    <!-- ── 顶部标题栏 ── -->
    <div class="sync-topbar">
      <h1 class="sync-title">同步任务</h1>
      <div class="sync-topbar-actions">
        <button class="sync-btn-refresh" title="刷新" @click="getFoldersInfo()">
          <i class="iconfont icon-refresh" style="font-size: 16px"></i>
        </button>
        <button class="sync-btn-add" @click="emit('requestAddFolder')">
          <span style="font-size: 18px; font-weight: 300; line-height: 1; margin-right: 2px">+</span>
          添加目录
        </button>
      </div>
    </div>

    <!-- ── 统计卡片 ── -->
    <div class="sync-stats">
      <div class="sync-stat-card">
        <div class="sync-stat-icon sync-stat-icon-sync">
          <i class="iconfont icon-arrow-up-down" style="font-size: 18px; color: var(--accent)"></i>
        </div>
        <div>
          <div class="sync-stat-label">同步目录</div>
          <div class="sync-stat-num sync-stat-num-default">{{ folders.length }}</div>
        </div>
      </div>
      <div class="sync-stat-card">
        <div class="sync-stat-icon sync-stat-icon-up">
          <i class="iconfont icon-upload" style="font-size: 18px; color: #f97316"></i>
        </div>
        <div>
          <div class="sync-stat-label">上传中</div>
          <div class="sync-stat-num sync-stat-num-up">{{ totalUploadCount }}</div>
        </div>
      </div>
      <div class="sync-stat-card">
        <div class="sync-stat-icon sync-stat-icon-down">
          <i class="iconfont icon-download" style="font-size: 18px; color: var(--accent)"></i>
        </div>
        <div>
          <div class="sync-stat-label">下载中</div>
          <div class="sync-stat-num sync-stat-num-down">{{ totalDownloadCount }}</div>
        </div>
      </div>
    </div>

    <!-- ── 同步目录列表 ── -->
    <div class="sync-list">
      <div v-if="folders.length">
        <div
          v-for="folder in folders"
          :key="folder.id"
          class="sync-card"
        >
          <!-- 卡片头 -->
          <div class="sync-card-hd">
            <div class="sync-card-hd-info flex items-center gap-8 flex-wrap">
              <!-- 状态点 -->
              <span
                class="sync-status-dot"
                :class="(folder.uploadTasks?.length || folder.downloadTasks?.length) ? 'active' : 'idle'"
              ></span>
              <!-- 目录名 -->
              <span class="sync-card-name">
                {{ folder.local.split('/').pop() || folder.local.split('\\').pop() || folder.local }}
              </span>

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

            <div class="sync-card-hd-actions flex items-center gap-4">
              <div v-if="folder.checking" class="loader h-24 w-24"></div>
              <button
                v-else
                class="card-icon-btn"
                title="手动检查"
                @click="manualCheck(folder.id)"
              >
                <i class="iconfont icon-refresh" style="font-size: 14px"></i>
              </button>
              <button class="card-icon-btn" title="编辑" @click="modFolderId = folder.id">
                <i class="iconfont icon-edit text-14"></i>
              </button>
              <button class="card-icon-btn" title="删除" @click="onDeleteClick(folder.id)">
                <i class="iconfont icon-close text-14"></i>
              </button>
            </div>
          </div>

          <!-- 卡片体 -->
          <div class="sync-card-body">
            <!-- 路径展示框 -->
            <div class="sync-paths-box">
              <template v-if="config.isMobile">
                <div class="sync-path-row">
                  <i class="iconfont icon-computer sync-path-icon"></i>
                  <span class="sync-path-text">{{ folder.local }}</span>
                </div>
                <div class="sync-path-arrow-v">
                  <i v-if="folder.direction === 1" class="iconfont icon-arrow-up-long font-bold text-orange-500"></i>
                  <i v-else-if="folder.direction === 2" class="iconfont icon-arrow-up-long rotate-180 font-bold text-blue-500"></i>
                  <i v-else class="iconfont icon-arrow-up-down left-orange-right-blue font-bold"></i>
                </div>
                <div class="sync-path-row">
                  <i class="iconfont icon-cloud sync-path-icon"></i>
                  <span class="sync-path-text">{{ folder.remote }}</span>
                </div>
              </template>
              <template v-else>
                <div class="sync-path-row flex-1">
                  <i class="iconfont icon-computer sync-path-icon"></i>
                  <span class="sync-path-text">{{ folder.local }}</span>
                </div>
                <div class="sync-path-arrow-h">
                  <i v-if="folder.direction === 1" class="iconfont icon-arrow-up-long rotate-90 text-orange-500"></i>
                  <i v-else-if="folder.direction === 2" class="iconfont icon-arrow-up-long -rotate-90 text-blue-500"></i>
                  <i v-else class="iconfont icon-arrow-up-down left-orange-right-blue rotate-90"></i>
                </div>
                <div class="sync-path-row flex-1">
                  <i class="iconfont icon-cloud sync-path-icon"></i>
                  <span class="sync-path-text">{{ folder.remote }}</span>
                </div>
              </template>
            </div>

            <!-- 任务列表 -->
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

            <!-- 队列 badge -->
            <div
              v-if="folder.uploadQueue || folder.downloadQueue"
              class="flex items-center gap-8 mt-4"
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

            <!-- 无任务 -->
            <div
              v-if="!folder.uploadTasks.length && !folder.downloadTasks.length && !folder.uploadQueue && !folder.downloadQueue"
              class="sync-empty-hint"
            >
              无任务
            </div>
          </div>
        </div>
      </div>

      <!-- 空状态 -->
      <div
        v-else
        class="flex flex-col items-center justify-center py-64"
        style="color: var(--text-muted)"
      >
        <i class="iconfont icon-empty" style="font-size: 72px; margin-bottom: 12px; opacity: 0.4"></i>
        <div>暂无同步目录</div>
        <button class="sync-btn-add mt-16" @click="emit('requestAddFolder')">
          <span style="font-size: 18px; font-weight: 300; line-height: 1; margin-right: 2px">+</span>
          添加目录
        </button>
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
import { computed, onMounted, onUnmounted, ref } from 'vue'

// 接收 workbench 传下来的 emit 事件
const emit = defineEmits<{
  requestAddFolder: []
}>()

const folders = ref<IHttpFoldersInfoRes['folders']>([])
const modFolderId = ref('')
const timer = ref<number | undefined>(void 0)

// 统计
const totalUploadCount = computed(() =>
  folders.value.reduce((s, f) => s + (f.uploadTasks?.length || 0), 0),
)
const totalDownloadCount = computed(() =>
  folders.value.reduce((s, f) => s + (f.downloadTasks?.length || 0), 0),
)

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
  timer.value = window.setTimeout(() => getFoldersInfo(), 1000)
}

async function onDeleteClick(inFolderId: string) {
  if (!(await Dialog.confirm({ title: '删除同步目录', okText: '删除', okType: 'error' }))) return
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
  if (!inTime) return '无'
  const nextDate = dayjs(inTime)
  const nowDate = dayjs(Date.now())
  if (nextDate.isBefore(nowDate)) return '已过'
  if (nextDate.isSame(nowDate, 'day')) return nextDate.format('今天 HH:mm')
  if (nextDate.isSame(nowDate.add(1, 'day'), 'day')) return nextDate.format('明天 HH:mm')
  if (nextDate.isSame(nowDate.add(2, 'day'), 'day')) return nextDate.format('后天 HH:mm')
  return nextDate.format('MM-DD HH:mm')
}
</script>

<style scoped>
/* ── 页面布局 ── */
.sync-page {
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
}

@media (max-width: 600px) {
  .sync-page {
    height: auto;
    min-height: 100%;
    overflow: visible;
  }
}

/* ── 顶部标题栏 ── */
.sync-topbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 20px 24px 0;
  flex-shrink: 0;
}
.sync-title {
  font-size: 22px;
  font-weight: 700;
  color: var(--text-primary);
  margin: 0;
}
.sync-topbar-actions {
  display: flex;
  align-items: center;
  gap: 10px;
}
.sync-btn-refresh {
  width: 36px;
  height: 36px;
  border-radius: var(--radius-md);
  border: 1px solid var(--border);
  background: var(--surface);
  color: var(--text-secondary);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: border-color 0.15s, color 0.15s;
  flex-shrink: 0;
}
.sync-btn-refresh:hover {
  border-color: var(--accent);
  color: var(--accent);
}
.sync-btn-add {
  display: flex;
  align-items: center;
  gap: 4px;
  height: 36px;
  padding: 0 16px;
  background: var(--accent);
  color: #fff;
  border: none;
  border-radius: var(--radius-md);
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
  flex-shrink: 0;
}
.sync-btn-add:hover {
  background: var(--accent-hover);
}

/* ── 统计卡片 ── */
.sync-stats {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 14px;
  padding: 16px 24px 0;
  flex-shrink: 0;
}
.sync-stat-card {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: var(--radius-lg);
  padding: 16px 18px;
  display: flex;
  align-items: center;
  gap: 14px;
  box-shadow: var(--shadow-sm);
}
.sync-stat-icon {
  width: 44px;
  height: 44px;
  border-radius: var(--radius-md);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}
.sync-stat-icon-sync { background: color-mix(in srgb, var(--accent) 10%, transparent); }
.sync-stat-icon-up   { background: rgba(249, 115, 22, 0.10); }
.sync-stat-icon-down { background: color-mix(in srgb, var(--accent) 10%, transparent); }
.sync-stat-label {
  font-size: 12px;
  color: var(--text-muted);
  margin-bottom: 4px;
}
.sync-stat-num {
  font-size: 28px;
  font-weight: 700;
  line-height: 1;
}

/* ── 移动端覆盖：统计卡片改为横向单行，内容紧凑 ── */
@media (max-width: 600px) {
  .sync-topbar {
    padding: 14px 16px 0;
  }
  .sync-title {
    font-size: 18px;
  }
  .sync-stats {
    grid-template-columns: repeat(3, 1fr);
    gap: 6px;
    padding: 8px 12px 0;
  }
  .sync-stat-card {
    padding: 7px 8px;
    gap: 7px;
    flex-direction: row;
    align-items: center;
  }
  .sync-stat-icon {
    width: 28px;
    height: 28px;
    border-radius: 8px;
    flex-shrink: 0;
  }
  .sync-stat-icon svg,
  .sync-stat-icon .iconfont {
    font-size: 14px;
  }
  .sync-stat-label {
    font-size: 10px;
    white-space: nowrap;
    overflow: visible;
    margin-bottom: 2px;
  }
  .sync-stat-num {
    font-size: 18px;
    line-height: 1;
  }
  .sync-list {
    padding: 12px 16px 16px;
  }
  .sync-card-hd {
    padding: 8px 10px;
    flex-wrap: nowrap;
    gap: 6px;
  }
  /* 标题区（左侧）不换行，超出截断 */
  .sync-card-hd-info {
    flex-wrap: nowrap;
    overflow: hidden;
    flex: 1;
    min-width: 0;
    gap: 5px;
  }
  .sync-card-name {
    font-size: 13px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    min-width: 0;
    flex-shrink: 1;
  }
  /* badge 压缩 */
  .sync-badge {
    font-size: 10px;
    padding: 2px 5px;
    white-space: nowrap;
    flex-shrink: 0;
  }
  /* 操作按钮区不换行、不压缩 */
  .sync-card-hd-actions {
    flex-shrink: 0;
    flex-wrap: nowrap;
    gap: 4px;
  }
  .card-icon-btn {
    width: 26px;
    height: 26px;
    padding: 0;
  }
  .sync-card-body {
    padding: 8px 10px;
  }
}
.sync-stat-num-default { color: var(--text-primary); }
.sync-stat-num-up      { color: #f97316; }
.sync-stat-num-down    { color: var(--accent); }

/* ── 任务列表区域 ── */
.sync-list {
  flex: 1;
  overflow-y: auto;
  padding: 16px 24px 24px;
}

/* ── 同步卡片 ── */
.sync-card {
  margin-bottom: 16px;
  border-radius: var(--radius-lg);
  background: var(--surface);
  box-shadow: var(--shadow-md);
  border: 1px solid var(--border);
  overflow: hidden;
}
.sync-card-hd {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 14px;
  background: linear-gradient(135deg, var(--card-hd-from), var(--card-hd-to));
  gap: 8px;
  flex-wrap: wrap;
}
.sync-card-name {
  font-size: 14px;
  font-weight: 600;
  color: #fff;
}
.sync-card-body {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 12px 14px;
}

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
.badge-encrypt   { background: rgba(34, 197, 94, 0.2);  color: #86efac; }
.badge-noencrypt { background: rgba(239, 68, 68, 0.2);  color: #fca5a5; }
.badge-up        { background: rgba(249, 115, 22, 0.2); color: #fdba74; }
.badge-down      { background: rgba(59, 130, 246, 0.2); color: #93c5fd; }
.badge-both      { background: rgba(168, 85, 247, 0.2); color: #d8b4fe; }

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
.queue-badge-upload   { background: rgba(249, 115, 22, 0.12); color: #ea580c; }
.queue-badge-download { background: rgba(59, 130, 246, 0.12); color: #2563eb; }

/* ── 空任务提示 ── */
.sync-empty-hint {
  text-align: center;
  color: var(--text-muted);
  font-size: 13px;
  padding: 8px 0;
}
</style>
