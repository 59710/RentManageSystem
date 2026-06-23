<template>
  <div class="login-page">
    <!-- 装饰背景 -->
    <div class="login-bg">
      <div class="bg-circle circle-1"></div>
      <div class="bg-circle circle-2"></div>
      <div class="bg-circle circle-3"></div>
    </div>

    <div class="login-container fade-in-up">
      <!-- 左侧品牌展示 -->
      <div class="login-brand">
        <div class="brand-content">
          <span class="brand-icon">🏠</span>
          <h1 class="brand-title">青年品质租房</h1>
          <p class="brand-desc">为年轻人打造的高品质居住体验</p>
          <div class="brand-features">
            <div class="feature-item">
              <el-icon color="#ff6b35"><CircleCheck /></el-icon>
              <span>真实房源</span>
            </div>
            <div class="feature-item">
              <el-icon color="#ff6b35"><CircleCheck /></el-icon>
              <span>品质保障</span>
            </div>
            <div class="feature-item">
              <el-icon color="#ff6b35"><CircleCheck /></el-icon>
              <span>便捷管理</span>
            </div>
          </div>
        </div>
      </div>

      <!-- 右侧登录表单 -->
      <div class="login-form-wrapper">
        <div class="form-header">
          <h2>欢迎回来</h2>
          <p>登录您的账号，开启品质生活</p>
        </div>

        <el-form
          ref="formRef"
          :model="form"
          :rules="rules"
          size="large"
          @submit.prevent="handleLogin"
        >
          <el-form-item prop="phone">
            <el-input
              v-model="form.phone"
              placeholder="请输入手机号"
              :prefix-icon="Iphone"
              maxlength="11"
            />
          </el-form-item>

          <el-form-item prop="password">
            <el-input
              v-model="form.password"
              type="password"
              placeholder="请输入密码"
              :prefix-icon="Lock"
              show-password
              @keyup.enter="handleLogin"
            />
          </el-form-item>

          <el-button
            type="primary"
            :loading="loading"
            class="login-btn"
            @click="handleLogin"
          >
            {{ loading ? '登录中...' : '登 录' }}
          </el-button>
        </el-form>

        <div class="form-footer">
          <span>还没有账号？</span>
          <router-link to="/register" class="link-primary">
            立即注册 →
          </router-link>
        </div>

        <div class="guest-entry">
          <el-divider>
            <span class="divider-text">或者</span>
          </el-divider>
          <router-link to="/home" class="guest-link">
            <el-icon><View /></el-icon>
            以游客身份浏览房源
          </router-link>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import { useUserStore } from '@/stores/user'
import { Iphone, Lock, CircleCheck, View } from '@element-plus/icons-vue'
import type { FormInstance, FormRules } from 'element-plus'
import { ElMessage } from 'element-plus'

const router = useRouter()
const route = useRoute()
const userStore = useUserStore()

const formRef = ref<FormInstance>()
const loading = ref(false)

const form = reactive({
  phone: '',
  password: '',
})

const rules: FormRules = {
  phone: [
    { required: true, message: '请输入手机号', trigger: 'blur' },
    {
      pattern: /^1[3-9]\d{9}$/,
      message: '手机号格式不正确',
      trigger: 'blur',
    },
  ],
  password: [
    { required: true, message: '请输入密码', trigger: 'blur' },
    { min: 6, max: 20, message: '密码长度为6-20位', trigger: 'blur' },
  ],
}

async function handleLogin() {
  if (!formRef.value) return

  await formRef.value.validate(async (valid) => {
    if (!valid) return

    loading.value = true
    try {
      await userStore.login(form.phone, form.password)
      ElMessage.success('登录成功！欢迎回来 🎉')

      // 跳转到之前要访问的页面或首页
      const redirect = (route.query.redirect as string) || '/home'
      router.push(redirect)
    } catch (err: any) {
      // 错误已在拦截器中处理
    } finally {
      loading.value = false
    }
  })
}
</script>

