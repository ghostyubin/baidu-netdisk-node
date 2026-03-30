<template>
  <!-- Sidebar overlay (mobile) -->
  <div
    class="wb-sidebar-overlay"
    :class="{ open: sidebarOpen }"
    @click="sidebarOpen = false"
  ></div>

  <!-- Root layout -->
  <div class="flex h-full w-full" style="background: var(--bg)">
    <!-- ── Sidebar ── -->
    <aside
      class="wb-sidebar"
      :class="{ open: sidebarOpen }"
    >
      <!-- Logo / brand -->
      <div class="wb-logo-area">
        <div class="wb-logo-icon">
          <i class="iconfont icon-cloud" style="font-size: 20px; color: #fff"></i>
        </div>
        <span class="wb-logo-text">BaiduSync</span>
      </div>

      <!-- Nav links -->
      <nav class="wb-nav" style="flex: 1; overflow-y: auto">
        <!-- 功能分组 -->
        <div class="wb-nav-group-label">功能</div>

        <router-link
          :to="`/workbench/sync${currentSearchParams}`"
          class="wb-nav-link"
          :class="{ active: isCurrentPath('/workbench/sync') }"
          @click="sidebarOpen = false"
        >
          <i class="iconfont icon-arrow-up-down wb-nav-icon"></i>
          <span class="wb-nav-text">同步任务</span>
          <span
            v-if="syncTaskCount > 0"
            class="wb-nav-badge"
          >{{ syncTaskCount }}</span>
        </router-link>

        <router-link
          :to="`/workbench/disk${currentSearchParams}`"
          class="wb-nav-link"
          :class="{ active: isCurrentPath('/workbench/disk') && !isCurrentPath('/workbench/disk/tasks') }"
          @click="sidebarOpen = false"
        >
          <i class="iconfont icon-cloud wb-nav-icon" style="color: #f59e0b"></i>
          <span class="wb-nav-text">网盘文件</span>
        </router-link>

        <router-link
          :to="`/workbench/disk/tasks${currentSearchParams}`"
          class="wb-nav-link"
          :class="{ active: isCurrentPath('/workbench/disk/tasks') }"
          @click="sidebarOpen = false"
        >
          <i class="iconfont icon-download wb-nav-icon"></i>
          <span class="wb-nav-text">传输任务</span>
        </router-link>

        <!-- 系统分组 -->
        <div class="wb-nav-group-label" style="margin-top: 16px">系统</div>

        <button
          class="wb-nav-link"
          :class="{ active: modConfigDialogVisible }"
          @click="modConfigDialogVisible = true"
        >
          <!-- 齿轮 -->
          <svg class="wb-nav-icon" width="18" height="18" viewBox="0 0 24 24" fill="currentColor">
            <path d="M19.14 12.94c.04-.3.06-.61.06-.94s-.02-.64-.07-.94l2.03-1.58a.49.49 0 0 0 .12-.61l-1.92-3.32a.49.49 0 0 0-.59-.22l-2.39.96a7.1 7.1 0 0 0-1.62-.94l-.36-2.54A.484.484 0 0 0 14 2h-4a.484.484 0 0 0-.48.41l-.36 2.54a7.47 7.47 0 0 0-1.62.94l-2.39-.96a.48.48 0 0 0-.59.22L2.74 8.87a.47.47 0 0 0 .12.61l2.03 1.58c-.05.3-.09.63-.09.94s.02.64.07.94l-2.03 1.58a.49.49 0 0 0-.12.61l1.92 3.32c.12.22.37.29.59.22l2.39-.96c.5.36 1.04.67 1.62.94l.36 2.54c.05.27.29.47.56.47h4c.27 0 .51-.2.56-.47l.36-2.54a7.47 7.47 0 0 0 1.62-.94l2.39.96c.22.08.47.01.59-.22l1.92-3.32a.47.47 0 0 0-.12-.61l-2.01-1.58zM12 15.6A3.6 3.6 0 0 1 8.4 12 3.6 3.6 0 0 1 12 8.4a3.6 3.6 0 0 1 3.6 3.6 3.6 3.6 0 0 1-3.6 3.6z"/>
          </svg>
          <span class="wb-nav-text">全局设置</span>
        </button>

        <button
          class="wb-nav-link"
          :class="{ active: aboutDialogVisible }"
          @click="aboutDialogVisible = true"
        >
          <!-- i 字母圆圈 -->
          <span class="wb-nav-icon wb-nav-i-icon">i</span>
          <span class="wb-nav-text">关于</span>
        </button>
      </nav>

      <!-- Bottom user card -->
      <div class="wb-user-area">
        <Popover position="TR">
          <template #trigger>
            <div class="wb-user-trigger">
              <!-- 头像 -->
              <img
                v-if="user.avatar_url"
                class="wb-user-avatar"
                :src="user.avatar_url"
              />
              <div
                v-else
                class="wb-user-avatar wb-user-avatar-fallback"
              >
                {{ (user.netdisk_name || '?').charAt(0).toUpperCase() }}
              </div>
              <!-- 用户信息 -->
              <div class="wb-user-info">
                <div class="wb-user-name">{{ user.baidu_name || user.netdisk_name || '未登录' }}</div>
                <div class="wb-user-sub">
                  {{ formatBytes(user.free) }} 可用
                </div>
              </div>
              <!-- 更多 -->
              <span class="wb-user-more">···</span>
            </div>
          </template>

          <!-- Popover content -->
          <div class="wb-user-popover">
            <!-- 存储进度 -->
            <div class="wb-storage-row">
              <span class="wb-storage-label">已用 {{ formatBytes(user.used) }}</span>
              <span class="wb-storage-label">{{ formatBytes(user.total) }}</span>
            </div>
            <Progress
              :percentage="(user.used / user.total) * 100"
              class="my-6"
            />

            <div class="mt-4 text-12" style="color: var(--text-secondary)">
              登录有效期:
              {{ ((user.expireAt - Date.now()) / 1000 / 60 / 60).toFixed(1) }}小时
              <span class="text-red-500">{{ user.expireAt - Date.now() <= 0 ? '已过期' : '' }}</span>
            </div>

            <div class="mt-12 flex flex-col gap-6">
              <button class="wb-popover-btn" @click="onManageUserClick">
                <!-- 人物 SVG -->
                <svg width="14" height="14" viewBox="0 0 24 24" fill="currentColor">
                  <path d="M12 12c2.7 0 4.8-2.1 4.8-4.8S14.7 2.4 12 2.4 7.2 4.5 7.2 7.2 9.3 12 12 12zm0 2.4c-3.2 0-9.6 1.6-9.6 4.8v2.4h19.2v-2.4c0-3.2-6.4-4.8-9.6-4.8z"/>
                </svg>
                管理百度用户
              </button>
              <Button size="small" type="error" @click="removeBaiduUser">移除此百度用户</Button>
            </div>
          </div>
        </Popover>

        <!-- 底部存储进度条 -->
        <div class="wb-storage-bar-area">
          <div class="wb-storage-bar-row">
            <span>已用 {{ formatBytes(user.used) }}</span>
            <span>{{ formatBytes(user.total) }}</span>
          </div>
          <div class="wb-storage-bar-track">
            <div
              class="wb-storage-bar-fill"
              :style="`width: ${Math.min((user.used / user.total) * 100 || 0, 100)}%`"
            ></div>
          </div>
        </div>
      </div>
    </aside>

    <!-- ── Main area ── -->
    <div class="flex flex-1 flex-col" style="min-width: 0; overflow: hidden">
      <!-- Top bar (mobile: hamburger + title; desktop: hidden) -->
      <header
        class="wb-topbar"
        style="height: var(--topbar-h); background: var(--surface); border-bottom: 1px solid var(--border); display: flex; align-items: center; padding: 0 12px; gap: 10px; flex-shrink: 0"
      >
        <button
          class="wb-hamburger"
          style="flex-shrink: 0"
          @click="sidebarOpen = !sidebarOpen"
        >
          <i class="iconfont icon-menu" style="font-size: 20px"></i>
        </button>
        <span
          style="font-family: logo; font-size: 16px; font-weight: 700; color: var(--accent)"
          class="wb-topbar-title"
        >
          BaiduSync
        </span>
        <div style="flex: 1"></div>
        <button
          class="wb-hamburger"
          style="flex-shrink: 0"
          title="添加同步目录"
          @click="folderDialogVisible = true"
        >
          <i class="iconfont icon-add-folder" style="font-size: 20px"></i>
        </button>
      </header>

      <!-- Page content -->
      <main class="wb-main-content flex flex-1 flex-col" style="overflow: hidden">
        <router-view v-slot="{ Component }">
          <component
            :is="Component"
            :user-app-name="user.app_name"
            @request-add-folder="folderDialogVisible = true"
          />
        </router-view>
      </main>
    </div>

    <!-- ── Bottom nav (mobile only) ── -->
    <nav class="wb-bottom-nav">
      <button
        class="wb-bottom-nav-item"
        :class="{ active: isCurrentPath('/workbench/sync') }"
        @click="goTo(`/workbench/sync${currentSearchParams}`)"
      >
        <i class="iconfont icon-arrow-up-down wb-bottom-nav-icon"></i>
        <span>同步</span>
      </button>
      <button
        class="wb-bottom-nav-item"
        :class="{ active: isCurrentPath('/workbench/disk') && !isCurrentPath('/workbench/disk/tasks') }"
        @click="goTo(`/workbench/disk${currentSearchParams}`)"
      >
        <i class="iconfont icon-cloud wb-bottom-nav-icon"></i>
        <span>网盘</span>
      </button>
      <button
        class="wb-bottom-nav-item"
        :class="{ active: isCurrentPath('/workbench/disk/tasks') }"
        @click="goTo(`/workbench/disk/tasks${currentSearchParams}`)"
      >
        <i class="iconfont icon-download wb-bottom-nav-icon"></i>
        <span>传输</span>
      </button>
      <button
        class="wb-bottom-nav-item"
        @click="modConfigDialogVisible = true"
      >
        <i class="iconfont icon-options wb-bottom-nav-icon"></i>
        <span>设置</span>
      </button>
    </nav>
  </div>

  <!-- Modals -->
  <ModalModConfig
    v-if="modConfigDialogVisible"
    @ok="modConfigDialogVisible = false"
    @cancel="modConfigDialogVisible = false"
  />

  <ModalAbout
    v-if="aboutDialogVisible"
    @ok="aboutDialogVisible = false"
  />

  <ModalFolder
    v-if="folderDialogVisible"
    :app-name="user.app_name"
    @ok="folderDialogVisible = false"
    @cancel="folderDialogVisible = false"
  />
