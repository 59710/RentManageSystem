<template>
  <div class="main-layout">
    <!-- 顶部导航栏 -->
    <header class="navbar glass-effect" :class="{ 'scrolled': isScrolled }">
      <div class="navbar-inner">
        <!-- Logo -->
        <router-link to="/" class="logo">
          <span class="logo-icon">🏠</span>
          <span class="logo-text gradient-text">青年品质租房</span>
        </router-link>

        <!-- 导航菜单 -->
        <nav class="nav-menu">
          <router-link to="/home" class="nav-item" active-class="active">
            <el-icon><HomeFilled /></el-icon>
            <span>首页</span>
          </router-link>
          <router-link to="/house" class="nav-item" active-class="active">
            <el-icon><OfficeBuilding /></el-icon>
            <span>找房</span>
          </router-link>
          <!-- 我的订单（仅租客可见） -->
          <router-link v-if="userStore.isLoggedIn && !userStore.isLandlord && !userStore.isAdmin" to="/tenant/orders" class="nav-item nav-tenant" active-class="active">
            <el-icon><Tickets /></el-icon>
            <span>我的订单</span>
          </router-link>
          <!-- 房东专属菜单（仅房东可见） -->
          <template v-if="userStore.isLandlord">
            <router-link to="/landlord/publish" class="nav-item nav-landlord" active-class="active">
              <el-icon><Plus /></el-icon>
              <span>发布房源</span>
            </router-link>
            <router-link to="/landlord/houses" class="nav-item nav-landlord" active-class="active">
              <el-icon><List /></el-icon>
              <span>我的房源</span>
            </router-link>
            <router-link to="/landlord/orders" class="nav-item nav-landlord" active-class="active">
              <el-icon><Bell /></el-icon>
              <span>订单管理</span>
              <!-- 待确认订单角标 -->
              <el-badge v-if="landlordPendingCount > 0" :value="landlordPendingCount" :max="99" class="nav-badge" />
            </router-link>
          </template>
          <!-- 管理员专属菜单 -->
          <template v-if="userStore.isAdmin">
            <router-link to="/admin/audit" class="nav-item nav-admin" active-class="active">
              <el-icon><Stamp /></el-icon>
              <span>审核管理</span>
              <el-badge v-if="pendingAuditCount > 0" :value="pendingAuditCount" :max="99" class="nav-badge" />
            </router-link>
          </template>
        </nav>

        <!-- 右侧用户区 -->
        <div class="nav-right">
          <template v-if="userStore.isLoggedIn">
            <!-- 搜索框 -->
            <el-input
              v-model="searchKeyword"
              placeholder="搜索房源..."
              :prefix-icon="Search"
              clearable
              class="nav-search"
              @keyup.enter="handleSearch"
            />

            <!-- 用户头像 + 下拉 -->
            <el-dropdown trigger="click" @command="handleCommand">
              <div class="user-avatar">
                <el-avatar :size="36" :style="{ backgroundColor: avatarColor }">
                  {{ userStore.userInfo?.nickname?.charAt(0) || '?' }}
                </el-avatar>
                <span class="user-name">{{ userStore.userInfo?.nickname || '用户' }}</span>
                <el-icon class="arrow"><ArrowDown /></el-icon>
              </div>
              <template #dropdown>
                <el-dropdown-menu>
                  <el-dropdown-item disabled>
                    <el-icon><User /></el-icon>
                    {{ userStore.roleName }}
                  </el-dropdown-item>
                  <el-dropdown-item divided command="profile">
                    <el-icon><Setting /></el-icon>
                    个人设置
                  </el-dropdown-item>
                  <el-dropdown-item command="logout" style="color: var(--color-danger)">
                    <el-icon><SwitchButton /></el-icon>
                    退出登录
                  </el-dropdown-item>
                </el-dropdown-menu>
              </template>
            </el-dropdown>
          </template>

          <template v-else>
            <el-button type="primary" round @click="$router.push('/login')">
              登录 / 注册
            </el-button>
          </template>
        </div>
      </div>
    </header>

    <!-- 主内容区 -->
    <main class="main-content">
      <router-view v-slot="{ Component }">
        <transition name="fade" mode="out-in">
          <component :is="Component" />
        </transition>
      </router-view>
    </main>

    <!-- 页脚 -->
    <footer class="footer">
      <p>&copy; 2026 青年品质租房管理系统 · 连梓祺 & 团队</p>
    </footer>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { useRouter } from 'vue-router'
