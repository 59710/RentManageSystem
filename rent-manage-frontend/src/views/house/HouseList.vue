<template>
  <div class="house-list-page">
    <!-- 搜索筛选区 -->
    <section class="filter-section glass-effect">
      <div class="filter-inner page-container">
        <!-- 搜索框 -->
        <el-input
          v-model="query.keyword"
          placeholder="搜索房源名称、地址..."
          :prefix-icon="Search"
          clearable
          class="search-input"
          @keyup.enter="handleSearch"
        />

        <!-- 筛选项 -->
        <div class="filter-items">
          <!-- 城市 -->
          <el-select v-model="query.city" placeholder="城市" clearable>
            <el-option label="广州" value="广州" />
            <el-option label="深圳" value="深圳" />
            <el-option label="北京" value="北京" />
            <el-option label="上海" value="上海" />
          </el-select>

          <!-- 区域 -->
          <el-select v-model="query.district" placeholder="区域" clearable>
            <el-option label="天河" value="天河" />
            <el-option label="海珠" value="海珠" />
            <el-option label="番禺" value="番禺" />
            <el-option label="白云" value="白云" />
            <el-option label="越秀" value="越秀" />
            <el-option label="南山" value="南山" />
            <el-option label="福田" value="福田" />
          </el-select>

          <!-- 价格区间 -->
          <el-select v-model="query.priceRange" placeholder="租金范围" clearable>
            <el-option label="¥2000以下" value="0-2000" />
            <el-option label="¥2000-3500" value="2000-3500" />
            <el-option label="¥3500-5000" value="3500-5000" />
            <el-option label="¥5000以上" value="5000-999999" />
          </el-select>

          <!-- 户型 -->
          <el-select v-model="query.roomCount" placeholder="户型" clearable>
            <el-option label="1室" :value="1" />
            <el-option label="2室" :value="2" />
            <el-option label="3室+" :value="3" />
            <el-option label="4室+" :value="4" />
          </el-select>

          <!-- 排序 -->
          <el-select v-model="query.sortBy" placeholder="排序方式">
            <el-option label="最新发布" value="newest" />
            <el-option label="价格从低到高" value="price_asc" />
            <el-option label="价格从高到低" value="price_desc" />
          </el-select>

          <el-button type="primary" @click="handleSearch">
            <el-icon><Search /></el-icon> 搜索
          </el-button>
        </div>
      </div>
    </section>

    <!-- 结果统计 -->
    <div class="result-header page-container">
      <p>
        共找到
        <span class="count-text">{{ total }}</span>
        套房源
      </p>
    </div>

    <!-- 房源列表 -->
    <div class="list-container page-container">
      <!-- 加载中 -->
      <div v-if="loading" class="loading-grid">
        <el-skeleton v-for="i in 6" :key="i" :rows="5" animated />
      </div>

      <!-- 卡片列表 -->
      <div v-else-if="houseList.length > 0" class="house-list">
        <div
          v-for="house in houseList"
          :key="house.houseId"
          class="house-card"
          @click="$router.push(`/house/${house.houseId}`)"
        >
          <div class="card-image">
            <img :src="getCoverImage(house)" :alt="house.title" @error="handleImgError" :data-house-id="house.houseId" :data-title="house.title" />
            <div class="image-overlay">
              <span class="status-badge" :class="'status-' + house.status">
                {{ getStatusText(house.status) }}
              </span>
            </div>
          </div>
          <div class="card-body">
            <h3 class="card-title">{{ house.title }}</h3>
            <p class="card-location">
              <el-icon><Location /></el-icon>
              {{ house.city }}{{ house.district }} · {{ house.address }}
            </p>
            <div class="card-tags">
              <el-tag size="small">{{ house.area }}m²</el-tag>
              <el-tag size="small" type="info">{{ house.rooms }}室{{ house.halls }}厅</el-tag>
              <el-tag size="small" type="info">{{ house.orientation || '-' }}</el-tag>
              <el-tag size="small" type="info">{{ house.floor || '-' }}</el-tag>
            </div>
            <div class="card-bottom">
              <span class="price">¥{{ house.rentMonthly?.toLocaleString() || '-' }}<small>/月</small></span>
              <span class="time">{{ formatTime(house.createTime) }}</span>
            </div>
          </div>
        </div>
      </div>

      <!-- 空状态 -->
      <el-empty v-else description="没有找到符合条件的房源 😢">
        <el-button type="primary" @click="resetFilters">清空筛选条件</el-button>
      </el-empty>
    </div>

    <!-- 分页 -->
    <div v-if="total > 0" class="pagination-wrap page-container">
      <el-pagination
        v-model:current-page="query.page"
        v-model:page-size="query.size"
        :total="total"
        :page-sizes="[6, 12, 18]"
        layout="total, sizes, prev, pager, next, jumper"
        background
        @current-change="fetchHouses"
        @size-change="fetchHouses"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { Search, Location } from '@element-plus/icons-vue'