</template>

<script setup lang="ts">
import { httpDelUser, httpFoldersInfo, httpLogout, httpUsers } from '@src/common/api'
import ModalAbout from '@src/components/modal-about.vue'
import ModalFolder from '@src/components/modal-folder.vue'
import ModalModConfig from '@src/components/modal-mod-config.vue'
import Button from '@src/ui-components/button.vue'
import Dialog from '@src/ui-components/dialog'
import Message from '@src/ui-components/message'
import Popover from '@src/ui-components/popover.vue'
import Progress from '@src/ui-components/progress.vue'
import { type IHttpUsersRes } from 'baidu-netdisk-srv/types'
import { computed, onMounted, ref } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()

const __USER_INIT__: IHttpUsersRes['users'][number] = {
  id: '',
  expire: true,
  expireAt: 0,
  free: 0,
  total: 0,
  used: 0,
  avatar_url: '',
  baidu_name: '',
  netdisk_name: '',
  uk: 0,
  vip_type: 0,
  app_name: '',
}

const user = ref<IHttpUsersRes['users'][number]>(__USER_INIT__)
const modConfigDialogVisible = ref(false)
const aboutDialogVisible = ref(false)
const folderDialogVisible = ref(false)
const sidebarOpen = ref(false)
const syncTaskCount = ref(0)

