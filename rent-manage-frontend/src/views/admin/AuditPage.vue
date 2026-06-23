<template>
  <div class="audit-page">
    <div class="page-container">
      <!-- 页面头部 -->
      <div class="page-header">
        <div class="header-left">
          <h1>
            <el-icon><Stamp /></el-icon>
            房源审核管理
          </h1>
          <span class="pending-count">
            当前有
            <strong>{{ total }}</strong>
            套房源待审核
          </span>
        </div>

        <!-- 城市筛选 -->
        <div class="filter-bar">
          <el-select v-model="filterCity" placeholder="筛选城市" clearable @change="handleFilter" style="width: 160px">
            <el-option label="全部城市" value="" />
            <el-option v-for="city in cityOptions" :key="city" :label="city" :value="city" />
          </el-select>
        </div>
      </div>

      <!-- 统计卡片 -->
      <div class="stats-row">
        <div class="stat-card stat-pending">
          <span class="stat-num">{{ total }}</span>
          <span class="stat-label">待审核</span>
          <span class="stat-icon">⏳</span>
        </div>
        <div class="stat-card stat-info">
          <span class="stat-num">{{ todayCount }}</span>
          <span class="stat-label">今日新增</span>
          <span class="stat-icon">📅</span>
        </div>
      </div>

      <!-- 审核列表 -->
      <div v-loading="loading" class="audit-list">
        <template v-if="houses.length > 0">
          <div v-for="house in houses" :key="house.houseId" class="audit-card">
            <!-- 左侧：封面图 -->
            <div class="audit-cover">
              <img :src="getCoverImage(house)" :alt="house.title" @error="handleImgError" :data-house-id="house.houseId" :data-title="house.title" />
              <span class="time-badge">
                {{ formatTimeAgo(house.createTime) }}
              </span>
            </div>

            <!-- 中间：房源信息 -->
            <div class="audit-info">
              <h3 class="house-title">{{ house.title }}</h3>
              <p class="house-addr">
                📍 {{ house.city }} · {{ house.district }} · {{ house.address }}
              </p>

              <div class="house-tags">
                <el-tag size="small">{{ house.rooms }}室{{ house.halls }}厅{{ house.bathrooms || 0 }}卫</el-tag>
                <el-tag size="small" type="info">{{ house.area }}㎡</el-tag>
                <el-tag size="small" type="success">¥{{ house.rentMonthly }}/月</el-tag>
                <el-tag size="small" type="warning">{{ paymentTypeText(house.paymentType) }}</el-tag>
              </div>

              <p class="house-desc" v-if="house.description">
                {{ truncateDesc(house.description) }}
              </p>

              <div class="house-meta">
                <span>🆔 房东ID: {{ house.landlordId?.slice(0, 8) }}...</span>
                <span>📅 提交于 {{ formatDate(house.createTime) }}</span>
              </div>
            </div>

            <!-- 右侧：操作区 -->
            <div class="audit-actions">
              <el-button text type="info" @click="$router.push(`/house/${house.houseId}`)">
                <el-icon><View /></el-icon>查看
              </el-button>
              <el-button type="primary" size="large" @click="handleApprove(house)" :loading="house._approving">
                <el-icon><Select /></el-icon> 通过
              </el-button>
                <el-button type="danger" plain size="large" @click="openRejectDialog(house)" :loading="house._rejecting">
                  <el-icon><CloseBold /></el-icon> 拒绝
              </el-button>
            </div>
          </div>
        </template>

        <el-empty v-else-if="!loading" description="暂无待审核的房源 ✨">
          <template #image>
            <span style="font-size: 64px;">🎉</span>
          </template>
          <p style="color: var(--color-text-muted); margin-top: 8px;">所有房源已处理完毕，辛苦了！</p>
        </el-empty>
      </div>

      <!-- 分页 -->
      <div class="pagination-wrap" v-if="total > pageSize">
        <el-pagination
          v-model:current-page="currentPage"
          :page-size="pageSize"
          :total="total"
          layout="prev, pager, next"
          background
          @current-change="fetchAuditList"
        />
      </div>
    </div>

    <!-- 拒绝原因弹窗 -->
    <el-dialog
      v-model="rejectDialogVisible"
      title="❌ 审核拒绝"
      width="480px"
      :close-on-click-modal="false"
      destroy-on-close
      class="reject-dialog"
    >
      <div class="reject-dialog-body">
        <p class="reject-tip">请填写拒绝原因，帮助房东了解问题所在并修改后重新提交：</p>
        <el-input
          v-model="rejectReason"
          type="textarea"
          :rows="4"
          placeholder="例如：房源图片模糊不清、地址信息不完整、租金明显偏离市场价格..."
          maxlength="200"
          show-word-limit
        />

        <!-- 快捷拒绝原因 -->
        <div class="quick-reasons">
          <span class="quick-label">快捷选择：</span>
          <el-tag
            v-for="reason in quickRejectReasons"
            :key="reason"
            effect="plain"
            class="quick-tag"
            @click="rejectReason = reason"
          >{{ reason }}</el-tag>
        </div>
      </div>

      <template #footer>
        <el-button @click="rejectDialogVisible = false">取消</el-button>
        <el-button type="danger" @click="handleReject" :disabled="!rejectReason.trim()" :loading="rejectLoading">
          确认拒绝
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { Stamp, Select, CloseBold, View } from '@element-plus/icons-vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import * as houseApi from '@/api/house'

