<template>
  <div class="house-detail-page">
    <!-- 加载中 -->
    <div v-if="pageState === 'loading'" class="detail-loading page-container" style="padding-top: 40px;">
      <el-skeleton :rows="10" animated />
    </div>

    <!-- 已下架 -->
    <div v-else-if="pageState === 'off_shelf'" class="page-container not-found">
      <el-empty description="该房源已下架">
        <template #image>
          <div class="status-icon off-shelf-icon">📦</div>
        </template>
        <el-button type="primary" @click="$router.push('/house')">返回房源列表</el-button>
      </el-empty>
    </div>

    <!-- 不存在 -->
    <div v-else-if="pageState === 'not_found'" class="page-container not-found">
      <el-empty description="房源不存在或已被删除">
        <template #image>
          <div class="status-icon not-found-icon">🔍</div>
        </template>
        <el-button type="primary" @click="$router.push('/house')">返回房源列表</el-button>
      </el-empty>
    </div>

    <template v-else-if="pageState === 'active' && house">
      <!-- 面包屑 -->
      <nav class="breadcrumb page-container">
        <el-breadcrumb separator="/">
          <el-breadcrumb-item :to="{ path: '/' }">首页</el-breadcrumb-item>
          <el-breadcrumb-item :to="{ path: '/house' }">房源列表</el-breadcrumb-item>
          <el-breadcrumb-item>{{ house.title }}</el-breadcrumb-item>
        </el-breadcrumb>
      </nav>

      <div class="detail-content page-container">
        <!-- 左侧：图片 + 详情 -->
        <div class="detail-main">
          <!-- 图片轮播 -->
          <div class="image-gallery">
            <img
              v-if="currentImage"
              :src="currentImage"
              alt=""
              class="main-image"
              @error="handleMainImgError"
            />
            <div v-else class="main-image placeholder-image">
              <span>🏠 暂无房源图片</span>
            </div>

            <!-- 缩略图 -->
            <div v-if="imageList.length > 1" class="thumb-list">
              <div
                v-for="(img, idx) in imageList"
                :key="idx"
                class="thumb-item"
                :class="{ active: currentIdx === idx }"
                @click="currentIdx = idx"
              >
                <img :src="img" alt="" @error="handleThumbError" />
              </div>
            </div>
          </div>

          <!-- 房源信息 -->
          <div class="info-card glass-effect">
            <h1 class="title">{{ house.title }}</h1>
            <p class="price-line">
              <span class="price">¥{{ house.rentMonthly?.toLocaleString() || '-' }}<small>/月</small></span>
              <el-tag
                :type="statusTagType(house.status)"
                size="large"
              >
                {{ getStatusText(house.status) }}
              </el-tag>
            </p>

            <div class="info-grid">
              <div class="info-item">
                <label>面积</label>
                <value>{{ house.area }}m²</value>
              </div>
              <div class="info-item">
                <label>户型</label>
                <value>{{ house.rooms }}室{{ house.halls }}厅{{ house.bathrooms || 0 }}卫</value>
              </div>
              <div class="info-item">
                <label>朝向</label>
                <value>{{ house.orientation || '未知' }}</value>
              </div>
              <div class="info-item">
                <label>楼层</label>
                <value>{{ house.floor || '-' }}</value>
              </div>
              <div class="info-item">
                <label>城市</label>
                <value>{{ house.city }}</value>
              </div>
              <div class="info-item">
                <label>区域</label>
                <value>{{ house.district }}</value>
              </div>
            </div>

            <!-- 地址 -->
            <div class="address-block">
              <el-icon><Location /></el-icon>
              <span>{{ house.address }}</span>
            </div>

            <!-- 房源描述 -->
            <div class="desc-block" v-if="house.description">
              <h3>房源介绍</h3>
              <p>{{ house.description }}</p>
            </div>
          </div>
        </div>

        <!-- 右侧：操作面板 -->
        <aside class="detail-sidebar">
          <!-- 管理员审核面板（管理员查看任意房源时显示） -->
          <div v-if="isAdminView" class="action-card admin-panel">
            <h3>🛡️ 管理员审核</h3>
            <p class="action-desc">{{ adminStatusDesc }}</p>

            <!-- 审核拒绝原因（已拒绝的房源） -->
            <div v-if="house.status === 4 && house.auditRemark" class="landlord-rejection">
              <p class="rejection-label">拒绝原因</p>
              <p class="rejection-text">{{ house.auditRemark }}</p>
            </div>

            <!-- 待审核：审核操作按钮 -->
            <template v-if="house.status === 0">
              <el-button
                type="primary" size="large" class="action-btn approve-btn"
                :loading="auditing"
                @click="handleAdminApprove"
              >
                <el-icon><Select /></el-icon>审核通过
              </el-button>
              <el-button
                type="danger" size="large" class="action-btn outline"
                :loading="auditing"
                @click="openAdminRejectDialog"
              >
                <el-icon><CloseBold /></el-icon>审核拒绝
              </el-button>
            </template>

            <!-- 返回审核列表 -->
            <el-button size="large" class="action-btn outline" @click="$router.push('/admin/audit')">
              返回审核列表
            </el-button>
          </div>

          <!-- 房东管理面板（仅房东查看自己房源时显示） -->
          <div v-else-if="isLandlordView" class="action-card landlord-panel">
            <h3>📋 房源管理</h3>
            <p class="action-desc">{{ landlordStatusDesc }}</p>

            <!-- 审核拒绝原因 -->
            <div v-if="house.status === 4 && house.auditRemark" class="landlord-rejection">
              <p class="rejection-label">拒绝原因</p>
              <p class="rejection-text">{{ house.auditRemark }}</p>
            </div>

            <!-- 操作按钮 -->
            <el-button
              type="primary" size="large" class="action-btn edit-btn"
              @click="$router.push({ path: '/landlord/publish', query: { id: house.houseId } })"
            >
              <el-icon><EditPen /></el-icon>
              {{ house.status === 4 ? '修改后重新提交' : '编辑房源' }}
            </el-button>

            <!-- 下架按钮（status=1 已发布或 status=2 已出租） -->
            <el-button
              v-if="house.status === 1 || house.status === 2"
              size="large" class="action-btn outline" type="warning"
              :loading="togglingStatus"
              @click="handleOffShelf"
            >
              <el-icon><Bottom /></el-icon>下架房源
            </el-button>

            <!-- 重新上架按钮（status=3 已下架） -->
            <el-button
              v-if="house.status === 3"
              size="large" class="action-btn outline" type="success"
              :loading="togglingStatus"
              @click="handleOnShelf"
            >
              <el-icon><Top /></el-icon>重新上架
            </el-button>

            <!-- 返回管理页 -->
            <el-button size="large" class="action-btn outline" @click="$router.push('/landlord/houses')">
              返回房源管理
            </el-button>
          </div>

          <!-- 租客操作面板（非房东视角） -->
          <div v-else class="action-card">
            <h3>{{ house.status === 2 ? '📋 房源状态' : '🏠 立即租房' }}</h3>
            <p class="action-desc">
              {{ house.status === 2 ? '该房源已被出租，您可以浏览其他房源' : '看中这套房？提交租房申请，房东确认后即可签约入住！' }}
            </p>
            <el-button
              v-if="house.status !== 2"
              type="primary"
              size="large"
              class="action-btn rent-btn"
              @click="handleRentClick"
            >
              <el-icon><Key /></el-icon>
              立即租房
            </el-button>
            <el-button v-else size="large" class="action-btn outline" disabled>
              <el-icon><Check /></el-icon>
              该房源已被出租
            </el-button>
            <el-button size="large" class="action-btn outline" @click="handleFavorite">
              <el-icon :class="{ 'is-favorited': isFavorited }"><Star /></el-icon>
              {{ isFavorited ? '已收藏' : '收藏房源' }}
            </el-button>
          </div>

          <!-- 房东信息（模拟，仅租客视角显示） -->
          <div v-if="!isLandlordView" class="landlord-card">
            <div class="landlord-avatar">
              <el-avatar :size="48" style="background: var(--color-primary)">房</el-avatar>
              <div class="landlord-info">
                <strong>品质房东</strong>
                <small>信用良好 · 响应及时</small>
              </div>
            </div>
            <div class="landlord-stats">
              <div class="stat">
                <strong>98%</strong><small>回复率</small>
              </div>
              <div class="stat">
                <strong>&lt;2h</strong><small>平均响应</small>
              </div>
            </div>
          </div>
        </aside>
      </div>
    </template>

    <!-- 租房下单对话框（放在v-if/v-else链条之外，避免编译错误） -->
    <el-dialog
      v-model="showRentDialog"
      title="📋 提交租房申请"
      width="520px"
      class="rent-dialog"
      :close-on-click-modal="false"
    >
      <div class="rent-form">
        <div class="rent-house-summary">
          <p class="rent-house-title">{{ house?.title }}</p>
          <p class="rent-house-price">
            <span class="price-num">¥{{ house?.rentMonthly?.toLocaleString() }}</span>/月 · 押金 ¥{{ house?.deposit?.toLocaleString() }}
          </p>
        </div>

        <el-form :model="rentForm" label-position="top" :rules="rentRules" ref="rentFormRef">
          <el-form-item label="租期开始日期" prop="startDate">
            <el-date-picker
              v-model="rentForm.startDate"
              type="date"
              placeholder="选择入住日期"
              format="YYYY-MM-DD"
              value-format="YYYY-MM-DD"
              style="width: 100%"
              :disabled-date="(date: Date) => date < new Date(new Date().setHours(0,0,0,0))"
            />
          </el-form-item>

          <el-form-item label="租期结束日期" prop="endDate">
            <el-date-picker
              v-model="rentForm.endDate"
              type="date"
              placeholder="选择退房日期"
              format="YYYY-MM-DD"
              value-format="YYYY-MM-DD"
              style="width: 100%"
              :disabled-date="disableEndDate"
            />
          </el-form-item>

          <el-row :gutter="16">
            <el-col :span="12">
              <el-form-item label="联系人姓名" prop="contactName">
                <el-input v-model="rentForm.contactName" placeholder="您的真实姓名" />
              </el-form-item>
            </el-col>
            <el-col :span="12">
              <el-form-item label="联系电话" prop="contactPhone">
                <el-input v-model="rentForm.contactPhone" placeholder="手机号码" maxlength="11" />
              </el-form-item>
            </el-col>
          </el-row>

          <el-form-item label="备注留言（选填）">
            <el-input
              v-model="rentForm.remark"
              type="textarea"
              :rows="3"
              placeholder="如：希望尽快入住、是否有家具需求等..."
              maxlength="200"
              show-word-limit
            />
          </el-form-item>
        </el-form>

        <!-- 费用预估 -->
        <div class="rent-fee-preview" v-if="rentForm.startDate && rentForm.endDate">
          <div class="fee-row">
            <span>月租金</span>
            <span>¥{{ house?.rentMonthly?.toLocaleString() }}/月</span>
          </div>
          <div class="fee-row">
            <span>押金</span>
            <span>¥{{ house?.deposit?.toLocaleString() }}</span>
          </div>
          <div class="fee-divider"></div>
          <div class="fee-row total">
            <span>预计总金额</span>
            <span class="total-price">¥{{ estimatedTotal.toLocaleString() }}</span>
          </div>
          <p class="fee-note">* 实际金额以最终合同为准，此处为预估值（押金 + 月租金 × 月数）</p>
        </div>
      </div>

      <template #footer>
        <el-button @click="showRentDialog = false">取消</el-button>
        <el-button type="primary" @click="handleSubmitRent" :loading="submitting" class="submit-rent-btn">
          <el-icon><Check /></el-icon>
          提交租房申请
        </el-button>
      </template>
    </el-dialog>

    <!-- 游客登录提示对话框 -->
    <el-dialog
      v-model="showLoginPrompt"
      title=""
      width="420px"
      class="login-prompt-dialog"
      :close-on-click-modal="false"
      center
    >
      <div class="login-prompt-content">
        <div class="prompt-icon-wrapper">
          <el-icon :size="48" color="#ff6b35"><UserFilled /></el-icon>
        </div>
        <h3>请您先登录</h3>
        <p>登录后即可提交租房申请，与房东确认签约入住</p>
        <div class="prompt-actions">
          <el-button type="primary" size="large" class="prompt-btn" @click="goToLogin">
            <el-icon><Key /></el-icon>
            登录账号
          </el-button>
          <el-button size="large" class="prompt-btn outline" @click="goToRegister">
            注册新账号
          </el-button>
        </div>
        <p class="prompt-cancel">
          <el-button type="info" link @click="showLoginPrompt = false">继续浏览，暂不登录</el-button>
        </p>
      </div>
    </el-dialog>

    <!-- 管理员拒绝原因弹窗 -->
    <el-dialog
      v-model="showRejectDialog"
      title="审核拒绝"
      width="480px"
      :close-on-click-modal="false"
      destroy-on-close
    >
      <div class="admin-reject-body">
        <p class="reject-tip">请填写拒绝原因，帮助房东了解问题所在：</p>
        <el-input
          v-model="rejectReason"
          type="textarea"
          :rows="4"
          placeholder="例如：房源图片模糊不清、地址信息不完整..."
          maxlength="200"
          show-word-limit
        />
      </div>
      <template #footer>
        <el-button @click="showRejectDialog = false">取消</el-button>
        <el-button type="danger" @click="handleAdminReject" :disabled="!rejectReason.trim()" :loading="rejectLoading">
          确认拒绝
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { Location, Key, Star, Check, UserFilled, EditPen, Top, Bottom, Select, CloseBold } from '@element-plus/icons-vue'
import type { FormInstance, FormRules } from 'element-plus'
import { ElMessage, ElMessageBox } from 'element-plus'
import { useUserStore } from '@/stores/user'
import { getHouseDetail, getMyHouseDetail, approveHouse, rejectHouse } from '@/api/house'
import { offShelfHouse, onShelfHouse } from '@/api/house'
import { getHouseImages } from '@/api/upload'
import { createOrder } from '@/api/order'
import { addFavorite, removeFavorite, checkFavorited } from '@/api/favorite'
import type { HouseInfo } from '@/api/house'