const currentSearchParams = computed(() => window.location.search || '')

const isCurrentPath = (path: string) => window.location.pathname.startsWith(path)

function goTo(path: string) {
  router.push(path)
}

onMounted(() => {
  getUser()
  pollSyncCount()
})

async function getUser() {
  try {
    const searchParams = new URLSearchParams(window.location.search)
    const userId = Object.fromEntries(searchParams.entries()).id
    if (userId) {
      const users = (await httpUsers({ id: userId })).users
      user.value = users.find(item => item.id === userId) || __USER_INIT__
    } else {
      const users = (await httpUsers()).users
      user.value = users[0] || __USER_INIT__
    }
  } catch (inErr) {
    Message.error(`获取用户失败: ${(inErr as Error).message}`)
  }
}

// 轮询同步任务数量（用于 badge）
async function pollSyncCount() {
  try {
    const info = await httpFoldersInfo()
    let count = 0
    for (const f of info.folders) {
      count += (f.uploadTasks?.length || 0) + (f.downloadTasks?.length || 0)
    }
    syncTaskCount.value = count
  } catch {
    // 静默失败
  }
  setTimeout(pollSyncCount, 2000)
}

function onManageUserClick() {
  location.href = '/pick-user'
}

async function logout() {
  if (!(await Dialog.confirm({ title: '退出登录', okText: '登出' }))) return
  try {
    await httpLogout()
    location.href = '/login'
  } catch (inErr) {
    Message.error(`登出失败: ${(inErr as Error).message}`)
  }
}

async function removeBaiduUser() {
  if (
    !(await Dialog.confirm({
      title: '移除百度用户',
      content: user.value.netdisk_name,
      okText: '移除',
      okType: 'error',
    }))
  )
    return

  try {
    await httpDelUser({ id: user.value.id })
    location.href = '/pick-user'
  } catch (inErr) {
    Message.error(`移除用户失败: ${(inErr as Error).message}`)
  }
}

function formatBytes(bytes: number): string {
  if (!bytes) return '0 B'
  if (bytes >= 1024 ** 4) return (bytes / 1024 ** 4).toFixed(2) + ' TB'
  if (bytes >= 1024 ** 3) return (bytes / 1024 ** 3).toFixed(2) + ' GB'
  if (bytes >= 1024 ** 2) return (bytes / 1024 ** 2).toFixed(2) + ' MB'
  if (bytes >= 1024) return (bytes / 1024).toFixed(1) + ' KB'
  return bytes + ' B'
}
</script>

