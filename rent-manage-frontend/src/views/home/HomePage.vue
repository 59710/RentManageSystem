<template>
  <div class="home-page">
    <!-- Hero 区域 -->
    <section class="hero-section">
      <div class="hero-bg-pattern"></div>
      <div class="hero-content fade-in-up">
        <h1 class="hero-title">
          找到你的
          <span class="gradient-text">理想居所</span>
        </h1>
        <p class="hero-subtitle">
          专为年轻人打造的高品质租房平台 · 真实房源 · 品质保障
        </p>

        <!-- 搜索框 -->
        <div class="hero-search">
          <el-input
            v-model="keyword"
            placeholder="搜索城市、区域、小区名称..."
            size="large"
            :prefix-icon="Search"
            clearable
            @keyup.enter="goSearch"
          />
          <el-button type="primary" size="large" @click="goSearch">
            <el-icon><Search /></el-icon>
            搜索房源
          </el-button>
        </div>

        <!-- 热门标签 -->
        <div class="hot-tags">
          <span>热门：</span>
          <a
            v-for="tag in hotTags"
            :key="tag"
            class="tag-item"
            @click="quickSearch(tag)"
          >
            {{ tag }}
          </a>
        </div>
      </div>

      <!-- 装饰元素 -->
      <div class="hero-decoration">
        <div class="deco-card deco-1">
          <span>🏢</span>
          <span>精装公寓</span>
        </div>
        <div class="deco-card deco-2">
          <span>🏠</span>
          <span>整租优选</span>
        </div>
        <div class="deco-card deco-3">
          <span>✨</span>
          <span>品质合租</span>
        </div>
      </div>
    </section>

    <!-- 特色服务 -->
    <section class="features-section page-container">
      <div class="features-grid">
        <div
          v-for="(feature, index) in features"
          :key="feature.title"
          class="feature-card"
          :style="{ animationDelay: `${index * 0.1}s` }"
        >
          <div class="feature-icon" :style="{ background: feature.color }">
            {{ feature.icon }}
          </div>
          <h3>{{ feature.title }}</h3>
          <p>{{ feature.desc }}</p>
        </div>
      </div>
    </section>

    <!-- 最新房源推荐 -->
    <section class="houses-section page-container">
      <div class="section-header">
        <h2>
          <el-icon><House /></el-icon>
          最新房源
        </h2>
        <router-link to="/house" class="view-all">
          查看全部 →
        </router-link>
      </div>

      <!-- 加载状态 -->
      <div v-if="loading" class="houses-loading">
        <el-skeleton :rows="3" animated />
      </div>

      <!-- 房源卡片网格 -->
      <div v-else class="house-grid">
        <div
          v-for="house in houseList"
          :key="house.houseId"
          class="house-card"
          @click="$router.push(`/house/${house.houseId}`)"
        >
          <!-- 图片区 -->
          <div class="card-image">
            <img :src="getCoverImage(house)" :alt="house.title" @error="handleImgError" :data-house-id="house.houseId" :data-title="house.title" />
            <div class="image-overlay">
              <span class="status-tag">{{ getStatusText(house.status) }}</span>
            </div>
          </div>

          <!-- 信息区 -->
          <div class="card-body">
            <h4 class="card-title">{{ house.title }}</h4>
            <p class="card-address">
              <el-icon><Location /></el-icon>
              {{ house.city }} · {{ house.district }}
            </p>
            <div class="card-info">
              <span>{{ house.area }}m²</span>
              <span>|</span>
              <span>{{ house.rooms }}室{{ house.halls }}厅</span>
              <span>|</span>
              <span>{{ house.orientation || '-' }}</span>
            </div>
            <div class="card-footer">
              <span class="rent-price">
                ¥{{ house.rentMonthly?.toLocaleString() || '-' }}
                <small>/月</small>
              </span>
            </div>
          </div>
        </div>
      </div>

      <!-- 空状态 -->
      <el-empty v-if="!loading && houseList.length === 0" description="暂无房源数据">
        <el-button type="primary" @click="fetchHouses">刷新试试</el-button>
      </el-empty>
    </section>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { Search, House, Location } from '@element-plus/icons-vue'
import { getHouseList } from '@/api/house'
import type { HouseInfo } from '@/api/house'

const router = useRouter()

