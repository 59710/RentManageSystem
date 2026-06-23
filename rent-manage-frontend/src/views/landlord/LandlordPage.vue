<template>
  <div class="landlord-page">
    <div class="page-container">
      <!-- 页面头部 -->
      <div class="page-header">
        <div class="header-left">
          <h1>
            <el-icon><OfficeBuilding /></el-icon>
            我的房源
          </h1>
          <span class="house-count">共 {{ total }} 套</span>
        </div>
        <el-button type="primary" @click="$router.push('/landlord/publish')">
          <el-icon><Plus /></el-icon>
          发布新房源
        </el-button>
      </div>

      <!-- 统计卡片 -->
      <div class="stats-row">
        <div class="stat-card" v-for="(item, index) in statsData" :key="index"
             :style="{ '--card-color': item.color }">
          <span class="stat-num">{{ item.count }}</span>
          <span class="stat-label">{{ item.label }}</span>
          <span class="stat-icon">{{ item.icon }}</span>
        </div>
      </div>

      <!-- 房源列表 -->
      <div v-loading="loading" class="house-list">
        <template v-if="houses.length > 0">
          <div v-for="house in houses" :key="house.houseId" class="house-card" :class="{ rejected: house.status === 4 }">
            <!-- 封面图 -->
            <div class="house-cover">
              <img :src="getCoverImage(house)" :alt="house.title" @error="handleImgError" :data-house-id="house.houseId" :data-title="house.title" />
              <span class="status-badge" :class="'status-' + house.status">
                {{ statusText(house.status) }}
              </span>
            </div>

            <!-- 信息 -->
            <div class="house-info">
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

              <div class="house-meta">
                <span>👁 {{ house.viewCount || 0 }} 浏览</span>
                <span>❤️ {{ house.favoriteCount || 0 }} 收藏</span>
                <span>📅 发布于 {{ formatDate(house.createTime) }}</span>
              </div>

              <!-- 审核拒绝提示（仅 status=4 时显示，内联在信息区） -->
              <div v-if="house.status === 4" class="rejection-notice">
                <div class="rejection-head">
                  <span class="rejection-icon">⚠️</span>
                  <span class="rejection-label">审核未通过</span>
                  <span v-if="house.auditTime" class="rejection-time">· {{ formatDate(house.auditTime) }}</span>
                </div>
                <p v-if="house.auditRemark" class="rejection-remark">{{ house.auditRemark }}</p>
              </div>
            </div>

            <!-- 操作按钮 -->
            <div class="house-actions">
              <el-button text type="info" @click="$router.push(`/house/${house.houseId}`)">
                <el-icon><View /></el-icon>查看
              </el-button>
              <el-button text type="primary" @click="$router.push({ path: '/landlord/publish', query: { id: house.houseId } })">
                <el-icon><EditPen /></el-icon>编辑
              </el-button>
              <el-button
                text
                :type="house.status === 3 ? 'success' : 'warning'"
                @click="handleToggleStatus(house)"
                :disabled="house.status === 2 || house.status === 0 || house.status === 4"
              >
                <el-icon><component :is="house.status === 3 ? 'Top' : 'Bottom'" /></el-icon>
                {{ house.status === 3 ? '重新上架' : '下架' }}
              </el-button>
              <el-button
                text
                type="danger"
                @click="handleDelete(house)"
                :disabled="house.status === 1 || house.status === 2"
                :title="house.status === 1 || house.status === 2 ? '需先下架才能删除' : ''"
              >
                <el-icon><Delete /></el-icon>删除
              </el-button>
            </div>

          </div>
        </template>

        <el-empty v-else-if="!loading" description="还没有发布任何房源，快去发布吧！🏠">
          <el-button type="primary" @click="$router.push('/landlord/publish')">
            立即发布
          </el-button>
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
          @current-change="fetchHouses"
        />
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import {
  OfficeBuilding,
  Plus,
  EditPen,
  Delete,
  Top,
  Bottom,
  View,
} from '@element-plus/icons-vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import * as houseApi from '@/api/house'

