<template>
  <div class="publish-page" v-loading="pageLoading" element-loading-text="正在加载房源信息...">
    <div class="page-container">
      <!-- 页面标题 -->
      <div class="page-header">
        <h1>
          <el-icon><EditPen /></el-icon>
          {{ isEdit ? '编辑房源' : '发布新房源' }}
        </h1>
        <p class="header-desc">填写详细信息，让租客快速了解您的优质房源 ✨</p>
      </div>

      <el-form
        ref="formRef"
        :model="form"
        :rules="rules"
        label-width="100px"
        size="large"
        class="publish-form"
      >
        <!-- 基本信息 -->
        <el-card shadow="never" class="section-card">
          <template #header>
            <span class="card-title"><el-icon><HomeFilled /></el-icon> 基本信息</span>
          </template>

          <el-row :gutter="24">
            <el-col :span="16">
              <el-form-item label="房源标题" prop="title">
                <el-input
                  v-model="form.title"
                  placeholder="如：精装修一室一厅，地铁口步行5分钟"
                  maxlength="50"
                  show-word-limit
                />
              </el-form-item>
            </el-col>
            <el-col :span="8">
              <el-form-item label="房源类型" prop="houseType">
                <el-radio-group v-model="form.houseType" class="house-type-group">
                  <el-radio-button :value="1">整租</el-radio-button>
                  <el-radio-button :value="2">合租</el-radio-button>
                </el-radio-group>
              </el-form-item>
            </el-col>
          </el-row>

          <el-row :gutter="24">
            <el-col :span="24">
              <el-form-item label="所在地区" prop="region">
                <el-cascader
                  v-model="form.region"
                  :options="regionOptions"
                  placeholder="请选择省/市/区"
                  style="width:100%"
                  clearable
                />
              </el-form-item>
            </el-col>
          </el-row>
          <el-row :gutter="24">
            <el-col :span="24">
              <el-form-item label="详细地址" prop="address">
                <el-input v-model="form.address" placeholder="具体街道/小区/楼号" />
              </el-form-item>
            </el-col>
          </el-row>

          <el-form-item label="房屋描述" prop="description">
            <el-input
              v-model="form.description"
              type="textarea"
              :rows="4"
              placeholder="详细介绍房源特点、周边环境、交通情况等..."
              maxlength="500"
              show-word-limit
            />
          </el-form-item>
        </el-card>

        <!-- 户型与价格 -->
        <el-card shadow="never" class="section-card">
          <template #header>
            <span class="card-title"><el-icon><Coin /></el-icon> 户型与价格</span>
          </template>

          <el-row :gutter="20">
            <el-col :span="6">
              <el-form-item label="室" prop="rooms">
                <el-input-number v-model="form.rooms" :min="0" :max="10" controls-position="right" style="width:100%" />
              </el-form-item>
            </el-col>
            <el-col :span="6">
              <el-form-item label="厅" prop="halls">
                <el-input-number v-model="form.halls" :min="0" :max="5" controls-position="right" style="width:100%" />
              </el-form-item>
            </el-col>
            <el-col :span="6">
              <el-form-item label="卫" prop="bathrooms">
                <el-input-number v-model="form.bathrooms" :min="0" :max="5" controls-position="right" style="width:100%" />
              </el-form-item>
            </el-col>
            <el-col :span="6">
              <el-form-item label="面积(㎡)" prop="area">
                <el-input-number v-model="form.area" :min="5" :max="999" :precision="1" controls-position="right" style="width:100%" />
              </el-form-item>
            </el-col>
          </el-row>

          <el-row :gutter="20">
            <el-col :span="12">
              <el-form-item label="月租金(元)" prop="rentMonthly">
                <el-input-number v-model="form.rentMonthly" :min="0" :max="99999" :step="100" controls-position="right" style="width:100%">
                  <template #prefix>￥</template>
                </el-input-number>
              </el-form-item>
            </el-col>
            <el-col :span="12">
              <el-form-item label="押金(元)" prop="deposit">
                <el-input-number v-model="form.deposit" :min="0" :max="99999" :step="500" controls-position="right" style="width:100%">
                  <template #prefix>￥</template>
                </el-input-number>
              </el-form-item>
            </el-col>
          </el-row>

          <el-row :gutter="20">
            <el-col :span="12">
              <el-form-item label="楼层" prop="floor">
                <el-input v-model="form.floor" placeholder="如：6层/18层" />
              </el-form-item>
            </el-col>
            <el-col :span="6">
              <el-form-item label="朝向" prop="orientation">
                <el-select v-model="form.orientation" placeholder="选择朝向" style="width:100%">
                  <el-option value="东" label="东" />
                  <el-option value="南" label="南" />
                  <el-option value="西" label="西" />
                  <el-option value="北" label="北" />
                  <el-option value="南北" label="南北通透" />
                  <el-option value="东南" label="东南" />
                  <el-option value="西南" label="西南" />
                  <el-option value="其他" label="其他" />
                </el-select>
              </el-form-item>
            </el-col>
            <el-col :span="6">
              <el-form-item label="支付方式" prop="paymentType">
                <el-select v-model="form.paymentType" placeholder="选择支付方式" style="width:100%">
                  <el-option :value="0" label="月付" />
                  <el-option :value="1" label="季付" />
                  <el-option :value="2" label="半年付" />
                  <el-option :value="3" label="年付" />
                </el-select>
              </el-form-item>
            </el-col>
          </el-row>
        </el-card>

        <!-- 配套设施 -->
        <el-card shadow="never" class="section-card">
          <template #header>
            <span class="card-title"><el-icon><Box /></el-icon> 配套设施</span>
          </template>

          <el-form-item label="">
            <div class="facilities-grid">
              <div
                v-for="(item, index) in facilityOptions"
                :key="item.value"
                class="facility-tag"
                :class="{ active: form.facilities.includes(item.value) }"
                @click="toggleFacility(item.value)"
              >
                <span class="tag-icon">{{ item.icon }}</span>
                <span>{{ item.label }}</span>
              </div>
            </div>
          </el-form-item>
        </el-card>

        <!-- 封面图片 -->
        <el-card shadow="never" class="section-card">
          <template #header>
            <span class="card-title"><el-icon><PictureFilled /></el-icon> 房源图片</span>
          </template>

          <el-upload
            ref="uploadRef"
            :action="uploadAction"
            :headers="uploadHeaders"
            list-type="picture-card"
            :auto-upload="true"
            :limit="9"
            :on-success="handleUploadSuccess"
            :on-error="handleUploadError"
            :on-remove="handleImageRemove"
            :on-exceed="handleExceed"
            :before-upload="beforeUpload"
            accept="image/jpeg,image/png,image/webp"
          >
            <div class="upload-placeholder">
              <el-icon :size="28"><Plus /></el-icon>
              <p>上传图片</p>
            </div>
            <template #tip>
              <div class="upload-tip">
                最多上传 9 张图片，第一张自动设为封面。支持 JPG/PNG/WebP，单张不超过 10MB
              </div>
            </template>
          </el-upload>
        </el-card>

        <!-- 提交按钮 -->
        <div class="form-actions">
          <el-button size="large" @click="$router.back()">
            取消返回
          </el-button>
          <el-button type="primary" size="large" :loading="submitting" @click="handleSubmit">
            {{ isEdit ? '保存修改' : '立即发布' }}
            <el-icon v-if="!submitting"><Promotion /></el-icon>
          </el-button>
        </div>
      </el-form>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, computed, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import {
  EditPen,
  HomeFilled,
  Coin,
  Box,
  PictureFilled,
  Plus,
  Promotion,
} from '@element-plus/icons-vue'
import type { FormInstance, FormRules, UploadFile, UploadInstance } from 'element-plus'
import { ElMessage, genFileId } from 'element-plus'
import { pcaTextArr } from 'element-china-area-data'
import * as houseApi from '@/api/house'
import { getHouseImages } from '@/api/upload'
import request from '@/utils/request'