import { getHouseList } from '@/api/house'
import type { HouseInfo } from '@/api/house'

const route = useRoute()
const router = useRouter()

// 查询参数（从 URL 恢复筛选状态）
const query = reactive({
  keyword: (route.query.keyword as string) || '',
  city: (route.query.city as string) || '',
  district: (route.query.district as string) || '',
  priceRange: (route.query.priceRange as string) || '',
  roomCount: route.query.roomCount ? Number(route.query.roomCount) : (null as number | null),
  sortBy: (route.query.sortBy as string) || 'newest',
  page: route.query.page ? Number(route.query.page) : 1,
  size: route.query.size ? Number(route.query.size) : 12,
})

// 数据状态
const houseList = ref<HouseInfo[]>([])
const total = ref(0)
const loading = ref(false)

async function fetchHouses() {
  loading.value = true
  try {
    const params: Record<string, any> = {
      page: query.page,
      size: query.size,
    }

    if (query.keyword) params.keyword = query.keyword
    if (query.city) params.city = query.city
    if (query.district) params.district = query.district

    // 价格区间处理
    if (query.priceRange) {
      const [min, max] = query.priceRange.split('-').map(Number)
      params.minPrice = min
      params.maxPrice = max
    }

    if (query.roomCount) params.roomCount = query.roomCount
    if (query.sortBy && query.sortBy !== 'newest') params.sortBy = query.sortBy

    const res = await getHouseList(params)
    houseList.value = res.records || res.list || []
    total.value = res.total || res.totalSize || houseList.value.length
  } catch {
    houseList.value = []
    total.value = 0
  } finally {
    loading.value = false
  }
}

function handleSearch() {
  query.page = 1
  const q: Record<string, any> = {}
  if (query.keyword) q.keyword = query.keyword
  if (query.city) q.city = query.city
  if (query.district) q.district = query.district
  if (query.priceRange) q.priceRange = query.priceRange
  if (query.roomCount) q.roomCount = query.roomCount
  if (query.sortBy && query.sortBy !== 'newest') q.sortBy = query.sortBy
  router.replace({ query: q })
  fetchHouses()
}

function resetFilters() {
  query.keyword = ''
  query.city = ''
  query.district = ''
  query.priceRange = ''
  query.roomCount = null
  query.sortBy = 'newest'
  query.page = 1
  router.replace({ query: {} })
  fetchHouses()
}

function getCoverImage(house: HouseInfo): string {
  if (house.coverImage && house.coverImage.trim().length > 0 && !house.coverImage.startsWith('null')) return house.coverImage
  const colors = ['#ff6b35', '#4facfe', '#2ecc71', '#9b59b6', '#f39c12', '#e74c3c']
  const idx = (house.houseId?.charCodeAt(0) || 0) % colors.length
  const color = colors[idx]
  const text = (house.title || '🏠').substring(0, 6)
  const info = `${house.rooms || '-'}室${house.halls || '-'}厅 · ${house.area || '?'}m²`
  return genPlaceholder(color, text, info)
}