const route = useRoute()
const router = useRouter()
const userStore = useUserStore()

// 页面状态：loading | active | off_shelf | not_found
const pageState = ref<'loading' | 'active' | 'off_shelf' | 'not_found'>('loading')
const house = ref<HouseInfo | null>(null)
const currentIdx = ref(0)
// 是否为房东查看自己的房源
const isLandlordView = ref(false)
// 上下架操作 loading
const togglingStatus = ref(false)
// 是否为管理员视角
const isAdminView = ref(false)
// 管理员审核操作 loading
const auditing = ref(false)
// 拒绝弹窗
const showRejectDialog = ref(false)
const rejectReason = ref('')
const rejectLoading = ref(false)

// 管理员面板的状态描述
const adminStatusDesc = computed(() => {
  if (!house.value) return ''
  const map: Record<number, string> = {
    0: '该房源正在等待审核，请仔细核查房源信息后决定通过或拒绝',
    1: '该房源已审核通过并公开发布',
    2: '该房源已被出租',
    3: '该房源已由房东下架',
    4: '该房源已被审核拒绝',
  }
  return map[house.value.status] || ''
})

// 管理员操作：审核通过
async function handleAdminApprove() {
  if (!house.value) return
  try {
    await ElMessageBox.confirm(
      `确认通过「${house.value.title}」的审核？通过后房源将公开发布。`,
      '审核通过',
      { confirmButtonText: '确认通过', cancelButtonText: '取消', type: 'success' }
    )
    auditing.value = true
    await approveHouse(house.value.houseId)
    ElMessage.success('房源已通过审核并发布')
    house.value.status = 1
  } catch (err: any) {
    if (err !== 'cancel') { /* 已在拦截器中处理 */ }
  } finally {
    auditing.value = false
  }
}