const regionOptions = pcaTextArr

const route = useRoute()
const router = useRouter()
const formRef = ref<FormInstance>()
const uploadRef = ref<UploadInstance>()
const submitting = ref(false)
const pageLoading = ref(false)

// 是否编辑模式
const isEdit = computed(() => !!route.query.id)

// 上传配置
const uploadAction = '/api/upload/image'
const uploadHeaders = {
  Authorization: `Bearer ${localStorage.getItem('token') || ''}`,
}

// 已上传的图片URL列表（按顺序，第一张为封面）
const uploadedUrls = ref<string[]>([])

// 表单数据
const form = reactive({
  title: '',
  region: [] as string[],
  address: '',
  houseType: 1,
  rooms: 1,
  halls: 0,
  bathrooms: 1,
  area: undefined as number | undefined,
  rentMonthly: undefined as number | undefined,
  deposit: undefined as number | undefined,
  floor: '',
  orientation: '' as string,
  paymentType: 0,
  description: '',
  facilities: [] as string[],
  coverImage: '' as string,
})

/** 原始数据快照（用于对比是否修改过） */
let originalSnapshot: Record<string, any> | null = null

// 配套设施选项
const facilityOptions = [
  { value: 'wifi', icon: '📶', label: 'WiFi' },
  { value: 'ac', icon: '❄️', label: '空调' },
  { value: 'washer', icon: '🧺', label: '洗衣机' },
  { value: 'fridge', icon: '🧊', label: '冰箱' },
  { value: 'heater', icon: '🚿', label: '热水器' },
  { value: 'tv', icon: '📺', label: '电视' },
  { value: 'bed', icon: '🛏️', label: '床' },
  { value: 'wardrobe', icon: '🗄️', label: '衣柜' },
  { value: 'elevator', icon: '🛗', label: '电梯' },
  { value: 'parking', icon: '🅿️', label: '停车位' },
  { value: 'balcony', icon: '🌿', label: '阳台' },
  { value: 'kitchen', icon: '🍳', label: '独立厨房' },
]

