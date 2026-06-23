<template>
  <div class="profile-page">
    <div class="page-container">
      <h2 class="page-title">个人信息</h2>

      <el-tabs v-model="activeTab" class="profile-tabs">
        <!-- ==================== Tab 1: 个人信息 ==================== -->
        <el-tab-pane label="个人信息" name="info">
          <div class="profile-card" v-loading="profileLoading">
            <div class="profile-top">
              <el-avatar :size="80" :src="profile?.avatarUrl || undefined">
                {{ profile?.nickname?.charAt(0) || 'U' }}
              </el-avatar>
              <div class="profile-badges">
                <el-tag type="primary" size="small">{{ profile?.roleName || '-' }}</el-tag>
                <el-tag v-if="profile?.isVerified === 1" type="success" size="small">已认证</el-tag>
                <el-tag v-else type="info" size="small">未认证</el-tag>
              </div>
            </div>

            <div class="info-grid">
              <div class="info-item">
                <span class="info-label">真实姓名</span>
                <span class="info-value">{{ profile?.realName || '未设置' }}</span>
              </div>
              <div class="info-item">
                <span class="info-label">手机号</span>
                <span class="info-value">{{ maskPhone(profile?.phone) }}</span>
              </div>
              <div class="info-item">
                <span class="info-label">邮箱</span>
                <span class="info-value">{{ profile?.email || '未设置' }}</span>
              </div>

              <!-- 可编辑字段 -->
              <div class="info-item">
                <span class="info-label">昵称</span>
                <span v-if="!editMode" class="info-value">{{ profile?.nickname || '-' }}</span>
                <el-input v-else v-model="editForm.nickname" size="small" class="edit-input" />
              </div>
              <div class="info-item">
                <span class="info-label">性别</span>
                <span v-if="!editMode" class="info-value">{{ genderLabel(profile?.gender) }}</span>
                <el-radio-group v-else v-model="editForm.gender" size="small">
                  <el-radio-button :value="0">未知</el-radio-button>
                  <el-radio-button :value="1">男</el-radio-button>
                  <el-radio-button :value="2">女</el-radio-button>
                </el-radio-group>
              </div>
              <div class="info-item">
                <span class="info-label">头像URL</span>
                <span v-if="!editMode" class="info-value url-value">{{ profile?.avatarUrl || '未设置' }}</span>
                <el-input v-else v-model="editForm.avatarUrl" size="small" class="edit-input" placeholder="输入图片URL" />
              </div>
              <div class="info-item">
                <span class="info-label">支付密码</span>
                <span class="info-value">
                  <template v-if="hasPayPwd">已设置</template>
                  <template v-else>未设置</template>
                  <el-button link type="primary" size="small" style="margin-left:8px" @click="openPayPwdDialog">
                    {{ hasPayPwd ? '修改' : '设置' }}
                  </el-button>
                </span>
              </div>
            </div>

            <div class="info-actions">
              <template v-if="!editMode">
                <el-button type="primary" size="small" @click="startEdit">编辑</el-button>
              </template>
              <template v-else>
                <el-button type="primary" size="small" :loading="saving" @click="saveProfile">保存</el-button>
                <el-button size="small" @click="cancelEdit">取消</el-button>
              </template>
            </div>
          </div>
        </el-tab-pane>

        <!-- ==================== Tab 2: 我的收藏 ==================== -->
        <el-tab-pane label="我的收藏" name="favorites">
          <!-- 加载中 -->
          <div v-if="favLoading" class="loading-grid">
            <el-skeleton v-for="i in 6" :key="i" :rows="5" animated />
          </div>

          <!-- 卡片列表 -->
          <div v-else-if="favorites.length > 0" class="fav-list">
            <div
              v-for="item in favorites"
              :key="item.id"
              class="fav-card"
              @click="goDetail(item.houseId)"
            >
              <div class="card-image">
                <img :src="getCoverImage(item)" :alt="item.title" @error="handleImgError" />
                <div class="image-overlay">
                  <span class="status-badge" :class="'status-' + item.status">
                    {{ getStatusText(item.status) }}
                  </span>
                </div>
              </div>
              <div class="card-body">
                <h3 class="card-title">{{ item.title }}</h3>
                <p class="card-location">
                  <el-icon><Location /></el-icon>
                  {{ item.city }}{{ item.district }} · {{ item.address }}
                </p>
                <div class="card-tags">
                  <el-tag size="small">{{ item.area }}m²</el-tag>
                  <el-tag size="small" type="info">{{ item.rooms }}室{{ item.halls }}厅</el-tag>
                  <el-tag size="small" type="info">{{ item.orientation || '-' }}</el-tag>
                </div>
                <div class="card-bottom">
                  <span class="price">¥{{ item.rentMonthly?.toLocaleString() || '-' }}<small>/月</small></span>
                  <span class="time">{{ formatTime(item.createTime) }}</span>
                </div>
                <el-button
                  type="danger"
                  plain
                  size="small"
                  class="unfav-btn"
                  :loading="removingIds.has(item.houseId)"
                  @click.stop="handleUnfavorite(item)"
                >
                  取消收藏
                </el-button>
              </div>
            </div>
          </div>

          <!-- 空状态 -->
          <el-empty v-else description="还没有收藏房源">
            <el-button type="primary" @click="$router.push('/house')">去浏览房源</el-button>
          </el-empty>

          <!-- 分页 -->
          <div v-if="favTotal > 0" class="pagination-wrap">
            <el-pagination
              v-model:current-page="favPage"
              v-model:page-size="favSize"
              :total="favTotal"
              :page-sizes="[6, 12, 18]"
              layout="total, sizes, prev, pager, next, jumper"
              background
              @current-change="fetchFavorites"
              @size-change="fetchFavorites"
            />
          </div>
        </el-tab-pane>
      </el-tabs>
    </div>

    <!-- 支付密码对话框 -->
    <el-dialog v-model="payPwdDialogVisible" :title="hasPayPwd ? '修改支付密码' : '设置支付密码'" width="380px" :close-on-click-modal="false">
      <el-form @submit.prevent="savePayPwd">
        <el-form-item v-if="hasPayPwd" label="原密码">
          <el-input v-model="payPwdForm.oldPwd" type="password" show-password maxlength="6" placeholder="请输入原支付密码" />
        </el-form-item>
        <el-form-item label="新密码">
          <el-input v-model="payPwdForm.newPwd" type="password" show-password maxlength="6" placeholder="请输入6位数字密码" />
        </el-form-item>
        <el-form-item label="确认新密码">
          <el-input v-model="payPwdForm.confirmPwd" type="password" show-password maxlength="6" placeholder="再次输入新密码" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="payPwdDialogVisible = false">取消</el-button>
        <el-button type="primary" :loading="savingPayPwd" @click="savePayPwd">确认</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, watch, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { ElMessage, ElMessageBox } from 'element-plus'