// 管理员操作：打开拒绝弹窗
function openAdminRejectDialog() {
  rejectReason.value = ''
  showRejectDialog.value = true
}

// 管理员操作：确认拒绝
async function handleAdminReject() {
  if (!house.value || !rejectReason.value.trim()) return
  try {
    rejectLoading.value = true
    await rejectHouse(house.value.houseId, rejectReason.value.trim())
    ElMessage.success('已拒绝该房源并通知房东')
    showRejectDialog.value = false
    house.value.status = 4
    house.value.auditRemark = rejectReason.value.trim()
  } catch (err: any) {
    // 已在拦截器中处理
  } finally {
    rejectLoading.value = false
  }
}

// 房东面板的状态描述
const landlordStatusDesc = computed(() => {
  if (!house.value) return ''
  const map: Record<number, string> = {
    0: '该房源正在等待管理员审核，审核通过后将自动发布',
    1: '该房源已公开发布，租客可以浏览和申请租房',
    2: '该房源已被出租，租客正在租住中',
    3: '该房源已下架，租客端不再展示。可重新上架提交审核',
    4: '该房源未通过管理员审核，请根据拒绝原因修改后重新提交',
  }
  return map[house.value.status] || ''
})

// 房东操作：下架
async function handleOffShelf() {
  if (!house.value) return
  try {
    await ElMessageBox.confirm(
      '下架后该房源将不再对租客展示',
      '确认下架',
      { confirmButtonText: '确认下架', cancelButtonText: '取消', type: 'warning' }
    )
    togglingStatus.value = true
    await offShelfHouse(house.value.houseId)
    ElMessage.success('房源已下架')
    house.value.status = 3
  } catch (err: any) {
    if (err !== 'cancel') { /* 已在拦截器中处理 */ }
  } finally {
    togglingStatus.value = false
  }
}