function toggleFacility(value: string) {
  const idx = form.facilities.indexOf(value)
  if (idx >= 0) {
    form.facilities.splice(idx, 1)
  } else {
    form.facilities.push(value)
  }
}

// 校验规则
const rules: FormRules = {
  title: [
    { required: true, message: '请输入房源标题', trigger: 'blur' },
    { min: 5, max: 50, message: '标题长度为5-50个字符', trigger: 'blur' },
  ],
  region: [{ required: true, message: '请选择所在地区', trigger: 'change' }],
  address: [{ required: true, message: '请输入详细地址', trigger: 'blur' }],
  area: [{ required: true, message: '请输入面积', trigger: 'blur' }],
  rentMonthly: [{ required: true, message: '请输入月租金', trigger: 'blur' }],
}

// ==================== 编辑模式：加载已有数据 ====================

/**
 * 加载房源详情回填表单（编辑模式）
 */
async function fetchExistingHouse() {
  if (!isEdit.value) return
  pageLoading.value = true
  try {
    const house = await houseApi.getMyHouseDetail(route.query.id as string)

    // 回填基础字段
    form.title = house.title || ''
    if (house.province || house.city || house.district) {
      form.region = [house.province || '', house.city || '', house.district || '']
    }
    form.address = house.address || ''
    form.houseType = house.houseType ?? 1
    form.rooms = house.rooms ?? 1
    form.halls = house.halls ?? 0
    form.bathrooms = house.bathrooms ?? 1
    form.area = house.area ? Number(house.area) : undefined
    form.rentMonthly = house.rentMonthly ? Number(house.rentMonthly) : undefined
    form.deposit = house.deposit ? Number(house.deposit) : undefined
    form.floor = house.floor || ''
    form.orientation = house.orientation || ''
    form.paymentType = house.paymentType ?? 0
    form.description = house.description || ''
    form.coverImage = house.coverImage || ''

    // 解析配套设施 JSON 字符串 → 数组
    try {
      const parsed = typeof house.facilities === 'string' && house.facilities
        ? JSON.parse(house.facilities)
        : []
      form.facilities = Array.isArray(parsed) ? parsed : []
    } catch {
      form.facilities = []
    }

    // 快照原始数据用于后续对比
    originalSnapshot = { ...form, facilities: [...form.facilities], coverImage: form.coverImage || '' }

    // 加载该房源已有的图片列表
    await loadExistingImages(route.query.id as string)
  } catch (err) {
    ElMessage.error('加载房源信息失败，请返回重试')
  } finally {
    pageLoading.value = false
  }
}