import { Location } from '@element-plus/icons-vue'
import { useUserStore } from '@/stores/user'
import { getUserInfo, updateUserInfo } from '@/api/user'
import type { UserProfile } from '@/api/user'
import { getMyFavorites, removeFavorite } from '@/api/favorite'
import type { FavoriteItem } from '@/api/favorite'
import { hasPaymentPassword, setPaymentPassword } from '@/api/payment'

const router = useRouter()
const userStore = useUserStore()

// ==================== 个人信息 ====================
const activeTab = ref('info')
const profile = ref<UserProfile | null>(null)
const profileLoading = ref(false)
const editMode = ref(false)
const saving = ref(false)
const hasPayPwd = ref(false)
const payPwdDialogVisible = ref(false)
const savingPayPwd = ref(false)
const payPwdForm = reactive({
  oldPwd: '',
  newPwd: '',
  confirmPwd: '',
})
const editForm = reactive({
  nickname: '',
  gender: 0,
  avatarUrl: '',
})

async function loadProfile() {
  profileLoading.value = true
  try {
    profile.value = await getUserInfo() as unknown as UserProfile
    // 检查支付密码状态
    try {
      hasPayPwd.value = await hasPaymentPassword() as unknown as boolean
    } catch { /* silent */ }
  } catch {
    // error handled by interceptor
  } finally {
    profileLoading.value = false
  }
}

function startEdit() {
  if (!profile.value) return
  editForm.nickname = profile.value.nickname || ''
  editForm.gender = profile.value.gender ?? 0
  editForm.avatarUrl = profile.value.avatarUrl || ''
  editMode.value = true
}

function cancelEdit() {
  editMode.value = false
}

async function saveProfile() {
  saving.value = true
  try {
    await updateUserInfo({
      nickname: editForm.nickname || undefined,
      gender: editForm.gender,
      avatarUrl: editForm.avatarUrl || undefined,
    })
    editMode.value = false
    ElMessage.success('保存成功')
    // 重新加载信息并刷新 store
    await loadProfile()
    await userStore.fetchUserInfo()
  } catch {
    // error handled by interceptor
  } finally {
    saving.value = false
  }
}

function genderLabel(g?: number): string {
  if (g === 1) return '男'
  if (g === 2) return '女'
  return '未知'
}

function maskPhone(phone?: string): string {
  if (!phone || phone.length !== 11) return phone || '-'
  return phone.slice(0, 3) + '****' + phone.slice(7)
}