// 房东操作：重新上架
async function handleOnShelf() {
  if (!house.value) return
  try {
    await ElMessageBox.confirm(
      '上架后需等待管理员审核通过才能公开显示',
      '确认重新上架',
      { confirmButtonText: '提交审核', cancelButtonText: '取消', type: 'warning' }
    )
    togglingStatus.value = true
    await onShelfHouse(house.value.houseId)
    ElMessage.success('已提交上架审核，请耐心等待审核结果')
    house.value.status = 0
  } catch (err: any) {
    if (err !== 'cancel') { /* 已在拦截器中处理 */ }
  } finally {
    togglingStatus.value = false
  }
}

// 租房对话框
const showRentDialog = ref(false)
// 游客登录提示对话框
const showLoginPrompt = ref(false)
const submitting = ref(false)
const isFavorited = ref(false)
const rentFormRef = ref<FormInstance>()

// 租房表单数据
const rentForm = ref({
  startDate: '',
  endDate: '',
  contactName: '',
  contactPhone: '',
  remark: '',
})

// 表单校验规则
const rentRules: FormRules = {
  startDate: [{ required: true, message: '请选择入住日期', trigger: 'change' }],
  endDate: [{ required: true, message: '请选择退房日期', trigger: 'change' }],
  contactName: [{ required: true, message: '请输入联系人姓名', trigger: 'blur' }],
  contactPhone: [
    { required: true, message: '请输入联系电话', trigger: 'blur' },
    { pattern: /^1[3-9]\d{9}$/, message: '请输入正确的手机号', trigger: 'blur' },
  ],
}