/** 加载已上传的图片并显示在 el-upload 中 */
async function loadExistingImages(houseId: string) {
  try {
    const res: any = await getHouseImages(houseId)
    const list: string[] = res?.list || []
    if (list.length > 0) {
      uploadedUrls.value = list
      // 如果没有 coverImage 但有图片列表，用第一张做封面
      if (!form.coverImage && list[0]) {
        form.coverImage = list[0]
        // 同步更新快照
        if (originalSnapshot) originalSnapshot.coverImage = list[0]
      }
    }
  } catch {
    console.warn('加载已有图片失败')
  }
}

onMounted(() => {
  fetchExistingHouse()
})

// ==================== 图片上传相关 ====================

/** 上传前校验：格式 + 大小 */
function beforeUpload(rawFile: UploadFile['raw']) {
  if (!rawFile) return false

  const allowedTypes = ['image/jpeg', 'image/png', 'image/webp']
  if (!allowedTypes.includes(rawFile.type)) {
    ElMessage.error('仅支持 JPG/PNG/WebP 格式')
    return false
  }

  const isLt10M = rawFile.size / 1024 / 1024 < 10
  if (!isLt10M) {
    ElMessage.error('图片大小不能超过 10MB')
    return false
  }

  return true
}

/** 上传成功回调 */
function handleUploadSuccess(res: any, file: UploadFile) {
  if (res.code === 200 && res.data?.url) {
    // 第一张自动设为封面
    if (uploadedUrls.value.length === 0) {
      form.coverImage = res.data.url
    }
    uploadedUrls.value.push(res.data.url)
    ElMessage.success(`图片 "${file.name}" 上传成功 ✅`)
  } else {
    ElMessage.error((res?.message || '上传失败'))
    // 从列表中移除失败的项
    if (uploadRef.value && file.uid) {
      uploadRef.value.handleRemove(file as any)
    }
  }
}

/** 上传失败回调 */
function handleUploadError() {
  ElMessage.error('图片上传失败，请检查网络后重试')
}

/** 删除已上传的图片 */
function handleImageRemove(file: UploadFile) {
  // 从 URL 列表中移除对应图片
  const urlToFind = (file.response?.data?.url || file.url)
  if (urlToFind) {
    const idx = uploadedUrls.value.indexOf(urlToFind)
    if (idx >= 0) {
      uploadedUrls.value.splice(idx, 1)
    }
  }

  // 如果删除的是封面，重新设置封面
  if (form.coverImage === urlToFind) {
    form.coverImage = uploadedUrls.value[0] || ''
  }
}

/** 超出数量限制 */
function handleExceed() {
  ElMessage.warning('最多只能上传9张图片')
}

// 提交表单
async function handleSubmit() {
  if (!formRef.value) return

  await formRef.value.validate(async (valid) => {
    if (!valid) return

    submitting.value = true
    try {
      const { region, ...rest } = form
      const payload: Record<string, any> = {
        ...rest,
        province: region[0] || '',
        city: region[1] || '',
        district: region[2] || '',
        facilities: JSON.stringify(form.facilities),
        coverImage: form.coverImage || '',
      }

      // 编辑模式：检测是否有内容变更
      let hasChanged = false
      if (isEdit.value && originalSnapshot) {
        hasChanged = detectChanges(originalSnapshot, payload)
        if (hasChanged) {
          payload.needReaudit = true  // 告知后端重置为待审核
        }
      }

      let result: any
      if (isEdit.value) {
        result = await houseApi.updateHouse(route.query.id as string, payload)
        ElMessage.success(hasChanged ? '已保存，需重新审核通过后生效 ⏳' : '房源信息未发生变更 🎉')

        // 如果有新上传的图片，批量关联到该房源
        if (uploadedUrls.value.length > 0) {
          await batchLinkImages(route.query.id as string)
          // 新图片也视为变更，如果之前没标记的话
          if (!hasChanged) {
            // 有新图片上传但表单没变，也算修改
            ElMessage.closeAll()
            ElMessage.success('检测到新图片，将重新审核 ⏳')
          }
        } else if (!hasChanged) {
          // 真正什么都没变
        }
      } else {
        result = await houseApi.publishHouse(payload)
        const newHouseId = result?.houseId || result?.data?.houseId

        // 发布成功后将已上传图片关联到新房源
        if (newHouseId && uploadedUrls.value.length > 0) {
          await batchLinkImages(newHouseId)
        }

        ElMessage.success('发布成功！等待审核中 ⏳')
      }
      router.push('/landlord/houses')
    } catch (err) {
      // 错误已在拦截器中处理
    } finally {
      submitting.value = false
    }
  })
}