function handleImgError(e: Event) {
  const img = e.target as HTMLImageElement
  if (img.dataset.fallbacked) return
  const title = img.dataset.title || '🏠'
  const colors = ['#ff6b35', '#4facfe', '#2ecc71', '#9b59b6', '#f39c12']
  const color = colors[Math.floor(Math.random() * colors.length)]
  img.src = genPlaceholder(color, title.substring(0, 6), '')
  img.dataset.fallbacked = '1'
}

function genPlaceholder(color: string, text: string, info: string): string {
  const iSvg = info ? `<text x='50%' y='80%' dominant-baseline='middle' text-anchor='middle' font-size='14' fill='white' opacity='0.7'>${info}</text>` : ''
  return `data:image/svg+xml,${encodeURIComponent(`<svg xmlns='http://www.w3.org/2000/svg' width='400' height='300'><defs><linearGradient id='g' x1='0' y1='0' x2='1' y2='1'><stop offset='0%' stop-color='${color}'/><stop offset='100%' stop-color='${color}88'/></linearGradient></defs><rect fill='url(%23g)' width='400' height='300'/><text x='50%' y='42%' dominant-baseline='middle' text-anchor='middle' font-size='48' fill='white' opacity='0.9'>🏠</text><text x='50%' y='65%' dominant-baseline='middle' text-anchor='middle' font-size='18' fill='white' font-weight='bold'>${text}</text>${iSvg}</svg>`)}`
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

function formatTime(time: string): string {
  const d = new Date(time)
  const now = new Date()
  const diff = now.getTime() - d.getTime()
  const days = Math.floor(diff / (1000 * 60 * 60 * 24))
  if (days === 0) return '今天发布'
  if (days === 1) return '昨天发布'
  if (days < 7) return `${days}天前`
  return time.substring(5, 10)
}

onMounted(() => {
  fetchHouses()
})
</script>

<style lang="scss" scoped>
.house-list-page {
  min-height: calc(100vh - 64px);
  background: var(--color-bg);
}

// ==================== 筛选栏 ====================
.filter-section {
  position: sticky;
  top: 64px;
  z-index: 100;
  padding: 16px 0;
  border-bottom: 1px solid var(--color-border-light);

  .filter-inner {
    display: flex;
    align-items: center;
    gap: 16px;
    flex-wrap: wrap;
  }

  .search-input {
    width: 280px;
  }

  .filter-items {
    display: flex;
    gap: 12px;
    flex-wrap: wrap;

    :deep(.el-select) {
      width: 140px;
    }

    .el-button {
      white-space: nowrap;
    }
  }
}

// ==================== 统计栏 ====================
.result-header {
  padding: 20px 24px 0;

  p {
    font-size: 14px;
    color: var(--color-text-secondary);

    .count-text {
      font-size: 22px;
      font-weight: 800;
      color: var(--color-primary);
      margin: 0 4px;
    }
  }
}

// ==================== 列表容器 ====================
.list-container {
  padding-top: 16px;
}

.loading-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 20px;
}

// 房源卡片（列表式）
.house-list {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 24px;
}

.house-card {
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

        &.status-0 {
          background: #f39c12;  // 待审核
        }
        &.status-1 {
          background: linear-gradient(135deg, #2ecc71, #27ae60);  // 已发布
        }
        &.status-2 {
          background: #3498db;  // 已出租
        }
        &.status-3 {
          background: #95a5a6;  // 已下架
        }
        &.status-4 {
          background: #e74c3c;  // 审核拒绝
        }
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
  }
}

// 分页
.pagination-wrap {
  padding: 32px 0;
  display: flex;
  justify-content: center;
}
</style>
