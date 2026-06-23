<template>
  <div class="landlord-orders-page">
    <!-- 页头 -->
    <div class="page-header page-container">
      <h1 class="page-title">
        <el-icon><Bell /></el-icon>
        租房申请管理
      </h1>
      <p class="page-subtitle">处理租客的租房申请，确认后房源将变为已出租状态</p>
    </div>

    <!-- 统计卡片 -->
    <div class="stats-row page-container">
      <div
        v-for="(stat, idx) in stats"
        :key="idx"
        class="stat-card"
        :class="{ active: currentTab === stat.key }"
        @click="currentTab = stat.key; fetchOrders()"
      >
        <div class="stat-icon" :style="{ background: stat.color }">{{ stat.icon }}</div>
        <div class="stat-info">
          <strong>{{ stat.count }}</strong>
          <span>{{ stat.label }}</span>
        </div>
        <!-- 待确认有新订单时闪烁提示 -->
        <span v-if="stat.key === 'pending' && stat.count > 0" class="pulse-dot"></span>
      </div>
    </div>

    <!-- 订单列表 -->
    <div class="order-list page-container">
      <div v-if="loading" style="padding: 40px;">
        <el-skeleton :rows="4" animated />
      </div>

      <template v-else-if="orders.length > 0">
        <div class="order-card glass-effect" v-for="order in filteredOrders" :key="order.orderId">
          <!-- 卡片头部：状态 + 订单编号 + 时间 -->
          <div class="card-head">
            <div class="head-left">
              <el-tag
                :type="statusTagMap[order.status]?.type || 'info'"
                size="large"
                effect="dark"
                round
                :style="{ background: statusTagMap[order.status]?.color || '' }"
              >
                {{ statusTagMap[order.status]?.label || '未知' }}
              </el-tag>
              <span class="order-no">订单号：{{ order.orderNo }}</span>
            </div>
            <span class="order-time">{{ formatRelativeTime(order.createTime) }}</span>
          </div>

          <!-- 卡片内容：租客信息 + 房源信息 + 费用 -->
          <div class="card-body">
            <!-- 左侧：租客信息 + 房源信息 -->
            <div class="info-col">
              <router-link :to="`/house/${order.houseId}`" class="house-link">
                <h3 class="house-title">
                  <el-icon><House /></el-icon>
                  {{ order.title || '房源标题' }}
                </h3>
              </router-link>

              <div class="tenant-info-box">
                <div class="info-row">
                  <el-icon><User /></el-icon>
                  <span>申请人：<strong>{{ order.contactName }}</strong></span>
                  <el-tag size="small" type="info" round>{{ order.contactPhone }}</el-tag>
                </div>
                <div class="info-row">
                  <el-icon><Calendar /></el-icon>
                  <span>期望租期：<b>{{ order.startDate }}</b> 至 <b>{{ order.endDate }}</b></span>
                </div>
                <div class="remark-text" v-if="order.remark">
                  📝 {{ order.remark }}
                </div>
              </div>
            </div>

            <!-- 右侧：费用信息 -->
            <div class="fee-col">
              <div class="fee-item">
                <span>月租金</span>
                <strong>¥{{ order.rentMonthly?.toLocaleString() }}/月</strong>
              </div>
              <div class="fee-item">
                <span>押金</span>
                <strong>¥{{ order.deposit?.toLocaleString() }}</strong>
              </div>
              <div class="fee-divider"></div>
              <div class="fee-total">
                <span>总金额</span>
                <strong class="total-num">¥{{ order.totalAmount?.toLocaleString() }}</strong>
              </div>
            </div>
          </div>

          <!-- 操作按钮 -->
          <div class="card-actions">
            <router-link :to="`/house/${order.houseId}`">
              <el-button size="small" plain>
                查看房源
              </el-button>
            </router-link>

            <!-- 待确认状态：显示 确认/拒绝 按钮 -->
            <template v-if="order.status === 0">
              <el-button type="primary" size="small" @click="handleConfirm(order)" :loading="confirmingId === order.orderId">
                <el-icon><Select /></el-icon>
                确认出租
              </el-button>
              <el-popconfirm title="确定要拒绝此租房申请吗？" @confirm="handleReject(order)">
                <template #reference>
                  <el-button type="danger" size="small" plain>拒绝</el-button>
                </template>
              </el-popconfirm>
            </template>

            <!-- 已确认 -->
            <el-tag v-else-if="order.status === 1" type="success" size="small" effect="light" round>
              ✅ 已确认 — 房源已出租
            </el-tag>
            <!-- 已完成 -->
            <el-tag v-else-if="order.status === 2" type="info" size="small" effect="light" round>
              🏠 已完成
            </el-tag>
            <!-- 租客已取消 -->
            <el-tag v-else-if="order.status === 3" type="info" size="small" effect="light" round>
              ❌ 租客已取消
            </el-tag>
            <!-- 房东已拒绝 -->
            <el-tag v-else-if="order.status === 4" type="danger" size="small" effect="light" round>
              ❌ 已拒绝
            </el-tag>
          </div>
        </div>

        <!-- 分页 -->
        <div class="pagination-wrap" v-if="total > pageSize">
          <el-pagination
            v-model:current-page="currentPage"
            :page-size="pageSize"
            :total="total"
            layout="prev, pager, next"
            @current-change="fetchOrders"
            background
          />
        </div>
      </template>

      <!-- 空状态 -->
      <div v-else class="empty-state">
        <el-empty description="暂无租房申请">
          <p class="empty-hint">当租客对你的房源发起租房申请时，会显示在这里</p>
        </el-empty>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, computed, onMounted } from 'vue'