const loading = ref(false)
const currentPage = ref(1)
const pageSize = ref(10)
const total = ref(0)
const filterCity = ref('')

interface AuditHouse {
  houseId: string
  title: string
  city: string
  district: string
  address: string
  landlordId: string
  rooms: number
  halls: number
  bathrooms: number | null
  area: number | null
  rentMonthly: number | null
  deposit: number | null
  floor: string
  orientation: string
  paymentType: number
  status: number
  coverImage: string
  description?: string
  createTime: string
  // 前端临时状态
  _approving?: boolean
  _rejecting?: boolean
}

const houses = ref<AuditHouse[]>([])

// 城市选项（从数据中动态收集）
const cityOptions = computed(() => {
  const cities = new Set(houses.value.map(h => h.city).filter(Boolean))
  return Array.from(cities).sort()
})

// 今日新增数（简单判断）
const todayCount = computed(() => {
  const today = new Date().toISOString().slice(0, 10)
  return houses.value.filter(h => h.createTime?.startsWith(today)).length
})

// 默认封面图
const defaultCover = 'data:image/svg+xml,' + encodeURIComponent(`
  <svg xmlns="http://www.w3.org/2000/svg" width="240" height="180">
    <defs><linearGradient id="g" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" style="stop-color:#f39c12;stop-opacity:0.15"/>
      <stop offset="100%" style="stop-color:#e74c3c;stop-opacity:0.25"/>
    </linearGradient></defs>
    <rect fill="url(#g)" width="240" height="180"/>
    <text x="50%" y="48%" text-anchor="middle" font-size="40" fill="#bbb">🏠</text>
    <text x="50%" y="65%" text-anchor="middle" font-size="12" fill="#999">待审核</text>
  </svg>
`)

function getCoverImage(house: AuditHouse): string {
  if (house.coverImage && house.coverImage.trim().length > 0 && !house.coverImage.startsWith('null')) return house.coverImage
  return defaultCover
}

function handleImgError(e: Event) {
  const img = e.target as HTMLImageElement
  if (img.dataset.fallbacked) return
  img.src = defaultCover
  img.dataset.fallbacked = '1'
}

function statusText(status: number): string {
  const map: Record<number, string> = { 0: '待审核', 1: '已发布', 2: '已出租', 3: '已下架', 4: '审核拒绝' }
  return map[status] || '未知'
}

function paymentTypeText(type: number): string {
  const map: Record<number, string> = { 0: '月付', 1: '季付', 2: '半年付', 3: '年付' }
  return map[type] || ''
}

function formatDate(dateStr: string): string {
  if (!dateStr) return '-'
  return new Date(dateStr).toLocaleString('zh-CN')
}