// 搜索关键词
const keyword = ref('')

// 房源列表
const houseList = ref<HouseInfo[]>([])
const loading = ref(false)

// 热门搜索标签
const hotTags = ['天河', '海珠', '番禺', '白云', '精装修', '近地铁']

// 特色服务
const features = [
  {
    icon: '🔍',
    title: '真实房源',
    desc: '所有房源均经过人工审核，杜绝虚假信息',
    color: 'linear-gradient(135deg, #ff6b35, #f7931e)',
  },
  {
    icon: '💰',
    title: '透明价格',
    desc: '无中介费、无隐形消费，租金一目了然',
    color: 'linear-gradient(135deg, #4facfe, #00f2fe)',
  },
  {
    icon: '⚡',
    title: '极速响应',
    desc: '报修工单24小时内处理，管家式服务体验',
    color: 'linear-gradient(135deg, #2ecc71, #27ae60)',
  },
  {
    icon: '🛡️',
    title: '安全保障',
    desc: '电子合同+在线支付，全程资金安全保障',
    color: 'linear-gradient(135deg, #9b59b6, #8e44ad)',
  },
]

function goSearch() {
  if (keyword.value.trim()) {
    router.push({ path: '/house', query: { keyword: keyword.value.trim() } })
  }
}

function quickSearch(tag: string) {
  router.push({ path: '/house', query: { keyword: tag } })
}

async function fetchHouses() {
  loading.value = true
  try {
    const res = await getHouseList({ page: 1, size: 6 })
    // 后端返回格式可能是 PageResult，取 records 或 list 字段
    houseList.value = res.records || res.list || res || []
  } catch {
    houseList.value = []
  } finally {
    loading.value = false
  }
}

function getCoverImage(house: HouseInfo): string {
  // 空值、纯空白、"null" 字符串都视为无图
  if (house.coverImage && house.coverImage.trim().length > 0 && !house.coverImage.startsWith('null')) return house.coverImage
  const colors = ['#ff6b35', '#4facfe', '#2ecc71', '#9b59b6', '#f39c12', '#e74c3c']
  const idx = (house.houseId?.charCodeAt(0) || 0) % colors.length
  const color = colors[idx]
  const text = (house.title || '🏠').substring(0, 6)
  const info = `${house.rooms || '-'}室${house.halls || '-'}厅 · ${house.area || '?'}m²`
  return genPlaceholder(color, text, info)
}

/** 图片加载失败时自动替换为占位图 */
function handleImgError(e: Event) {
  const img = e.target as HTMLImageElement
  if (img.dataset.fallbacked) return
  const title = img.dataset.title || '🏠'
  const colors = ['#ff6b35', '#4facfe', '#2ecc71', '#9b59b6', '#f39c12']
  const color = colors[Math.floor(Math.random() * colors.length)]
  img.src = genPlaceholder(color, title.substring(0, 6), '')
  img.dataset.fallbacked = '1'
}

/** 生成 SVG 占位图 */
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

onMounted(() => {
  fetchHouses()
})
</script>

<style lang="scss" scoped>
.home-page {
  min-height: calc(100vh - 64px);
}