<style lang="scss" scoped>
.login-page {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  position: relative;
  overflow: hidden;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

// 背景装饰圆圈
.login-bg {
  position: absolute;
  inset: 0;
  pointer-events: none;

  .bg-circle {
    position: absolute;
    border-radius: 50%;
    background: rgba(255, 255, 255, 0.08);
  }

  .circle-1 {
    width: 400px;
    height: 400px;
    top: -100px;
    right: -100px;
    animation: float 8s ease-in-out infinite;
  }

  .circle-2 {
    width: 300px;
    height: 300px;
    bottom: -80px;
    left: -60px;
    animation: float 10s ease-in-out infinite reverse;
  }

  .circle-3 {
    width: 150px;
    height: 150px;
    top: 40%;
    left: 20%;
    animation: float 6s ease-in-out infinite 1s;
  }
}

@keyframes float {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-30px); }
}

// 登录容器（卡片）
.login-container {
  display: flex;
  width: 900px;
  max-width: 95vw;
  min-height: 520px;
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(40px);
  border-radius: var(--radius-xl);
  box-shadow: var(--shadow-lg), 0 25px 80px rgba(0, 0, 0, 0.15);
  overflow: hidden;
  position: relative;
  z-index: 1;
}

// 左侧品牌区
.login-brand {
  flex: 1;
  background: linear-gradient(135deg, #ff6b35 0%, #f7931e 100%);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 48px;
  position: relative;
  overflow: hidden;

  &::after {
    content: '';
    position: absolute;
    top: -50%;
    right: -50%;
    width: 200%;
    height: 200%;
    background: radial-gradient(circle, rgba(255, 255, 255, 0.08) 0%, transparent 60%);
    pointer-events: none;
  }

  .brand-content {
    text-align: center;
    color: white;
    position: relative;
    z-index: 1;
  }

  .brand-icon {
    font-size: 64px;
    display: block;
    margin-bottom: 16px;
    filter: drop-shadow(0 4px 12px rgba(0, 0, 0, 0.2));
  }

  .brand-title {
    font-size: 32px;
    font-weight: 800;
    margin-bottom: 8px;
    letter-spacing: 2px;
  }

  .brand-desc {
    font-size: 15px;
    opacity: 0.9;
    margin-bottom: 36px;
  }

  .brand-features {
    display: flex;
    flex-direction: column;
    gap: 14px;
    align-items: flex-start;
  }

  .feature-item {
    display: flex;
    align-items: center;
    gap: 10px;
    font-size: 15px;
    font-weight: 500;
    padding: 8px 16px;
    background: rgba(255, 255, 255, 0.15);
    border-radius: 9999px;
    backdrop-filter: blur(4px);

    .el-icon {
      font-size: 18px;
    }
  }
}

// 右侧表单区
.login-form-wrapper {
  flex: 1;
  padding: 56px 48px;
  display: flex;
  flex-direction: column;
  justify-content: center;

  .form-header {
    margin-bottom: 32px;

    h2 {
      font-size: 28px;
      font-weight: 700;
      color: var(--color-text);
      margin-bottom: 8px;
    }

    p {
      color: var(--color-text-secondary);
      font-size: 15px;
    }
  }
}

// 登录按钮
.login-btn {
  width: 100%;
  height: 48px !important;
  font-size: 16px !important;
  font-weight: 600 !important;
  border-radius: var(--radius-sm) !important;
  background: linear-gradient(135deg, var(--color-primary), #ff9a56) !important;
  border: none !important;
  letter-spacing: 4px;
  transition: all var(--transition-normal) !important;

  &:hover:not(:disabled) {
    transform: translateY(-2px);
    box-shadow: var(--shadow-primary) !important;
  }

  &:active:not(:disabled) {
    transform: translateY(0);
  }
}

// 底部链接
.form-footer {
  text-align: center;
  margin-top: 24px;
  color: var(--color-text-secondary);
  font-size: 14px;

  .link-primary {
    color: var(--color-primary);
    font-weight: 600;
    margin-left: 4px;

    &:hover {
      text-decoration: underline;
    }
  }
}

// 游客浏览入口
.guest-entry {
  text-align: center;
  margin-top: 8px;

  .divider-text {
    color: var(--color-text-muted);
    font-size: 13px;
  }

  .guest-link {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 10px 24px;
    margin-top: 8px;
    color: var(--color-text-secondary);
    font-size: 14px;
    border: 1px dashed var(--color-border);
    border-radius: var(--radius-sm);
    transition: all var(--transition-fast);

    &:hover {
      color: var(--color-primary);
      border-color: var(--color-primary);
      background: rgba(255, 107, 53, 0.04);
    }
  }
}
</style>
