<template>
  <div class="register-page">
    <div class="register-bg">
      <div class="bg-circle circle-1"></div>
      <div class="bg-circle circle-2"></div>
    </div>

    <div class="register-container fade-in-up">
      <!-- 左侧装饰 -->
      <div class="register-brand">
        <span class="brand-icon">✨</span>
        <h2>加入我们</h2>
        <p>开启品质租房新体验</p>
      </div>

      <!-- 注册表单 -->
      <div class="form-wrapper">
        <h3>创建账号</h3>

        <el-form
          ref="formRef"
          :model="form"
          :rules="rules"
          size="large"
          label-position="top"
          @submit.prevent="handleRegister"
        >
          <el-form-item prop="nickname" label="昵称">
            <el-input
              v-model="form.nickname"
              placeholder="请输入您的昵称"
              :prefix-icon="User"
            />
          </el-form-item>

          <el-form-item prop="realName" label="真实姓名">
            <el-input
              v-model="form.realName"
              placeholder="请输入真实姓名（选填）"
              :prefix-icon="User"
            />
          </el-form-item>

          <el-form-item prop="email" label="邮箱">
            <el-input
              v-model="form.email"
              placeholder="请输入邮箱（选填）"
              :prefix-icon="Message"
            />
          </el-form-item>

          <el-form-item prop="gender" label="性别">
            <el-radio-group v-model="form.gender" size="large">
              <el-radio-button :value="0">未知</el-radio-button>
              <el-radio-button :value="1">男</el-radio-button>
              <el-radio-button :value="2">女</el-radio-button>
            </el-radio-group>
          </el-form-item>

          <el-form-item prop="phone" label="手机号">
            <el-input
              v-model="form.phone"
              placeholder="请输入手机号"
              :prefix-icon="Iphone"
              maxlength="11"
              @blur="checkPhone"
              @input="checkPhone"
            />
            <transition name="el-fade-in">
              <span v-if="phoneStatus === 'checking'" class="phone-hint checking">
                <el-icon class="is-loading"><Loading /></el-icon> 正在检查...
              </span>
              <span v-else-if="phoneStatus === 'available'" class="phone-hint available">
                <el-icon><CircleCheckFilled /></el-icon> 手机号可用
              </span>
              <span v-else-if="phoneStatus === 'exists'" class="phone-hint exists">
                <el-icon><WarningFilled /></el-icon> 该手机号已注册，
                <router-link to="/login" class="link-login">去登录 →</router-link>
              </span>
            </transition>
          </el-form-item>

          <el-form-item prop="password" label="密码">
            <el-input
              v-model="form.password"
              type="password"
              placeholder="8-20位，包含字母和数字"
              :prefix-icon="Lock"
              show-password
            />
          </el-form-item>

          <el-form-item prop="confirmPassword" label="确认密码">
            <el-input
              v-model="form.confirmPassword"
              type="password"
              placeholder="再次输入密码"
              :prefix-icon="Lock"
              show-password
              @keyup.enter="handleRegister"
            />
          </el-form-item>

          <el-form-item prop="role" label="注册身份">
            <el-radio-group v-model="form.role" size="large">
              <el-radio-button :value="0">我是租客 🧑‍💼</el-radio-button>
              <el-radio-button :value="1">我是房东 👨‍💼</el-radio-button>
            </el-radio-group>
          </el-form-item>

          <el-button
            type="primary"
            :loading="loading"
            class="submit-btn"
            @click="handleRegister"
          >
            {{ loading ? '注册中...' : '注 册' }}
          </el-button>
        </el-form>

        <div class="form-footer">
          <span>已有账号？</span>
          <router-link to="/login" class="link-primary">
            ← 返回登录
          </router-link>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive } from 'vue'
import { useRouter } from 'vue-router'
import { useUserStore } from '@/stores/user'
import { User, Iphone, Lock, Loading, CircleCheckFilled, WarningFilled, Message } from '@element-plus/icons-vue'
import type { FormInstance, FormRules } from 'element-plus'
import { ElMessage } from 'element-plus'
import request from '@/utils/request'

const router = useRouter()
const userStore = useUserStore()

const formRef = ref<FormInstance>()
const loading = ref(false)

// 手机号检查状态：'' | 'checking' | 'exists' | 'available'
const phoneStatus = ref('')
let checkTimer: ReturnType<typeof setTimeout> | null = null

const form = reactive({
  nickname: '',
  realName: '',
  email: '',
  gender: 0,
  phone: '',
  password: '',
  confirmPassword: '',
  role: 0,
})

async function checkPhone() {
  const phone = form.phone.trim()

  // 格式不对时清除状态
  if (!/^1[3-9]\d{9}$/.test(phone)) {
    phoneStatus.value = ''
    if (checkTimer) clearTimeout(checkTimer)
    return
  }

  // 防抖 500ms
  if (checkTimer) clearTimeout(checkTimer)
  phoneStatus.value = 'checking'
  checkTimer = setTimeout(async () => {
    try {
      const exists = await request.get('/auth/check-phone', { params: { phone } })
      phoneStatus.value = exists ? 'exists' : 'available'
    } catch {
      phoneStatus.value = ''
    }
  }, 500)
}

// 确认密码校验
const confirmPassValidator = (_rule: any, value: string, callback: any) => {
  if (value !== form.password) {
    callback(new Error('两次输入的密码不一致'))
  } else {
    callback()
  }
}

