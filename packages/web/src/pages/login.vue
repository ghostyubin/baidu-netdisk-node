<template>
  <div
    class="flow-bg absolute bottom-0 left-0 right-0 top-0 flex flex-col items-center justify-center"
    style="padding: 24px"
  >
    <!-- Logo -->
    <div
      class="mb-32 text-center font-bold text-white"
      style="font-family: logo; font-size: clamp(26px, 6vw, 36px)"
    >
      BAIDU SYNC
    </div>

    <!-- Login card -->
    <div class="login-card">
      <input
        v-model="username"
        class="login-input"
        placeholder="管理员用户名"
        auto-complete="on"
        autofocus
        @keyup.enter="onLoginClick"
      />

      <input
        v-model="password"
        class="login-input"
        placeholder="管理员密码"
        auto-complete="on"
        type="password"
        @keyup.enter="onLoginClick"
      />

      <button
        class="login-btn"
        :disabled="loading"
        @click="onLoginClick"
      >
        {{ loading ? '登录中…' : '登录' }}
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { httpLogin } from '@src/common/api'
import Message from '@src/ui-components/message'
import { ref } from 'vue'

const loading = ref(false)
const username = ref('')
const password = ref('')

const onLoginClick = async () => {
  if (loading.value) return
  if (!username.value || !password.value) return

  loading.value = true

  try {
    await httpLogin({ username: username.value, password: password.value })
    location.href = '/'
  } catch (inErr) {
    Message.error(`登录失败: ${(inErr as Error).message}`)
  } finally {
    loading.value = false
  }
}
</script>

<style scoped>
.login-card {
  display: flex;
  flex-direction: column;
  align-items: stretch;
  gap: 16px;
  width: min(360px, 90vw);
  background: rgba(255, 255, 255, 0.18);
  backdrop-filter: blur(16px);
  -webkit-backdrop-filter: blur(16px);
  border: 1px solid rgba(255, 255, 255, 0.35);
  border-radius: 20px;
  padding: 32px 28px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.18);
}

.login-input {
  width: 100%;
  padding: 10px 18px;
  border-radius: 999px;
  border: 1px solid rgba(255, 255, 255, 0.5);
  background: rgba(255, 255, 255, 0.65);
  font-size: 14px;
  outline: none;
  color: #1e293b;
  transition: border-color 0.2s, background 0.2s;
}
.login-input::placeholder {
  color: #94a3b8;
}
.login-input:focus {
  border-color: var(--accent);
  background: rgba(255, 255, 255, 0.85);
}

.login-btn {
  width: 100%;
  padding: 10px 0;
  border-radius: 999px;
  background: var(--accent);
  color: #fff;
  font-size: 15px;
  font-weight: 600;
  border: none;
  cursor: pointer;
  transition: background 0.15s, opacity 0.15s;
}
.login-btn:hover:not(:disabled) {
  background: var(--accent-hover);
}
.login-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}
</style>