function formatTimeAgo(dateStr: string): string {
  if (!dateStr) return ''
  const diff = Date.now() - new Date(dateStr).getTime()
  const mins = Math.floor(diff / 60000)
  if (mins < 60) return `${mins}分钟前`
  const hours = Math.floor(mins / 60)
  if (hours < 24) return `${hours}小时前`
  const days = Math.floor(hours / 24)
  return `${days}天前`
}

function truncateDesc(desc: string, maxLen = 80): string {
  if (!desc) return ''
  return desc.length > maxLen ? desc.slice(0, maxLen) + '...' : desc
}

// 获取待审核列表
async function fetchAuditList(page = 1) {
  loading.value = true
  try {
    const params: any = { pageNum: page, pageSize: pageSize.value }
    if (filterCity.value) params.city = filterCity.value
    const res = await houseApi.getPendingAuditList(params)
    houses.value = (res.records || []).map((h: AuditHouse) => ({ ...h, _approving: false, _rejecting: false }))
    total.value = res.total || 0
    currentPage.value = page
  } catch (err) {
    // 已在拦截器中处理
  } finally {
    loading.value = false
  }
}

function handleFilter() {
  fetchAuditList(1)
}

// ========== 审核操作 ==========

// 审核通过
async function handleApprove(house: AuditHouse) {
  try {
    await ElMessageBox.confirm(
      `确认通过「${house.title}」的审核？`,
      '✅ 审核通过',
      { confirmButtonText: '确认通过', cancelButtonText: '取消', type: 'success' }
    )

    house._approving = true
    await houseApi.approveHouse(house.houseId)
    ElMessage.success(`「${house.title}」已通过审核，已发布 🎉`)

    // 从列表移除（动画感：重新加载当前页）
    fetchAuditList(currentPage.value)
  } catch (err: any) {
    if (err !== 'cancel') {
      // 已在拦截器中处理
    }
  } finally {
    house._approving = false
  }
}

// ========== 拒绝弹窗相关 ==========
const rejectDialogVisible = ref(false)
const rejectReason = ref('')
const rejectLoading = ref(false)
let currentRejectHouse: AuditHouse | null = null

// 快捷拒绝原因选项
const quickRejectReasons = [
  '房源图片模糊或不清晰',
  '地址信息不完整或不准确',
  '租金价格异常（过高或过低）',
  '描述包含违规内容或联系方式',
  '户型面积与实际不符',
]

function openRejectDialog(house: AuditHouse) {
  currentRejectHouse = house
  rejectReason.value = ''
  rejectDialogVisible.value = true
}

async function handleReject() {
  if (!currentRejectHouse || !rejectReason.value.trim()) return

  try {
    rejectLoading.value = true
    await houseApi.rejectHouse(currentRejectHouse.houseId, rejectReason.value.trim())
    ElMessage.success(`已拒绝「${currentRejectHouse.title}」并告知房东原因`)
    rejectDialogVisible.value = false

    // 刷新列表
    fetchAuditList(currentPage.value)
  } catch (err) {
    // 已在拦截器中处理
  } finally {
    rejectLoading.value = false
    currentRejectHouse = null
  }
}

onMounted(() => {
  fetchAuditList(1)
})
</script>

<style lang="scss" scoped>
.audit-page {
  background: var(--color-bg);
  min-height: calc(100vh - 64px - 60px);
  padding-bottom: 60px;
}

.page-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 28px 0 20px;
  flex-wrap: wrap;
  gap: 16px;

  .header-left {
    display: flex;
    align-items: baseline;
    gap: 16px;

    h1 {
      font-size: 26px;
      font-weight: 700;
      display: flex;
      align-items: center;
      gap: 10px;

      .el-icon {
        color: var(--color-primary);
      }
    }

    .pending-count {
      color: var(--color-text-muted);
      font-size: 14px;

      strong {
        color: #e67e22;
        font-size: 20px;
        font-weight: 800;
      }
    }
  }

  .filter-bar {
    display: flex;
    align-items: center;
    gap: 12px;
  }
}

// 统计卡片区
.stats-row {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 14px;
  margin-bottom: 24px;
  max-width: 520px;
}