import { useUserStore } from '@/stores/user'
import {
  HomeFilled,
  OfficeBuilding,
  Search,
  ArrowDown,
  User,
  Setting,
  SwitchButton,
  Plus,
  List,
  Stamp,
  Tickets,
  Bell,
} from '@element-plus/icons-vue'
import { ElMessageBox } from 'element-plus'

const router = useRouter()
const userStore = useUserStore()

// 搜索关键词
const searchKeyword = ref('')

// 待审核房源数量（管理员角标）
const pendingAuditCount = ref(0)

// 待确认订单数量（房东角标）
const landlordPendingCount = ref(0)

// 滚动状态（用于导航栏样式变化）
const isScrolled = ref(false)

// 头像颜色
const avatarColor = computed(() => {
  const colors = ['#ff6b35', '#4facfe', '#2ecc71', '#9b59b6', '#f39c12']
  const idx =
    (userStore.userInfo?.userId?.charCodeAt(0) || 0) % colors.length
  return colors[idx]
})

function handleSearch() {
  if (searchKeyword.value.trim()) {
    router.push({
      path: '/house',
      query: { keyword: searchKeyword.value.trim() },
    })
  }
}

async function handleCommand(command: string) {
  if (command === 'logout') {
    try {
      await ElMessageBox.confirm('确定要退出登录吗？', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning',
      })
      userStore.logout()
      router.push({ name: 'Login' })
    } catch {
      // 取消退出
    }
  } else if (command === 'profile') {
    router.push({ name: 'UserProfile' })
  }
}

// 监听滚动事件
function handleScroll() {
  isScrolled.value = window.scrollY > 20
}

// 获取待审核数量（仅管理员）
async function fetchPendingAuditCount() {
  try {
    const { getPendingAuditList } = await import('@/api/house')
    const res = await getPendingAuditList({ page: 1, size: 1 })
    pendingAuditCount.value = res.total || 0
  } catch {
    // 静默失败，不影响导航使用
  }
}

// 获取房东待确认订单数量（仅房东）
async function fetchLandlordPendingCount() {
  try {
    const { getReceivedOrders } = await import('@/api/order')
    const res = await getReceivedOrders(1, 100)
    const list = res.records || res.list || []
    landlordPendingCount.value = list.filter((o: any) => o.status === 0).length
  } catch {
    // 静默失败
  }
}

onMounted(() => {
  window.addEventListener('scroll', handleScroll)
  // 管理员自动获取待审核数量
  if (userStore.isAdmin) {
    fetchPendingAuditCount()
  }
  // 房东自动获取待确认订单数
  if (userStore.isLandlord) {
    fetchLandlordPendingCount()
  }
})

onUnmounted(() => {
  window.removeEventListener('scroll', handleScroll)
})
</script>