// 手机号已注册校验
const phoneExistsValidator = (_rule: any, _value: string, callback: any) => {
  if (phoneStatus.value === 'exists') {
    callback(new Error('该手机号已注册'))
  } else {
    // checking / available / 空白 → 不阻止，下方 hint 已展示状态
    callback()
  }
}

const rules: FormRules = {
  nickname: [
    { required: true, message: '请输入昵称', trigger: 'blur' },
    { min: 2, max: 50, message: '昵称长度为2-50位', trigger: 'blur' },
  ],
  email: [
    {
      type: 'email',
      message: '邮箱格式不正确',
      trigger: 'blur',
    },
  ],
  phone: [
    { required: true, message: '请输入手机号', trigger: 'blur' },
    {
      pattern: /^1[3-9]\d{9}$/,
      message: '手机号格式不正确',
      trigger: 'blur',
    },
    { validator: phoneExistsValidator, trigger: 'blur' },
  ],
  password: [
    { required: true, message: '请输入密码', trigger: 'blur' },
    { min: 8, max: 20, message: '密码长度为8-20位', trigger: 'blur' },
    {
      pattern: /^(?=.*[A-Za-z])(?=.*\d).+$/,
      message: '密码必须包含字母和数字',
      trigger: 'blur',
    },
  ],
  confirmPassword: [
    { required: true, message: '请确认密码', trigger: 'blur' },
    { validator: confirmPassValidator, trigger: 'blur' },
  ],
}

async function handleRegister() {
  if (!formRef.value) return

  // 手机号仍在检查中或已注册，阻止提交
  if (phoneStatus.value === 'checking') {
    ElMessage.warning('正在检查手机号，请稍候再试')
    return
  }
  if (phoneStatus.value === 'exists') {
    ElMessage.warning('该手机号已注册，请更换手机号')
    return
  }

  await formRef.value.validate(async (valid) => {
    if (!valid) return

    loading.value = true
    try {
      await userStore.register({
        nickname: form.nickname,
        realName: form.realName || undefined,
        email: form.email || undefined,
        gender: form.gender,
        phone: form.phone,
        password: form.password,
        role: form.role,
      })
      ElMessage.success('注册成功！欢迎加入青年品质租房 🎉')
      router.push({ name: 'Login', query: { phone: form.phone } })
    } catch (err) {
      // 错误已在拦截器中处理
    } finally {
      loading.value = false
    }
  })
}
</script>

<style lang="scss" scoped>
.register-page {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  position: relative;
  overflow: hidden;
  background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%);
}

.register-bg {
  position: absolute;
  inset: 0;
  pointer-events: none;

  .bg-circle {
    position: absolute;
    border-radius: 50%;
    background: rgba(255, 255, 255, 0.08);
  }

  .circle-1 {
    width: 500px;
    height: 500px;
    top: -150px;
    left: -100px;
    animation: float 12s ease-in-out infinite;
  }

  .circle-2 {
    width: 250px;
    height: 250px;
    bottom: -60px;
    right: -40px;
    animation: float 9s ease-in-out infinite reverse;
  }
}

@keyframes float {
  0%, 100% { transform: translateY(0) rotate(0); }
  50% { transform: translateY(-25px) rotate(5deg); }
}

.register-container {
  display: flex;
  width: 780px;
  max-width: 95vw;
  background: rgba(255, 255, 255, 0.96);
  backdrop-filter: blur(40px);
  border-radius: var(--radius-xl);
  box-shadow: var(--shadow-lg), 0 25px 80px rgba(0, 0, 0, 0.15);
  overflow: hidden;
  position: relative;
  z-index: 1;
}

.register-brand {
  width: 280px;
  padding: 48px 32px;
  background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: white;

  .brand-icon {
    font-size: 56px;
    margin-bottom: 16px;
  }

  h2 {
    font-size: 26px;
    font-weight: 800;
    margin-bottom: 8px;
  }

  p {
    font-size: 14px;
    opacity: 0.9;
    text-align: center;
  }
}

.form-wrapper {
  flex: 1;
  padding: 48px;
  overflow-y: auto;

  h3 {
    font-size: 24px;
    font-weight: 700;
    margin-bottom: 24px;
    color: var(--color-text);
  }
}

.submit-btn {
  width: 100%;
  height: 46px !important;
  font-size: 16px !important;
  font-weight: 600 !important;
  border-radius: var(--radius-sm) !important;
  letter-spacing: 4px;
  transition: all var(--transition-normal) !important;
  margin-top: 8px;

  &:hover:not(:disabled) {
    transform: translateY(-2px);
    box-shadow: 0 6px 24px rgba(79, 172, 254, 0.35) !important;
  }
}

.phone-hint {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  margin-top: 4px;

  .el-icon {
    font-size: 14px;
  }

  &.checking {
    color: #909399;
  }

  &.available {
    color: #67c23a;
  }

  &.exists {
    color: #f56c6c;
  }
}

.link-login {
  color: var(--color-primary, #409eff);
  font-weight: 600;
  text-decoration: underline;
}

.form-footer {
  text-align: center;
  margin-top: 20px;
  color: var(--color-text-secondary);

  .link-primary {
    color: var(--color-info);
    font-weight: 600;
    margin-left: 4px;
  }
}
</style>