.stat-card {
  position: relative;
  background: #fff;
  border-radius: var(--radius-md);
  padding: 18px 20px;
  box-shadow: var(--shadow-sm);
  overflow: hidden;
  transition: transform var(--transition-fast), box-shadow var(--transition-fast);

  &:hover {
    transform: translateY(-3px);
    box-shadow: var(--shadow-md);
  }

  &.stat-pending {
    border-left: 4px solid #f39c12;
    .stat-num { color: #f39c12; }
  }

  &.stat-info {
    border-left: 4px solid #3498db;
    .stat-num { color: #3498db; }
  }

  .stat-num {
    display: block;
    font-size: 30px;
    font-weight: 800;
    line-height: 1.2;
  }

  .stat-label {
    font-size: 13px;
    color: var(--color-text-secondary);
    margin-top: 4px;
  }

  .stat-icon {
    position: absolute;
    right: 16px;
    bottom: 8px;
    font-size: 36px;
    opacity: 0.15;
  }
}

// 审核列表
.audit-list {
  min-height: 300px;
}

.audit-card {
  display: flex;
  gap: 20px;
  background: #fff;
  border-radius: var(--radius-md);
  padding: 20px;
  margin-bottom: 14px;
  box-shadow: var(--shadow-sm);
  transition: all var(--transition-normal);
  align-items: flex-start;

  &:hover {
    box-shadow: var(--shadow-md);
    transform: translateX(4px);
  }
}

// 封面图
.audit-cover {
  width: 180px;
  height: 135px;
  border-radius: var(--radius-sm);
  overflow: hidden;
  flex-shrink: 0;
  position: relative;

  img {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }

  .time-badge {
    position: absolute;
    bottom: 6px;
    left: 6px;
    padding: 2px 8px;
    border-radius: 9999px;
    font-size: 11px;
    background: rgba(0, 0, 0, 0.65);
    color: #fff;
    backdrop-filter: blur(4px);
  }
}

// 房源信息
.audit-info {
  flex: 1;
  min-width: 0;

  .house-title {
    font-size: 17px;
    font-weight: 700;
    color: var(--color-text);
    margin-bottom: 6px;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .house-addr {
    font-size: 13px;
    color: var(--color-text-secondary);
    margin-bottom: 10px;
  }

  .house-tags {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-bottom: 10px;
  }

  .house-desc {
    font-size: 13px;
    color: var(--color-text-muted);
    line-height: 1.5;
    margin-bottom: 8px;
    padding: 8px 12px;
    background: var(--color-bg);
    border-radius: var(--radius-sm);
    white-space: pre-wrap;
    word-break: break-all;
  }

  .house-meta {
    display: flex;
    gap: 16px;
    font-size: 12px;
    color: var(--color-text-muted);
  }
}

// 操作按钮
.audit-actions {
  display: flex;
  flex-direction: column;
  gap: 10px;
  flex-shrink: 0;
  justify-content: center;
  padding-top: 10px;

  .el-button--primary {
    min-width: 90px;
    background: linear-gradient(135deg, #2ecc71, #27ae60) !important;
    border: none !important;

    &:hover {
      transform: translateY(-1px);
      box-shadow: 0 4px 12px rgba(46, 204, 113, 0.35);
    }
  }

  .el-button--danger.is-plain {
    &:hover {
      transform: translateY(-1px);
      box-shadow: 0 4px 12px rgba(231, 76, 60, 0.25);
    }
  }
}

.pagination-wrap {
  display: flex;
  justify-content: center;
  margin-top: 32px;
}

// 拒绝弹窗样式增强
.reject-dialog-body {
  .reject-tip {
    font-size: 14px;
    color: var(--color-text-secondary);
    margin-bottom: 14px;
    line-height: 1.5;
  }

  .quick-reasons {
    margin-top: 14px;
    display: flex;
    align-items: flex-start;
    flex-wrap: wrap;
    gap: 8px;

    .quick-label {
      font-size: 13px;
      color: var(--color-text-muted);
      white-space: nowrap;
      margin-top: 4px;
    }

    .quick-tag {
      cursor: pointer;
      transition: all 0.2s;

      &:hover {
        color: #e74c3c;
        border-color: #e74c3c;
        transform: scale(1.03);
      }
    }
  }
}
</style>