// 预估总金额
const estimatedTotal = computed(() => {
  if (!rentForm.value.startDate || !rentForm.value.endDate || !house.value) return 0
  const start = new Date(rentForm.value.startDate)
  const end = new Date(rentForm.value.endDate)
  const months = (end.getFullYear() - start.getFullYear()) * 12 + (end.getMonth() - start.getMonth())
  const monthCount = Math.max(1, months)
  const deposit = house.value.deposit || 0
  const monthlyRent = house.value.rentMonthly || 0
  return deposit + monthlyRent * monthCount
})

/** 结束日期不能早于开始日期 */
function disableEndDate(date: Date) {
  if (!rentForm.value.startDate) return date < new Date(new Date().setHours(0,0,0,0))
  return date < new Date(rentForm.value.startDate)
}

/** 点击"立即租房"：登录用户直接打开对话框，游客弹出登录提示 */
function handleRentClick() {
  if (userStore.isLoggedIn) {
    showRentDialog.value = true
  } else {
    showLoginPrompt.value = true
  }
}

/** 游客登录提示 → 跳转登录页，登录后回到当前房源 */
function goToLogin() {
  const houseId = route.params.id as string
  router.push({ name: 'Login', query: { redirect: `/house/${houseId}` } })
}

/** 游客登录提示 → 跳转注册页 */
function goToRegister() {
  router.push({ name: 'Register' })
}

/** 提交租房申请 */
async function handleSubmitRent() {
  if (!rentFormRef.value) return
  try {
    await rentFormRef.value.validate()
  } catch {
    return
  }
  if (!house.value?.houseId) return

  submitting.value = true
  try {
    await createOrder({
      houseId: house.value.houseId,
      startDate: rentForm.value.startDate,
      endDate: rentForm.value.endDate,
      contactName: rentForm.value.contactName,
      contactPhone: rentForm.value.contactPhone,
      remark: rentForm.value.remark,
    })
    ElMessage.success('🎉 租房申请提交成功！请等待房东确认')
    showRentDialog.value = false
    // 重置表单
    rentFormRef.value?.resetFields()
  } catch (err: any) {
    // 错误已在拦截器中处理，这里做兜底
    ElMessage.error(err?.message || '提交失败，请重试')
  } finally {
    submitting.value = false
  }
}

/** 收藏/取消收藏 */
async function handleFavorite() {
  if (!userStore.isLoggedIn) {
    showLoginPrompt.value = true
    return
  }
  const houseId = house.value?.houseId
  if (!houseId) return
  try {
    if (isFavorited.value) {
      await removeFavorite(houseId)
      isFavorited.value = false
      ElMessage.success('已取消收藏')
    } else {
      await addFavorite(houseId)
      isFavorited.value = true
      ElMessage.success('已加入收藏')
    }
  } catch {
    // error handled by interceptor
  }
}

// 图片列表（从 house_image 表查询）
const imageList = ref<string[]>([])

// 当前显示的图片
const currentImage = computed(() => {
  // 优先用 house_image 表的图片列表
  if (imageList.value.length > 0) {
    const img = imageList.value[currentIdx.value] || imageList.value[0]
    if (img && img.trim().length > 0 && !img.startsWith('null')) return img
  }
  // 回退到 coverImage 字段（过滤无效值）
  const cover = house.value?.coverImage
  if (cover && cover.trim().length > 0 && !cover.startsWith('null')) return cover
  // 无封面图时生成占位图
  return ''
})