// ==================== Hero 区域 ====================
.hero-section {
  position: relative;
  padding: 80px 40px 60px;
  overflow: hidden;
  background: linear-gradient(135deg, #fff5eb 0%, #ffe8d6 50%, #e8f4fd 100%);

  .hero-bg-pattern {
    position: absolute;
    inset: 0;
    background-image:
      radial-gradient(circle at 20% 80%, rgba(255, 107, 53, 0.08) 0%, transparent 50%),
      radial-gradient(circle at 80% 20%, rgba(79, 172, 254, 0.08) 0%, transparent 50%);
  }

  .hero-content {
    position: relative;
    max-width: 680px;
    margin: 0 auto;
    text-align: center;

    .hero-title {
      font-size: 48px;
      font-weight: 800;
      line-height: 1.2;
      margin-bottom: 16px;
      letter-spacing: -1px;
    }

    .hero-subtitle {
      font-size: 17px;
      color: var(--color-text-secondary);
      margin-bottom: 36px;
    }
  }

  // 搜索框
  .hero-search {
    display: flex;
    gap: 12px;
    max-width: 560px;
    margin: 0 auto 20px;

    :deep(.el-input__wrapper) {
      border-radius: var(--radius-md) !important;
      box-shadow: var(--shadow-md) !important;
    }

    :deep(.el-input) {
      flex: 1;
    }

    .el-button {
      padding: 0 32px;
      border-radius: var(--radius-md) !important;
      font-weight: 600;
    }
  }

  // 热门标签
  .hot-tags {
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 10px;
    font-size: 14px;
    color: var(--color-text-secondary);

    .tag-item {
      padding: 4px 14px;
      background: white;
      border-radius: 9999px;
      cursor: pointer;
      transition: all var(--transition-fast);
      box-shadow: 0 1px 4px rgba(0, 0, 0, 0.06);

      &:hover {
        background: var(--color-primary);
        color: white;
        transform: translateY(-1px);
      }
    }
  }
}

// Hero 右侧装饰卡片
.hero-decoration {
  position: absolute;
  right: 60px;
  top: 50%;
  transform: translateY(-50%);
  display: none; // 大屏幕显示
  flex-direction: column;
  gap: 16px;

  @media (min-width: 1200px) {
    display: flex;
  }

  .deco-card {
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 16px 22px;
    background: white;
    border-radius: var(--radius-md);
    box-shadow: var(--shadow-md);
    animation: floatCard 4s ease-in-out infinite;

    span:first-child {
      font-size: 28px;
    }

    span:last-child {
      font-size: 15px;
      font-weight: 600;
    }

    &.deco-1 { animation-delay: 0s; }
    &.deco-2 { animation-delay: 1s; }
    &.deco-3 { animation-delay: 2s; }
  }
}

@keyframes floatCard {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-10px); }
}

// ==================== 特色服务 ====================
.features-section {
  margin-top: 48px;
}

.features-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 24px;
}

.feature-card {
  text-align: center;
  padding: 36px 24px;
  background: white;
  border-radius: var(--radius-lg);
  box-shadow: var(--shadow-sm);
  transition: all var(--transition-normal);

  &:hover {
    transform: translateY(-6px);
    box-shadow: var(--shadow-lg);
  }

  .feature-icon {
    width: 64px;
    height: 64px;
    border-radius: var(--radius-md);
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 30px;
    margin: 0 auto 18px;
  }

  h3 {
    font-size: 18px;
    font-weight: 700;
    margin-bottom: 8px;
  }

  p {
    font-size: 13px;
    color: var(--color-text-secondary);
    line-height: 1.7;
  }
}

// ==================== 房源列表 ====================
.houses-section {
  margin-top: 56px;
  padding-bottom: 60px;
}

.section-header {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  margin-bottom: 28px;

  h2 {
    font-size: 26px;
    font-weight: 800;
    display: flex;
    align-items: center;
    gap: 8px;

    .el-icon {
      color: var(--color-primary);
      font-size: 28px;
    }
  }

  .view-all {
    color: var(--color-primary);
    font-weight: 600;
    font-size: 15px;

    &:hover {
      text-decoration: underline;
    }
  }
}

.houses-loading {
  padding: 20px;
}

.house-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 24px;
}

// 房源卡片
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
      transform: scale(1.05);
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
      top: 12px;
      left: 12px;

      .status-tag {
        padding: 4px 12px;
        background: rgba(46, 204, 113, 0.9);
        color: white;
        border-radius: 9999px;
        font-size: 12px;
        font-weight: 500;
      }
    }
  }

  .card-body {
    padding: 18px;

    .card-title {
      font-size: 17px;
      font-weight: 700;
      margin-bottom: 8px;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }

    .card-address {
      font-size: 13px;
      color: var(--color-text-secondary);
      display: flex;
      align-items: center;
      gap: 4px;
      margin-bottom: 10px;
    }

    .card-info {
      display: flex;
      align-items: center;
      gap: 8px;
      font-size: 13px;
      color: var(--color-text-muted);
      margin-bottom: 14px;
    }

    .card-footer {
      display: flex;
      justify-content: space-between;
      align-items: center;
      border-top: 1px solid var(--color-border-light);
      padding-top: 14px;

      .rent-price {
        font-size: 22px;
        font-weight: 800;
        color: var(--color-primary);

        small {
          font-size: 13px;
          font-weight: 400;
          color: var(--color-text-muted);
        }
      }
    }
  }
}
</style>