const loading = ref(false)
const currentPage = ref(1)
const pageSize = ref(10)
const total = ref(0)

interface HouseItem {
  houseId: string
  title: string
  city: string
  district: string
  address: string
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
  viewCount: number | null
  favoriteCount: number | null
  createTime: string
  // 审核相关字段
  auditRemark?: string | null
  auditTime?: string | null
}

const houses = ref<HouseItem[]>([])

// 获取封面图（过滤无效值）
function getCoverImage(house: HouseItem): string {
  if (house.coverImage && house.coverImage.trim().length > 0 && !house.coverImage.startsWith('null')) return house.coverImage
  return defaultCover
}

/** 图片加载失败时自动替换 */
function handleImgError(e: Event) {
  const img = e.target as HTMLImageElement
  if (img.dataset.fallbacked) return
  img.src = defaultCover
  img.dataset.fallbacked = '1'
}

// 默认封面图（渐变背景）
const defaultCover = 'data:image/svg+xml,' + encodeURIComponent(`
  <svg xmlns="http://www.w3.org/2000/svg" width="240" height="180">
    <defs><linearGradient id="g" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" style="stop-color:#ff6b35;stop-opacity:0.15"/>
      <stop offset="100%" style="stop-color:#4facfe;stop-opacity:0.25"/>
    </linearGradient></defs>
    <rect fill="url(#g)" width="240" height="180"/>
    <text x="50%" y="48%" text-anchor="middle" font-size="40" fill="#bbb">🏠</text>
    <text x="50%" y="65%" text-anchor="middle" font-size="12" fill="#999">暂无图片</text>
  </svg>
`)

// 统计数据
const statsData = computed(() => [
  { icon: '📊', label: '全部', count: total.value, color: '#4facfe' },
  { icon: '⏳', label: '待审核', count: houses.value.filter(h => h.status === 0).length, color: '#f39c12' },
  { icon: '✅', label: '已发布', count: houses.value.filter(h => h.status === 1).length, color: '#2ecc71' },
  { icon: '🏷️', label: '已出租', count: houses.value.filter(h => h.status === 2).length, color: '#9b59b6' },
  { icon: '📦', label: '已下架', count: houses.value.filter(h => h.status === 3).length, color: '#b2bec3' },
  { icon: '⚠️', label: '审核拒绝', count: houses.value.filter(h => h.status === 4).length, color: '#e74c3c' },
])

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
  return new Date(dateStr).toLocaleDateString('zh-CN')
}

async function fetchHouses(page = 1) {
  loading.value = true
  try {
    const res = await houseApi.getMyHouses(page, pageSize.value)
    houses.value = res.records || []
    total.value = res.total || 0
    currentPage.value = page
  } catch (err) {
    // 已在拦截器中处理
  } finally {
    loading.value = false
  }
}

async function handleToggleStatus(house: HouseItem) {
  const isOff = house.status === 3 // 当前是下架状态 → 操作是上架
  const action = isOff ? '重新上架' : '下架'
  const confirmText = isOff
    ? '上架后需等待管理员审核通过才能公开显示'
    : '下架后该房源将不再对租客展示'

  try {
    await ElMessageBox.confirm(
      `${confirmText}`,
      `确认${action}`,
      { confirmButtonText: isOff ? '提交审核' : '确认下架', cancelButtonText: '取消', type: 'warning' }
    )

    if (isOff) {
      await houseApi.onShelfHouse(house.houseId)
      ElMessage.success('已提交上架审核 ⏳ 请耐心等待审核结果')
    } else {
      await houseApi.offShelfHouse(house.houseId)
      ElMessage.success('房源已下架 📦 租客端将不再显示')
    }
    fetchHouses(currentPage.value)
  } catch (err: any) {
    if (err !== 'cancel') {
      // 已在 Axios 拦截器中统一处理错误提示
    }
  }
}