<style scoped>
/* ── Logo ── */
.wb-logo-area {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 18px 16px 14px;
  border-bottom: 1px solid var(--border);
}
.wb-logo-icon {
  width: 36px;
  height: 36px;
  border-radius: 10px;
  background: linear-gradient(135deg, #4f6ef7, #a78bfa);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}
.wb-logo-text {
  font-family: logo;
  font-size: 17px;
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: 0.3px;
}

/* ── Nav ── */
.wb-nav {
  padding: 12px 10px 0;
  display: flex;
  flex-direction: column;
  gap: 2px;
}
.wb-nav-group-label {
  font-size: 11px;
  font-weight: 600;
  color: var(--text-muted);
  letter-spacing: 0.08em;
  text-transform: uppercase;
  padding: 4px 10px 6px;
}
.wb-nav-link {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 9px 12px;
  border-radius: var(--radius-md);
  font-size: 14px;
  color: var(--text-secondary);
  text-decoration: none;
  transition: background 0.15s, color 0.15s;
  border: none;
  background: transparent;
  cursor: pointer;
  width: 100%;
  text-align: left;
}
.wb-nav-link:hover {
  background: var(--accent-light);
  color: var(--accent);
}
.wb-nav-link.active {
  background: var(--accent-light);
  color: var(--accent);
  font-weight: 600;
}
.wb-nav-icon {
  font-size: 18px;
  flex-shrink: 0;
  width: 18px;
  height: 18px;
}
.wb-nav-i-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  border: 1.5px solid currentColor;
  font-size: 12px;
  font-style: italic;
  font-weight: 700;
  font-family: serif;
  flex-shrink: 0;
}
.wb-nav-text {
  flex: 1;
}
.wb-nav-badge {
  background: var(--accent);
  color: #fff;
  font-size: 11px;
  font-weight: 700;
  border-radius: 100px;
  padding: 1px 7px;
  min-width: 20px;
  text-align: center;
  flex-shrink: 0;
}

/* ── User area ── */
.wb-user-area {
  border-top: 1px solid var(--border);
  padding: 14px 12px 12px;
}
.wb-user-trigger {
  display: flex;
  align-items: center;
  gap: 10px;
  cursor: pointer;
  border-radius: var(--radius-md);
  padding: 4px 6px;
  transition: background 0.15s;
}
.wb-user-trigger:hover {
  background: var(--surface-2);
}
.wb-user-avatar {
  width: 38px;
  height: 38px;
  border-radius: 50%;
  object-fit: cover;
  flex-shrink: 0;
}
.wb-user-avatar-fallback {
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #4f6ef7, #a78bfa);
  color: #fff;
  font-size: 16px;
  font-weight: 700;
}
.wb-user-info {
  flex: 1;
  min-width: 0;
}
.wb-user-name {
  font-size: 13px;
  font-weight: 600;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.wb-user-sub {
  font-size: 11px;
  color: var(--text-muted);
  margin-top: 1px;
}
.wb-user-more {
  font-size: 16px;
  color: var(--text-muted);
  letter-spacing: 2px;
  flex-shrink: 0;
}

/* ── Storage bar ── */
.wb-storage-bar-area {
  margin-top: 10px;
}
.wb-storage-bar-row {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
  color: var(--text-muted);
  margin-bottom: 5px;
}
.wb-storage-bar-track {
  height: 5px;
  border-radius: 3px;
  background: var(--surface-2);
  overflow: hidden;
}
.wb-storage-bar-fill {
  height: 100%;
  border-radius: 3px;
  background: linear-gradient(90deg, #4f6ef7, #a78bfa);
  transition: width 0.4s;
}

/* ── User popover ── */
.wb-user-popover {
  min-width: 220px;
  padding: 16px;
  background: var(--surface);
  border-radius: var(--radius-md);
}
.wb-storage-row {
  display: flex;
  justify-content: space-between;
  font-size: 12px;
  color: var(--text-secondary);
}
.wb-storage-label {
  font-size: 12px;
  color: var(--text-secondary);
}
.wb-popover-btn {
  display: flex;
  align-items: center;
  gap: 6px;
  width: 100%;
  padding: 7px 10px;
  font-size: 13px;
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  background: transparent;
  color: var(--text-primary);
  cursor: pointer;
  transition: background 0.15s;
}
.wb-popover-btn:hover {
  background: var(--surface-2);
}

/* ── Desktop: hide topbar ── */
@media (min-width: 601px) {
  .wb-topbar {
    display: none !important;
  }
}
</style>