/**
 * 深度对比原始数据与当前提交数据，判断是否发生了实质性变更
 * @returns true 表示有变化需要重新审核
 */
function detectChanges(original: Record<string, any>, current: Record<string, any>): boolean {
  // 需要对比的关键字段（排除 houseId、createTime 等不可编辑字段）
  const compareKeys = [
    'title', 'region', 'address',
    'houseType', 'rooms', 'halls', 'bathrooms',
    'area', 'rentMonthly', 'deposit', 'floor', 'orientation', 'paymentType',
    'description', 'facilities', 'coverImage'
  ]

  for (const key of compareKeys) {
    const origVal = String(original[key] ?? '')
    const currVal = String(current[key] ?? '')
    if (origVal !== currVal) return true
  }
  return false
}

/**
 * 批量将临时上传的图片关联到指定房源
 * （因为发布时还没有 houseId，所以先临时上传，发布后再关联）
 */
async function batchLinkImages(houseId: string) {
  for (let i = 0; i < uploadedUrls.value.length; i++) {
    try {
      await request.post('/upload/link-image', null, {
        params: {
          imageUrl: uploadedUrls.value[i],
          houseId: houseId,
          isCover: i === 0 ? 'true' : 'false',
        },
      })
    } catch {
      console.warn(`关联图片失败: ${uploadedUrls.value[i]}`)
    }
  }
}
</script>

<style lang="scss" scoped>
.publish-page {
  background: var(--color-bg);
  min-height: calc(100vh - 64px - 60px);
  padding-bottom: 60px;
}

.page-header {
  padding: 32px 0 16px;
  
  h1 {
    font-size: 26px;
    font-weight: 700;
    color: var(--color-text);
    display: flex;
    align-items: center;
    gap: 10px;
    
    .el-icon {
      color: var(--color-primary);
    }
  }

  .header-desc {
    margin-top: 8px;
    color: var(--color-text-secondary);
    font-size: 15px;
  }
}

.publish-form {
  max-width: 900px;
}

.section-card {
  margin-bottom: 20px;
  border-radius: var(--radius-md);

  :deep(.el-card__header) {
    padding: 16px 24px;
    border-bottom: 1px solid var(--color-border-light);
  }

  .card-title {
    font-size: 17px;
    font-weight: 600;
    color: var(--color-text);
    display: flex;
    align-items: center;
    gap: 8px;

    .el-icon {
      color: var(--color-primary);
    }
  }
}

.house-type-group {
  width: 100%;
}

.facilities-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
}

.facility-tag {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 10px 18px;
  border: 2px solid var(--color-border);
  border-radius: var(--radius-sm);
  cursor: pointer;
  transition: all var(--transition-fast);
  font-size: 14px;
  user-select: none;

  &:hover {
    border-color: var(--color-primary-light);
    background: rgba(255, 107, 53, 0.04);
  }

  &.active {
    border-color: var(--color-primary);
    background: rgba(255, 107, 53, 0.08);
    color: var(--color-primary);
    font-weight: 600;
    
    .tag-icon {
      transform: scale(1.15);
    }
  }

  .tag-icon {
    font-size: 18px;
    transition: transform 0.2s ease;
  }
}

.upload-placeholder {
  text-align: center;
  color: var(--color-text-muted);

  p {
    margin-top: 4px;
    font-size: 13px;
  }
}

.upload-tip {
  margin-top: 8px;
  color: var(--color-text-muted);
  font-size: 13px;
}

.form-actions {
  display: flex;
  justify-content: flex-end;
  gap: 16px;
  padding: 24px 0;

  .el-button {
    min-width: 120px;
    height: 46px !important;
    font-size: 15px !important;
    border-radius: var(--radius-sm) !important;
  }

  .el-button--primary {
    padding: 0 36px !important;
    background: linear-gradient(135deg, var(--color-primary), #ff9a56) !important;
    border: none !important;
    
    &:hover:not(:disabled) {
      transform: translateY(-2px);
      box-shadow: var(--shadow-primary);
    }
  }
}
</style>