async function handleDelete(house: HouseItem) {
  // 已发布/已出租的不能直接删，给个明确提示
  if (house.status === 1 || house.status === 2) {
    ElMessage.warning('请先将房源下架后才能删除 📦')
    return
  }

  try {
    await ElMessageBox.confirm(
      '确定要删除该房源吗？删除后将无法恢复！',
      '⚠️ 确认删除',
      { confirmButtonText: '确认删除', cancelButtonText: '取消', type: 'error', confirmButtonClass: 'el-button--danger' }
    )

    await houseApi.deleteHouse(house.houseId)
    ElMessage.success('房源已删除 🗑️')

    // 如果当前页删完了且不是第一页，回到上一页
    if (houses.value.length <= 1 && currentPage.value > 1) {
      fetchHouses(currentPage.value - 1)
    } else {
      fetchHouses(currentPage.value)
    }
  } catch (err: any) {
    if (err !== 'cancel') {
      // 已在拦截器中处理
    }
  }
}

onMounted(() => {
  fetchHouses(1)
})
</script>

<style lang="scss" scoped>
.landlord-page {
  background: var(--color-bg);
  min-height: calc(100vh - 64px - 60px);
  padding-bottom: 60px;
}

.page-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 28px 0 20px;

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

    .house-count {
      color: var(--color-text-muted);
      font-size: 14px;
    }
  }

  .el-button--primary {
    height: 40px !important;
    border-radius: var(--radius-sm) !important;
    background: linear-gradient(135deg, var(--color-primary), #ff9a56) !important;
    border: none !important;
    
    &:hover {
      transform: translateY(-1px);
      box-shadow: var(--shadow-primary);
    }
  }
}

// 统计卡片区
.stats-row {
  display: grid;
  grid-template-columns: repeat(6, 1fr);
  gap: 14px;
  margin-bottom: 24px;
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

  .stat-num {
    display: block;
    font-size: 30px;
    font-weight: 800;
    color: var(--card-color);
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

// 房源列表
.house-list {
  min-height: 300px;
}

.house-card {
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

.house-cover {
  width: 200px;
  height: 150px;
  border-radius: var(--radius-sm);
  overflow: hidden;
  flex-shrink: 0;
  position: relative;

  img {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }

  .status-badge {
    position: absolute;
    top: 10px;
    left: 10px;
    padding: 3px 10px;
    border-radius: 9999px;
    font-size: 12px;
    font-weight: 600;
    color: #fff;

    &.status-0 { background: linear-gradient(135deg, #f39c12, #e67e22); }
    &.status-1 { background: linear-gradient(135deg, #2ecc71, #27ae60); }
    &.status-2 { background: linear-gradient(135deg, #9b59b6, #8e44ad); }
    &.status-3 { background: linear-gradient(135deg, #95a5a6, #7f8c8d); }
    &.status-4 { background: linear-gradient(135deg, #e74c3c, #c0392b); }
  }
}

.house-info {
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

  .house-meta {
    display: flex;
    gap: 16px;
    font-size: 12px;
    color: var(--color-text-muted);
  }
}

.house-actions {
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 4px;
  flex-shrink: 0;
  
  .el-button {
    font-size: 13px;
  }
}

.pagination-wrap {
  display: flex;
  justify-content: center;
  margin-top: 32px;
}

// 审核拒绝房源卡片（左侧红色强调线）
.house-card.rejected {
  border-left: 4px solid #e74c3c;
  background: linear-gradient(90deg, #fef5f5 0%, #fff 8%);
}

// 审核拒绝内联提示
.rejection-notice {
  margin-top: 10px;
  padding: 8px 12px;
  background: #fef2f2;
  border-radius: var(--radius-sm);
  border-left: 3px solid #ef4444;

  .rejection-head {
    display: flex;
    align-items: center;
    gap: 6px;
    margin-bottom: 4px;

    .rejection-icon {
      font-size: 14px;
    }

    .rejection-label {
      font-size: 13px;
      font-weight: 700;
      color: #dc2626;
    }

    .rejection-time {
      font-size: 12px;
      color: var(--color-text-muted);
    }
  }

  .rejection-remark {
    font-size: 13px;
    color: #991b1b;
    line-height: 1.5;
    margin: 0;
  }
}
</style>
