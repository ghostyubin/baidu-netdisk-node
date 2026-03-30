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
      <div
        class="flex items-center gap-10 px-16 py-14"
        style="border-bottom: 1px solid var(--border)"
      >
        <span
          style="font-family: logo; font-size: 18px; font-weight: 700; color: var(--accent); letter-spacing: 0.5px"
        >
          BAIDU SYNC
        </span>
      </div>

      <!-- Nav links -->
      <nav class="flex flex-col gap-4 px-10 py-12" style="flex: 1; overflow-y: auto">
        <router-link
          :to="`/workbench/sync${currentSearchParams}`"
          class="wb-nav-link"
          :class="{ active: isCurrentPath('/workbench/sync') }"
          @click="sidebarOpen = false"
        >
          <i class="iconfont icon-arrow-up-down wb-nav-icon"></i>
          <span>同步</span>
        </router-link>

        <router-link
          :to="`/workbench/disk${currentSearchParams}`"
          class="wb-nav-link"
          :class="{ active: isCurrentPath('/workbench/disk') }"
          @click="sidebarOpen = false"
        >
          <i class="iconfont icon-cloud wb-nav-icon"></i>
          <span>网盘</span>
        </router-link>
      </nav>

      <!-- User card (bottom of sidebar) -->
      <div
        class="px-12 py-12"
        style="border-top: 1px solid var(--border)"
      >
        <Popover position="TR">
          <template #trigger>
            <div
              class="wb-user-card"
              style="cursor: pointer; border-radius: var(--radius-md); padding: 8px 10px; display: flex; align-items: center; gap: 10px; transition: background 0.15s"
              @mouseenter="e => (e.currentTarget.style.background = 'var(--surface-2)')"
              @mouseleave="e => (e.currentTarget.style.background = '')"
            >
              <!-- 有头像时显示图片，无头像时显示渐变圆+首字母 -->
              <img
                v-if="user.avatar_url"
                class="h-32 w-32 rounded-full"
                style="flex-shrink: 0; object-fit: cover"
                :src="user.avatar_url"
              />
              <div
                v-else
                style="
                  width: 32px; height: 32px; border-radius: 50%; flex-shrink: 0;
                  background: linear-gradient(135deg, #4f6ef7, #a78bfa);
                  display: flex; align-items: center; justify-content: center;
                  font-size: 14px; color: white; font-weight: 700;
                "
              >
                {{ (user.netdisk_name || '?').charAt(0).toUpperCase() }}
              </div>
              <div style="flex: 1; min-width: 0">
                <div
                  style="font-size: 13px; font-weight: 600; overflow: hidden; text-overflow: ellipsis; white-space: nowrap"
                >
                  {{ user.netdisk_name || '未登录' }}
                </div>
                <div style="font-size: 11px; color: var(--text-muted)">
                  {{ ((user.used / user.total) * 100 || 0).toFixed(1) }}% 已用
                </div>
              </div>
            </div>
          </template>

          <!-- User popover content -->
          <div class="flex flex-col bg-white p-16" style="min-width: 220px">
            <Progress
              :percentage="(user.used / user.total) * 100"
              class="my-8"
            />
            <div class="flex items-center justify-between text-13">
              <span style="color: var(--text-secondary)">
                {{ (user.used / 1024 / 1024 / 1024).toFixed(2) }} GB
                <span style="color: var(--accent)">
                  ({{ ((user.used / user.total) * 100).toFixed(2) }}%)
                </span>
              </span>
              <span style="color: var(--text-muted)">
                {{ (user.total / 1024 / 1024 / 1024).toFixed(2) }} GB
              </span>
            </div>
            <div class="mt-4 text-12" style="color: var(--text-secondary)">
              登录有效期:
              {{ ((user.expireAt - Date.now()) / 1000 / 60 / 60).toFixed(1) }}小时
              <span class="text-red-500">
                {{ user.expireAt - Date.now() <= 0 ? '已过期' : '' }}
              </span>
            </div>
            <div class="mt-12 flex items-center justify-end">
              <Button
                size="small"
                type="error"
                @click="removeBaiduUser"
              >
                移除此百度用户
              </Button>
            </div>
          </div>
        </Popover>

        <!-- Action buttons row -->
        <div class="mt-8 flex items-center gap-4">
          <button
            class="wb-sidebar-action-btn"
            title="添加同步目录"
            @click="folderDialogVisible = true"
          >
            <i class="iconfont icon-add-folder" style="font-size: 16px"></i>
          </button>
          <button
            class="wb-sidebar-action-btn"
            title="管理百度用户"
            @click="onManageUserClick"
          >
            <i class="iconfont icon-options" style="font-size: 16px"></i>
          </button>
          <button
            class="wb-sidebar-action-btn"
            title="全局设置"
            @click="modConfigDialogVisible = true"
          >
            <i class="iconfont icon-options" style="font-size: 14px; transform: rotate(90deg)"></i>
          </button>
          <button
            class="wb-sidebar-action-btn"
            title="关于"
            @click="aboutDialogVisible = true"
          >
            <i class="iconfont icon-info" style="font-size: 16px"></i>
          </button>
          <button
            class="wb-sidebar-action-btn"
            title="退出登录"
            @click="logout"
          >
            <i class="iconfont icon-close" style="font-size: 16px"></i>
          </button>
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
          BAIDU SYNC
        </span>
        <!-- Right: add folder shortcut on mobile -->
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
        <router-view></router-view>
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
import { httpDelUser, httpLogout, httpUsers } from '@src/common/api'
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

const currentSearchParams = computed(() => window.location.search || '')

const isCurrentPath = (path: string) => window.location.pathname.startsWith(path)

function goTo(path: string) {
  router.push(path)
}

onMounted(() => {
  getUser()
})

async function getUser() {
  try {
    const searchParams = new URLSearchParams(window.location.search)
    const userId = Object.fromEntries(searchParams.entries()).id
    const users = (await httpUsers({ id: userId })).users
    user.value = users.find(item => item.id === userId) || __USER_INIT__
  } catch (inErr) {
    Message.error(`获取用户失败: ${(inErr as Error).message}`)
  }
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
</script>

<style scoped>
/* Nav link in sidebar */
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
}

/* Sidebar action buttons */
.wb-sidebar-action-btn {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 6px 0;
  border: none;
  background: transparent;
  border-radius: var(--radius-sm);
  color: var(--text-secondary);
  cursor: pointer;
  transition: background 0.15s, color 0.15s;
}
.wb-sidebar-action-btn:hover {
  background: var(--surface-2);
  color: var(--accent);
}

/* Desktop: hide topbar entirely */
@media (min-width: 601px) {
  .wb-topbar {
    display: none !important;
  }
}
</style>