async function fetchDetail() {
  const id = route.params.id as string
  if (!id) {
    pageState.value = 'not_found'
    return
  }

  pageState.value = 'loading'
  isLandlordView.value = false
  isAdminView.value = false

  // 先尝试公开接口
  try {
    house.value = await getHouseDetail(id, { _silent: true })
    pageState.value = 'active'
    // 管理员视角优先
    if (userStore.isAdmin) {
      isAdminView.value = true
    } else if (userStore.isLoggedIn && house.value.landlordId === userStore.userInfo?.userId) {
      isLandlordView.value = true
    }
    await fetchHouseImages(id)
    // 租客视角下检查收藏状态
    if (userStore.isLoggedIn && !isLandlordView.value && !isAdminView.value) {
      try {
        const checked = await checkFavorited(id)
        isFavorited.value = checked as unknown as boolean
      } catch { /* silent */ }
    }
    return
  } catch (pubErr: any) {
    // 公开接口失败，已登录用户尝试房东接口（可查看自己任意状态的房源）
    if (userStore.isLoggedIn) {
      try {
        house.value = await getMyHouseDetail(id, { _silent: true })
        pageState.value = 'active'
        // 管理员身份优先
        if (userStore.isAdmin) {
          isAdminView.value = true
        } else {
          isLandlordView.value = true
        }
        await fetchHouseImages(id)
        return
      } catch {
        // 房东接口也失败，使用公开接口的错误码
      }
    }
    house.value = null
    if (pubErr.code === 310) {
      pageState.value = 'off_shelf'
    } else {
      pageState.value = 'not_found'
    }
  }
}

/** 从 house_image 表获取该房源的所有图片 */
async function fetchHouseImages(houseId: string) {
  try {
    const res = await getHouseImages(houseId)
    // 返回格式：{ list: [...], cover: '...', count: N }
    if (res?.list && res.list.length > 0) {
      imageList.value = res.list
      // 如果有封面图且不在列表中，把它加到最前面
      if (house.value?.coverImage && !res.list.includes(house.value.coverImage)) {
        imageList.value.unshift(house.value.coverImage)
      }
    } else if (house.value?.coverImage) {
      // 没有从表查到数据，但有 coverImage 字段，单独展示
      imageList.value = [house.value.coverImage]
    }
  } catch {
    // 图片获取失败不影响主流程
  }
}

/** 生成详情页大图占位 SVG */
function genDetailPlaceholder(): string {
  const colors = ['#ff6b35', '#4facfe', '#2ecc71', '#9b59b6', '#f39c12']
  const idx = (house.value?.houseId?.charCodeAt(0) || 0) % colors.length
  const color = colors[idx]
  const text = (house.value?.title || '🏠').substring(0, 8)
  return `data:image/svg+xml,${encodeURIComponent(`<svg xmlns='http://www.w3.org/2000/svg' width='800' height='500'><defs><linearGradient id='g' x1='0' y1='0' x2='1' y2='1'><stop offset='0%' stop-color='${color}'/><stop offset='100%' stop-color='${color}99'/></linearGradient></defs><rect fill='url(%23g)' width='800' height='500'/><text x='50%' y='48%' dominant-baseline='middle' text-anchor='middle' font-size='64' fill='white' opacity='0.9'>🏠</text><text x='50%' y='62%' dominant-baseline='middle' text-anchor='middle' font-size='28' fill='white' font-weight='bold'>${text}</text></svg>`)}`
}

/** 主图加载失败 → 替换为占位图 */
function handleMainImgError(e: Event) {
  const img = e.target as HTMLImageElement
  if (img.dataset.fallbacked) return
  img.src = genDetailPlaceholder()
  img.dataset.fallbacked = '1'
}

/** 缩略图加载失败 → 隐藏该缩略图 */
function handleThumbError(e: Event) {
  const img = e.target as HTMLImageElement
  if (!img.parentElement) return
  img.parentElement.style.visibility = 'hidden'
}

function getStatusText(status: number): string {
  const map: Record<number, string> = {
    0: '待审核',
    1: '已发布',
    2: '已出租',
    3: '已下架',
    4: '审核拒绝',
  }
  return map[status] || '未知'
}

function statusTagType(status: number): 'success' | 'warning' | 'info' | 'danger' | '' {
  const map: Record<number, 'success' | 'warning' | 'info' | 'danger' | ''> = {
    0: 'warning',  // 待审核
    1: 'success',  // 已发布
    2: 'info',     // 已出租
    3: 'danger',   // 已下架
    4: 'danger',   // 审核拒绝
  }
  return map[status] || 'info'
}

onMounted(() => {
  fetchDetail()
})

watch(() => route.params.id, () => {
  currentIdx.value = 0
  imageList.value = []
  fetchDetail()
})
</script>

<style lang="scss" scoped>
.house-detail-page {
  min-height: calc(100vh - 64px);
  background: var(--color-bg);
  padding-bottom: 60px;
}

.breadcrumb {
  padding-top: 20px;
  padding-bottom: 12px;
}

.detail-content {
  display: grid;
  grid-template-columns: 1fr 340px;
  gap: 28px;
  align-items: start;
}