// ==================== 支付密码 ====================
function openPayPwdDialog() {
  payPwdForm.oldPwd = ''
  payPwdForm.newPwd = ''
  payPwdForm.confirmPwd = ''
  payPwdDialogVisible.value = true
}

async function savePayPwd() {
  if (!payPwdForm.newPwd || payPwdForm.newPwd.length !== 6 || !/^\d{6}$/.test(payPwdForm.newPwd)) {
    ElMessage.warning('新密码必须为6位数字')
    return
  }
  if (payPwdForm.newPwd !== payPwdForm.confirmPwd) {
    ElMessage.warning('两次输入的密码不一致')
    return
  }
  if (hasPayPwd.value && !payPwdForm.oldPwd) {
    ElMessage.warning('请输入原支付密码')
    return
  }

  savingPayPwd.value = true
  try {
    await setPaymentPassword(
      hasPayPwd.value ? payPwdForm.oldPwd : '',
      payPwdForm.newPwd
    )
    hasPayPwd.value = true
    payPwdDialogVisible.value = false
    ElMessage.success(hasPayPwd.value ? '支付密码已修改' : '支付密码已设置')
  } catch {
    // error handled by interceptor
  } finally {
    savingPayPwd.value = false
  }
}

// ==================== 我的收藏 ====================
const favorites = ref<FavoriteItem[]>([])
const favTotal = ref(0)
const favPage = ref(1)
const favSize = ref(12)
const favLoading = ref(false)
const removingIds = ref<Set<string>>(new Set())
const favLoaded = ref(false)

async function fetchFavorites() {
  favLoading.value = true
  try {
    const res = await getMyFavorites(favPage.value, favSize.value)
    favorites.value = res.records || []
    favTotal.value = res.total || 0
    favLoaded.value = true
  } catch {
    favorites.value = []
    favTotal.value = 0
  } finally {
    favLoading.value = false
  }
}

async function handleUnfavorite(item: FavoriteItem) {
  try {
    await ElMessageBox.confirm(`确定取消收藏「${item.title}」？`, '提示', {
      type: 'warning',
      confirmButtonText: '确定',
      cancelButtonText: '取消',
    })
  } catch {
    return // 用户取消
  }

  removingIds.value.add(item.houseId)
  try {
    await removeFavorite(item.houseId)
    ElMessage.success('已取消收藏')
    // 从列表中移除
    favorites.value = favorites.value.filter(f => f.id !== item.id)
    favTotal.value = Math.max(0, favTotal.value - 1)
    // 如果当前页为空且不是第一页，回退一页
    if (favorites.value.length === 0 && favPage.value > 1) {
      favPage.value--
      fetchFavorites()
    }
  } catch {
    // error handled by interceptor
  } finally {
    removingIds.value.delete(item.houseId)
  }
}

function goDetail(houseId: string) {
  router.push(`/house/${houseId}`)
}

// ==================== 从 HouseList 复用的工具函数 ====================
function getCoverImage(item: FavoriteItem): string {
  if (item.coverImage && item.coverImage.trim().length > 0 && !item.coverImage.startsWith('null')) return item.coverImage
  const colors = ['#ff6b35', '#4facfe', '#2ecc71', '#9b59b6', '#f39c12', '#e74c3c']
  const idx = (item.houseId?.charCodeAt(0) || 0) % colors.length
  const color = colors[idx]
  const text = (item.title || '🏠').substring(0, 6)
  const info = `${item.rooms || '-'}室${item.halls || '-'}厅 · ${item.area || '?'}m²`
  return genPlaceholder(color, text, info)
}

function handleImgError(e: Event) {
  const img = e.target as HTMLImageElement
  if (img.dataset.fallbacked) return
  const colors = ['#ff6b35', '#4facfe', '#2ecc71', '#9b59b6', '#f39c12']
  const color = colors[Math.floor(Math.random() * colors.length)]
  img.src = genPlaceholder(color, '🏠', '')
  img.dataset.fallbacked = '1'
}

function genPlaceholder(color: string, text: string, info: string): string {
  const iSvg = info ? `<text x='50%' y='80%' dominant-baseline='middle' text-anchor='middle' font-size='14' fill='white' opacity='0.7'>${info}</text>` : ''
  return `data:image/svg+xml,${encodeURIComponent(`<svg xmlns='http://www.w3.org/2000/svg' width='400' height='300'><defs><linearGradient id='g' x1='0' y1='0' x2='1' y2='1'><stop offset='0%' stop-color='${color}'/><stop offset='100%' stop-color='${color}88'/></linearGradient></defs><rect fill='url(%23g)' width='400' height='300'/><text x='50%' y='42%' dominant-baseline='middle' text-anchor='middle' font-size='48' fill='white' opacity='0.9'>🏠</text><text x='50%' y='65%' dominant-baseline='middle' text-anchor='middle' font-size='18' fill='white' font-weight='bold'>${text}</text>${iSvg}</svg>`)}`
}