import { Bell, User, Calendar, House, Select } from '@element-plus/icons-vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import { getReceivedOrders, confirmOrder, rejectOrder } from '@/api/order'
import type { RentOrder } from '@/api/order'

const loading = ref(false)
const orders = ref<RentOrder[]>([])
const currentPage = ref(1)
const pageSize = ref(10)
const total = ref(0)
const confirmingId = ref<string | null>(null)
const currentTab = ref('all')

// 统计卡片数据
const stats = reactive([
  { key: 'all', label: '全部', icon: '📋', color: 'linear-gradient(135deg,#667eea,#764ba2)', count: 0 },
  { key: 'pending', label: '待确认', icon: '⏳', color: 'linear-gradient(135deg,#ff6b35,#ff9a56)', count: 0 },
  { key: 'confirmed', label: '已确认', icon: '✅', color: 'linear-gradient(135deg,#11998e,#38ef7d)', count: 0 },
])

// 状态映射（房东视角）
const statusTagMap: Record<number, { label: string; type: string; color: string }> = {
  0: { label: '待确认', type: 'warning', color: 'linear-gradient(135deg,#ff6b35,#ff9a56)' },
  1: { label: '租住中', type: 'success', color: 'linear-gradient(135deg,#11998e,#38ef7d)' },
  2: { label: '已完成', type: '', color: 'linear-gradient(135deg,#4facfe,#00f2fe)' },
  3: { label: '租客已取消', type: 'info', color: 'linear-gradient(135deg,#95a5a6,#7f8c8d)' },
  4: { label: '已拒绝', type: 'danger', color: 'linear-gradient(135deg,#e74c3c,#c0392b)' },
}

// 前端筛选后的订单列表
const filteredOrders = computed(() => {
  if (currentTab.value === 'all') return orders.value
  const filterMap: Record<string, number> = { pending: 0, confirmed: 1, rejected: 4 }
  const targetStatus = filterMap[currentTab.value]
  if (targetStatus !== undefined) {
    return orders.value.filter(o => o.status === targetStatus)
  }
  return orders.value
})

/** 获取收到的订单 */
async function fetchOrders() {
  loading.value = true
  try {
    const res = await getReceivedOrders(currentPage.value, pageSize.value)
    const list: RentOrder[] = res.records || res.list || []
    orders.value = list
    total.value = res.total || 0

    // 更新统计
    stats[0].count = total.value
    stats[1].count = list.filter(o => o.status === 0).length
    stats[2].count = list.filter(o => o.status === 1).length
  } catch {
    ElMessage.error('获取订单列表失败')
  } finally {
    loading.value = false
  }
}

/** 确认订单（房源变已出租） */
async function handleConfirm(order: RentOrder) {
  try {
    await ElMessageBox.confirm(
      `确认接受「${order.contactName}」对「${order.title}」的租房申请？\n\n⚠️ 确认后该房源将变为【已出租】状态，不再在找房页面显示。`,
      '确认出租',
      { confirmButtonText: '确认出租', cancelButtonText: '再考虑下', type: 'success' }
    )
    confirmingId.value = order.orderId
    await confirmOrder(order.orderId)
    ElMessage.success('🎉 已确认！房源已标记为出租')
    fetchOrders()
  } catch (err: any) {
    if (err !== 'cancel') ElMessage.error(err?.message || '操作失败')
  } finally {
    confirmingId.value = null
  }
}

/** 拒绝订单 */
async function handleReject(order: RentOrder) {
  try {
    await rejectOrder(order.orderId)
    ElMessage.success(`已拒绝 ${order.orderNo}`)
    fetchOrders()
  } catch {
    // error handled by interceptor
  }
}

/** 格式化相对时间 */
function formatRelativeTime(dateStr: string): string {
  if (!dateStr) return ''
  const date = new Date(dateStr)
  const now = new Date()
  const diffMs = now.getTime() - date.getTime()
  const diffMin = Math.floor(diffMs / 60000)
  if (diffMin < 1) return '刚刚'
  if (diffMin < 60) return `${diffMin}分钟前`
  const diffHour = Math.floor(diffMin / 60)
  if (diffHour < 24) return `${diffHour}小时前`
  const diffDay = Math.floor(diffHour / 24)
  if (diffDay < 30) return `${diffDay}天前`
  return dateStr.substring(0, 10)
}

onMounted(() => {
  fetchOrders()
})
</script>