// ==================== 主内容区 ====================
.detail-main {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.image-gallery {
  background: white;
  border-radius: var(--radius-lg);
  overflow: hidden;

  .main-image {
    width: 100%;
    height: 420px;
    object-fit: cover;
  }

  .placeholder-image {
    height: 420px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 20px;
    color: var(--color-text-muted);
    background: linear-gradient(135deg, #f8f9fa, #e9ecef);
  }

  .thumb-list {
    display: flex;
    gap: 8px;
    padding: 12px;
    overflow-x: auto;

    .thumb-item {
      width: 80px;
      height: 60px;
      border-radius: var(--radius-sm);
      overflow: hidden;
      cursor: pointer;
      border: 2px solid transparent;
      transition: border-color var(--transition-fast);

      img {
        width: 100%;
        height: 100%;
        object-fit: cover;
      }

      &.active {
        border-color: var(--color-primary);
      }
    }
  }
}

.info-card {
  background: white;
  padding: 28px;
  border-radius: var(--radius-lg);

  .title {
    font-size: 24px;
    font-weight: 800;
    margin-bottom: 12px;
  }

  .price-line {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 24px;
    padding-bottom: 20px;
    border-bottom: 1px solid var(--color-border-light);

    .price {
      font-size: 32px;
      font-weight: 800;
      color: var(--color-primary);

      small {
        font-size: 15px;
        font-weight: 400;
        color: var(--color-text-muted);
      }
    }
  }
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px;
  margin-bottom: 24px;

  .info-item {
    display: flex;
    flex-direction: column;
    gap: 4px;

    label {
      font-size: 13px;
      color: var(--color-text-muted);
    }

    value {
      font-size: 16px;
      font-weight: 600;
    }
  }
}

.address-block {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 14px 18px;
  background: var(--color-bg);
  border-radius: var(--radius-sm);
  font-size: 14px;
  color: var(--color-text-secondary);
  margin-bottom: 24px;
}

.desc-block {
  h3 {
    font-size: 17px;
    font-weight: 700;
    margin-bottom: 10px;
  }

  p {
    font-size: 14px;
    line-height: 1.8;
    color: var(--color-text-secondary);
  }
}

// ==================== 侧边栏 ====================
.detail-sidebar {
  display: flex;
  flex-direction: column;
  gap: 20px;
  position: sticky;
  top: 88px;
}

.action-card {
  background: white;
  padding: 24px;
  border-radius: var(--radius-lg);

  h3 {
    font-size: 19px;
    font-weight: 700;
    margin-bottom: 8px;
  }

  .action-desc {
    font-size: 14px;
    color: var(--color-text-secondary);
    margin-bottom: 20px;
  }

  .action-btn {
    width: 100%;
    margin-bottom: 10px;
    border-radius: var(--radius-sm) !important;
    font-weight: 600;

    &.outline {
      border: 2px solid var(--color-border) !important;
      background: transparent !important;
      color: var(--color-text-secondary) !important;

      &:hover {
        border-color: var(--color-primary) !important;
        color: var(--color-primary) !important;
      }
    }

    &.rent-btn {
      background: linear-gradient(135deg, #ff6b35, #ff9a56) !important;
      border: none !important;
      box-shadow: 0 4px 15px rgba(255, 107, 53, 0.35);

      &:hover {
        transform: translateY(-2px);
        box-shadow: 0 6px 20px rgba(255, 107, 53, 0.45) !important;
      }
    }

    &.edit-btn {
      background: linear-gradient(135deg, #4facfe, #2e8ece) !important;
      border: none !important;
      box-shadow: 0 4px 15px rgba(79, 172, 254, 0.35);

      &:hover {
        transform: translateY(-2px);
        box-shadow: 0 6px 20px rgba(79, 172, 254, 0.45) !important;
      }
    }
  }
}

// 房东面板内的审核拒绝提示（管理员面板也复用）
.landlord-rejection {
  background: #fef2f2;
  border-radius: var(--radius-sm);
  border-left: 3px solid #ef4444;
  padding: 10px 14px;
  margin-bottom: 16px;

  .rejection-label {
    font-size: 12px;
    font-weight: 600;
    color: #dc2626;
    margin-bottom: 4px;
  }

  .rejection-text {
    font-size: 13px;
    color: #991b1b;
    line-height: 1.5;
    margin: 0;
  }
}

// 管理员审核面板
.admin-panel {
  .approve-btn {
    background: linear-gradient(135deg, #2ecc71, #27ae60) !important;
    border: none !important;
    box-shadow: 0 4px 15px rgba(46, 204, 113, 0.35);

    &:hover {
      transform: translateY(-2px);
      box-shadow: 0 6px 20px rgba(46, 204, 113, 0.45) !important;
    }
  }
}

// 管理员拒绝弹窗
.admin-reject-body {
  .reject-tip {
    font-size: 14px;
    color: var(--color-text-secondary);
    margin-bottom: 14px;
    line-height: 1.5;
  }
}

.landlord-card {
  background: white;
  padding: 20px;
  border-radius: var(--radius-lg);

  .landlord-avatar {
    display: flex;
    align-items: center;
    gap: 12px;
    margin-bottom: 16px;

    .landlord-info {
      display: flex;
      flex-direction: column;

      strong { font-size: 15px; }
      small { font-size: 12px; color: var(--color-text-muted); }
    }
  }

  .landlord-stats {
    display: flex;
    gap: 24px;
    padding-top: 16px;
    border-top: 1px solid var(--color-border-light);

    .stat {
      display: flex;
      flex-direction: column;
      gap: 2px;

      strong { font-size: 17px; font-weight: 800; }
      small { font-size: 12px; color: var(--color-text-muted); }
    }
  }
}

// ==================== 租房对话框 ====================
:deep(.rent-dialog) {
  .el-dialog__header {
    background: linear-gradient(135deg, #ff6b35, #ff9a56);
    padding: 18px 24px;
    margin-right: 0;
    border-radius: var(--radius-lg) var(--radius-lg) 0 0;

    .el-dialog__title {
      color: #fff;
      font-size: 18px;
      font-weight: 700;
    }
  }

  .el-dialog__body {
    padding: 20px 24px;
  }
}

.rent-form {
  .rent-house-summary {
    background: linear-gradient(135deg, #fff5f0, #fff);
    border-radius: var(--radius-sm);
    padding: 14px 18px;
    margin-bottom: 20px;
    border-left: 4px solid var(--color-primary);

    .rent-house-title {
      font-size: 15px;
      font-weight: 600;
      color: var(--color-text-primary);
      margin: 0 0 6px;
    }

    .rent-house-price {
      margin: 0;

      .price-num {
        font-size: 22px;
        font-weight: 800;
        color: var(--color-primary);
      }
    }
  }
}

.rent-fee-preview {
  background: var(--color-bg);
  border-radius: var(--radius-sm);
  padding: 16px;
  margin-top: 16px;

  .fee-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 8px 0;
    font-size: 14px;
    color: var(--color-text-secondary);

    &.total {
      padding-top: 12px;
      border-top: 2px solid var(--color-border-light);
      margin-top: 4px;

      span:first-child {
        font-weight: 700;
        color: var(--color-text-primary);
        font-size: 15px;
      }
    }
  }

  .total-price {
    font-size: 20px !important;
    font-weight: 800 !important;
    color: var(--color-primary) !important;
  }

  .fee-divider {
    height: 1px;
    background: var(--color-border-light);
    margin: 4px 0;
  }

  .fee-note {
    font-size: 11px;
    color: var(--color-text-muted);
    margin: 10px 0 0;
  }
}

.submit-rent-btn {
  background: linear-gradient(135deg, #ff6b35, #ff9a56) !important;
  border: none !important;
  font-weight: 600 !important;
  padding: 12px 28px !important;
}

.is-favorited {
  color: var(--color-primary) !important;
}

.not-found {
  text-align: center;
  padding-top: 80px;
}

.status-icon {
  font-size: 64px;
  margin-bottom: 16px;
}

// 游客登录提示对话框
.login-prompt-dialog {
  :deep(.el-dialog__header) {
    display: none;
  }

  :deep(.el-dialog__body) {
    padding: 40px 32px 24px;
  }
}

.login-prompt-content {
  text-align: center;

  .prompt-icon-wrapper {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 80px;
    height: 80px;
    border-radius: 50%;
    background: rgba(255, 107, 53, 0.08);
    margin-bottom: 16px;
  }

  h3 {
    font-size: 20px;
    font-weight: 700;
    color: var(--color-text);
    margin-bottom: 8px;
  }

  p {
    color: var(--color-text-secondary);
    font-size: 14px;
    margin-bottom: 24px;
    line-height: 1.5;
  }

  .prompt-actions {
    display: flex;
    flex-direction: column;
    gap: 12px;
    max-width: 260px;
    margin: 0 auto;
  }

  .prompt-btn {
    width: 100%;
    height: 44px !important;
    font-size: 15px !important;
    font-weight: 600 !important;
    border-radius: var(--radius-sm) !important;

    &.outline {
      border-color: var(--color-primary);
      color: var(--color-primary);

      &:hover {
        background: rgba(255, 107, 53, 0.08);
      }
    }
  }

  .prompt-cancel {
    margin-top: 16px;
    margin-bottom: 0;
  }
}
</style>