function getStatusText(status: number): string {
  const map: Record<number, string> = {
    0: '待审核', 1: '已发布', 2: '已出租', 3: '已下架', 4: '审核拒绝',
  }
  return map[status] || '未知'
}

function formatTime(time: string): string {
  const d = new Date(time)
  const now = new Date()
  const diff = now.getTime() - d.getTime()
  const days = Math.floor(diff / (1000 * 60 * 60 * 24))
  if (days === 0) return '今天收藏'
  if (days === 1) return '昨天收藏'
  if (days < 7) return `${days}天前`
  return time.substring(0, 10)
}

// ==================== Lazy Load ====================
watch(activeTab, (tab) => {
  if (tab === 'favorites' && !favLoaded.value) {
    fetchFavorites()
  }
})

onMounted(() => {
  loadProfile()
})
</script>

<style lang="scss" scoped>
.profile-page {
  min-height: calc(100vh - 64px);
  background: var(--color-bg);
  padding: 32px 0;
}

.page-title {
  font-size: 24px;
  font-weight: 700;
  margin-bottom: 24px;
  color: var(--color-text-primary);
}

.profile-tabs {
  :deep(.el-tabs__header) {
    margin-bottom: 24px;
  }
}

// ==================== 个人信息卡片 ====================
.profile-card {
  background: white;
  border-radius: var(--radius-lg);
  padding: 32px;
  box-shadow: var(--shadow-sm);
}

.profile-top {
  display: flex;
  align-items: center;
  gap: 16px;
  margin-bottom: 28px;

  .profile-badges {
    display: flex;
    gap: 8px;
  }
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 20px;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 4px;

  .info-label {
    font-size: 13px;
    color: var(--color-text-muted);
  }

  .info-value {
    font-size: 15px;
    font-weight: 500;
    color: var(--color-text-primary);

    &.url-value {
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
      max-width: 300px;
    }
  }

  .edit-input {
    max-width: 260px;
  }
}

.info-actions {
  margin-top: 28px;
  padding-top: 20px;
  border-top: 1px solid var(--color-border-light);
}

// ==================== 收藏列表 ====================
.fav-list {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 24px;
}

.fav-card {
  position: relative;
  background: white;
  border-radius: var(--radius-lg);
  overflow: hidden;
  cursor: pointer;
  transition: all var(--transition-normal);

  &:hover {
    transform: translateY(-6px);
    box-shadow: var(--shadow-lg);

    .card-image img {
      transform: scale(1.06);
    }
  }

  .card-image {
    position: relative;
    height: 200px;
    overflow: hidden;
    background: linear-gradient(135deg, #f8f9fa, #e9ecef);

    img {
      width: 100%;
      height: 100%;
      object-fit: cover;
      transition: transform var(--transition-slow);
    }

    .image-overlay {
      position: absolute;
      top: 10px;
      left: 10px;

      .status-badge {
        padding: 3px 10px;
        border-radius: 9999px;
        font-size: 12px;
        font-weight: 600;
        color: white;

        &.status-0 { background: #f39c12; }
        &.status-1 { background: linear-gradient(135deg, #2ecc71, #27ae60); }
        &.status-2 { background: #3498db; }
        &.status-3 { background: #95a5a6; }
        &.status-4 { background: #e74c3c; }
      }
    }
  }

  .card-body {
    padding: 16px;

    .card-title {
      font-size: 17px;
      font-weight: 700;
      margin-bottom: 6px;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }

    .card-location {
      font-size: 13px;
      color: var(--color-text-secondary);
      display: flex;
      align-items: center;
      gap: 4px;
      margin-bottom: 10px;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }

    .card-tags {
      display: flex;
      flex-wrap: wrap;
      gap: 6px;
      margin-bottom: 12px;
    }

    .card-bottom {
      display: flex;
      justify-content: space-between;
      align-items: center;
      border-top: 1px solid var(--color-border-light);
      padding-top: 12px;
      margin-bottom: 10px;

      .price {
        font-size: 22px;
        font-weight: 800;
        color: var(--color-primary);

        small {
          font-size: 13px;
          font-weight: 400;
          color: var(--color-text-muted);
        }
      }

      .time {
        font-size: 12px;
        color: var(--color-text-muted);
      }
    }

    .unfav-btn {
      width: 100%;
    }
  }
}

.loading-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 20px;
}

.pagination-wrap {
  padding: 32px 0;
  display: flex;
  justify-content: center;
}
</style>