<style lang="scss" scoped>
.landlord-orders-page {
  min-height: calc(100vh - 64px);
  background: var(--color-bg);
  padding-bottom: 40px;
}

// ==================== 页头 ====================
.page-header {
  padding-top: 24px;
  padding-bottom: 8px;

  .page-title {
    display: flex;
    align-items: center;
    gap: 10px;
    font-size: 24px;
    font-weight: 800;
    margin: 0 0 6px;

    .el-icon { font-size: 26px; color: var(--color-primary); }
  }

  .page-subtitle {
    font-size: 14px;
    color: var(--color-text-muted);
    margin: 0;
  }
}

// ==================== 统计卡片 ====================
.stats-row {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px;
  padding-top: 16px;
  padding-bottom: 16px;
}

.stat-card {
  position: relative;
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 18px 20px;
  background: white;
  border-radius: var(--radius-md);
  cursor: pointer;
  transition: all 0.25s ease;
  border: 2px solid transparent;

  &:hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 20px rgba(0, 0, 0, 0.08);
  }

  &.active {
    border-color: var(--color-primary);
    box-shadow: 0 4px 15px rgba(255, 107, 53, 0.15);

    strong { color: var(--color-primary); }
  }

  .stat-icon {
    width: 48px;
    height: 48px;
    border-radius: 14px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 22px;
    flex-shrink: 0;
  }

  .stat-info {
    display: flex;
    flex-direction: column;

    strong {
      font-size: 24px;
      font-weight: 800;
      line-height: 1.2;
    }

    span {
      font-size: 12px;
      color: var(--color-text-muted);
    }
  }

  .pulse-dot {
    position: absolute;
    top: 12px;
    right: 12px;
    width: 10px;
    height: 10px;
    border-radius: 50%;
    background: #ff4757;
    animation: pulse 1.5s infinite;

    @keyframes pulse {
      0%, 100% { opacity: 1; transform: scale(1); }
      50% { opacity: 0.5; transform: scale(1.3); }
    }
  }
}

// ==================== 订单列表 ====================
.order-list {
  padding-top: 12px;
}

.order-card {
  background: white;
  border-radius: var(--radius-lg);
  margin-bottom: 16px;
  overflow: hidden;
  transition: transform 0.25s ease, box-shadow 0.25s ease;

  &:hover {
    transform: translateY(-2px);
    box-shadow: 0 8px 30px rgba(0, 0, 0, 0.08);
  }
}

.card-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 22px;
  border-bottom: 1px solid var(--color-border-light);

  .head-left {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .order-no {
    font-size: 13px;
    color: var(--color-text-muted);
    font-family: monospace;
  }

  .order-time {
    font-size: 12px;
    color: var(--color-text-muted);
  }
}

.card-body {
  display: flex;
  justify-content: space-between;
  gap: 24px;
  padding: 20px 22px;
}

.info-col {
  flex: 1;
  min-width: 0;

  .house-link {
    text-decoration: none;
    color: inherit;
    display: block;
    margin-bottom: 14px;

    &:hover .house-title { color: var(--color-primary); }
  }

  .house-title {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 17px;
    font-weight: 700;
    color: var(--color-text-primary);
    margin: 0;
    transition: color 0.2s;

    .el-icon { color: var(--color-primary); opacity: 0.7; }
  }
}

.tenant-info-box {
  background: rgba(255, 107, 53, 0.04);
  border: 1px solid rgba(255, 107, 53, 0.1);
  border-radius: var(--radius-sm);
  padding: 12px 16px;

  .info-row {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 13px;
    color: var(--color-text-secondary);
    margin: 6px 0;

    .el-icon { color: var(--color-primary); font-size: 15px; }
    strong { color: var(--color-text-primary); }
    b { color: var(--color-primary); }
  }

  .remark-text {
    font-size: 13px;
    color: var(--color-text-muted);
    margin-top: 8px;
    padding-top: 8px;
    border-top: 1px dashed rgba(255,107,53,0.15);
  }
}

.fee-col {
  min-width: 180px;
  text-align: right;
  padding: 12px 16px;
  background: var(--color-bg);
  border-radius: var(--radius-sm);

  .fee-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 4px 0;
    font-size: 13px;

    span { color: var(--color-text-muted); }
    strong { color: var(--color-text-secondary); }
  }

  .fee-divider {
    height: 1px;
    background: var(--color-border-light);
    margin: 8px 0;
  }

  .fee-total {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding-top: 6px;

    span { font-weight: 700; color: var(--color-text-primary); font-size: 14px; }
  }

  .total-num {
    font-size: 20px !important;
    font-weight: 800 !important;
    color: var(--color-primary) !important;
  }
}

.card-actions {
  display: flex;
  gap: 10px;
  padding: 14px 22px;
  border-top: 1px solid var(--color-border-light);
}

.pagination-wrap {
  display: flex;
  justify-content: center;
  padding: 24px 0 12px;
}

.empty-state {
  padding: 80px 0;

  .empty-hint {
    font-size: 13px;
    color: var(--color-text-muted);
    margin-top: 12px;
  }
}
</style>