<style lang="scss" scoped>
.main-layout {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

// ==================== 导航栏 ====================
.navbar {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 1000;
  transition: all var(--transition-normal);
  border-bottom: 1px solid transparent;

  &.scrolled {
    background: rgba(255, 255, 255, 0.92) !important;
    backdrop-filter: blur(20px);
    box-shadow: 0 2px 20px rgba(0, 0, 0, 0.06);
    border-bottom-color: var(--color-border-light);
  }
}

.navbar-inner {
  max-width: 1280px;
  margin: 0 auto;
  padding: 0 24px;
  height: 64px;
  display: flex;
  align-items: center;
  gap: 32px;
}

// Logo
.logo {
  display: flex;
  align-items: center;
  gap: 8px;
  text-decoration: none;
  flex-shrink: 0;
  font-size: 20px;
  font-weight: 700;

  .logo-icon {
    font-size: 28px;
  }

  .logo-text {
    letter-spacing: -0.5px;
  }

  &:hover .logo-text {
    opacity: 0.85;
  }
}

// 导航菜单
.nav-menu {
  display: flex;
  gap: 4px;
  margin-left: 16px;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 8px 16px;
  border-radius: var(--radius-sm);
  color: var(--color-text-secondary);
  font-weight: 500;
  font-size: 14px;
  transition: all var(--transition-fast);

  &:hover {
    color: var(--color-primary);
    background: var(--color-primary-bg);
  }

  &.active {
    color: var(--color-primary);
    background: var(--color-primary-bg);
    font-weight: 600;
  }

  &.nav-landlord {
    background: linear-gradient(135deg, rgba(255,107,53,0.08), rgba(255,154,86,0.08));

    &:hover {
      background: linear-gradient(135deg, rgba(255,107,53,0.15), rgba(255,154,86,0.15));
      color: var(--color-primary);
    }

    &.active {
      background: linear-gradient(135deg, var(--color-primary), #ff9a56) !important;
      color: #fff !important;

      .el-icon { color: #fff; }
    }
  }

  // 租客订单菜单
  &.nav-tenant {
    background: linear-gradient(135deg, rgba(79,172,254,0.08), rgba(0,242,254,0.06));

    &:hover {
      background: linear-gradient(135deg, rgba(79,172,254,0.15), rgba(0,242,254,0.12));
      color: #4facfe;
    }

    &.active {
      background: linear-gradient(135deg, #4facfe, #00f2fe) !important;
      color: #fff !important;

      .el-icon { color: #fff; }
    }
  }

  // 管理员审核菜单
  &.nav-admin {
    background: linear-gradient(135deg, rgba(231,76,60,0.06), rgba(243,156,18,0.08));

    &:hover {
      background: linear-gradient(135deg, rgba(231,76,60,0.12), rgba(243,156,18,0.14));
      color: #e74c3c;
    }

    &.active {
      background: linear-gradient(135deg, #e74c3c, #f39c12) !important;
      color: #fff !important;

      .el-icon { color: #fff; }
    }
  }

  .nav-badge {
    margin-left: 4px;

    :deep(.el-badge__content) {
      font-size: 10px;
      transform: scale(0.85);
    }
  }
}

// 右侧区域
.nav-right {
  margin-left: auto;
  display: flex;
  align-items: center;
  gap: 16px;
}

.nav-search {
  width: 200px;
  transition: width var(--transition-normal);

  &:focus-within {
    width: 260px;
  }
}

.user-avatar {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  padding: 4px 10px 4px 4px;
  border-radius: 9999px;
  transition: background var(--transition-fast);

  &:hover {
    background: rgba(0, 0, 0, 0.04);
  }

  .user-name {
    font-size: 14px;
    font-weight: 500;
    max-width: 80px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .arrow {
    font-size: 12px;
    color: var(--color-text-muted);
  }
}

// ==================== 主内容 ====================
.main-content {
  flex: 1;
  margin-top: 64px; // navbar 高度
  min-height: calc(100vh - 64px - 60px);
}

// ==================== 页脚 ====================
.footer {
  text-align: center;
  padding: 24px;
  color: var(--color-text-muted);
  font-size: 13px;
  border-top: 1px solid var(--color-border-light);
}

// ==================== 过渡动画 ====================
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.25s ease, transform 0.25s ease;
}

.fade-enter-from {
  opacity: 0;
  transform: translateY(8px);
}

.fade-leave-to {
  opacity: 0;
  transform: translateY(-8px);
}
</style>
