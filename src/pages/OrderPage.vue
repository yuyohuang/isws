<template>
  <q-page class="page-bg flex flex-center q-pa-md">
    <q-card class="order-card shadow-3">
      <!-- Header -->
      <q-card-section class="row items-start justify-between q-pb-none">
        <div>
          <div class="row items-center">
            <q-icon name="add_shopping_cart" size="22px" class="q-mr-sm text-primary" />
            <div class="text-h6 text-primary text-weight-bold">建立新訂單</div>
          </div>
          <div class="text-caption text-grey-7 q-mt-xs">
            系統會自動檢查庫存、分配貨箱，或排入等待區
          </div>
        </div>
      </q-card-section>

      <!-- 系統訊息 -->
      <div class="q-px-md q-pt-md q-gutter-sm full-width">
        <!-- 錯誤訊息（例如庫存不足） -->
        <q-banner
          v-if="errorMessage"
          class="banner-error"
          rounded
          dense
        >
          <div class="text-body2 text-weight-medium">
            {{ errorMessage }}
          </div>

          <div
            v-for="(item, idx) in insufficientDetails"
            :key="idx"
            class="text-caption q-mt-xs"
          >
            - {{ item.product_name || ('商品ID ' + item.product_id) }}：需求 {{ item.need }}，可用 {{ item.available }}
          </div>
        </q-banner>

        <!-- 成功訊息 -->
        <q-banner
          v-if="successMessage"
          class="banner-success"
          rounded
          dense
        >
          <div class="text-body2 text-weight-medium">
            {{ successMessage }}
          </div>

          <div v-if="assignedContainer !== null" class="text-caption q-mt-xs">
            已分配貨箱： #{{ assignedContainer }}
          </div>
          <div v-else class="text-caption q-mt-xs">
            目前無空貨箱，訂單已排入等待區
          </div>
        </q-banner>
      </div>

      <!-- 區塊 1: 客戶資訊 -->
      <q-card-section class="section-block">
        <div class="section-header row items-center q-mb-md">
          <q-icon name="person" size="20px" class="text-grey-8 q-mr-sm" />
          <div class="text-subtitle1 text-weight-medium text-grey-8">1. 客戶資訊</div>
          <div class="header-line q-ml-sm" />
        </div>

        <div class="row q-col-gutter-md">
          <div class="col-12 col-md-6">
            <q-input
              v-model="customerName"
              label="姓名 *"
              outlined
              dense
              :disable="loading"
              :rules="[val => !!val || '必填']"
            />
          </div>
          <div class="col-12 col-md-6">
            <q-input
              v-model="customerEmail"
              label="Email"
              outlined
              dense
              :disable="loading"
              type="email"
            />
          </div>
        </div>
      </q-card-section>

      <!-- 區塊 2: 訂單品項 -->
      <q-card-section class="section-block">
        <div class="section-header row items-center q-mb-md">
          <q-icon name="inventory_2" size="20px" class="text-grey-8 q-mr-sm" />
          <div class="text-subtitle1 text-weight-medium text-grey-8">2. 訂單品項</div>
          <div class="header-line q-ml-sm" />
          <q-btn
            dense
            flat
            color="primary"
            icon="add"
            label="新增品項"
            class="q-ml-auto"
            @click="addItem"
            :disable="loading"
          />
        </div>

        <div
          v-for="(row, idx) in items"
          :key="idx"
          class="item-row q-pa-md q-mb-sm"
        >
          <div class="row q-col-gutter-md items-end">
            <!-- 商品 -->
            <div class="col-12 col-md-6">
              <q-select
                v-model="row.product_id"
                :options="productOptions"
                option-value="value"
                option-label="label"
                emit-value
                map-options
                outlined
                dense
                :disable="loading"
                label="商品 *"
                :rules="[val => !!val || '必選']"
              />
            </div>

            <!-- 數量 -->
            <div class="col-8 col-md-4">
              <q-input
                v-model.number="row.quantity"
                type="number"
                outlined
                dense
                :disable="loading"
                label="數量 *"
                :rules="[
                  val => val !== null && val !== '' || '必填',
                  val => val > 0 || '必須 > 0'
                ]"
                min="1"
              >
                <template #prepend>
                  <q-btn
                    round
                    dense
                    flat
                    icon="remove"
                    @click="row.quantity = Math.max(1, (row.quantity || 1) - 1)"
                    :disable="loading"
                  />
                </template>
                <template #append>
                  <q-btn
                    round
                    dense
                    flat
                    icon="add"
                    @click="row.quantity = (row.quantity || 1) + 1"
                    :disable="loading"
                  />
                </template>
              </q-input>
            </div>

            <!-- 刪除 -->
            <div class="col-4 col-md-2 flex flex-center">
              <q-btn
                flat
                round
                color="negative"
                icon="delete"
                @click="removeItem(idx)"
                :disable="loading || items.length === 1"
              />
            </div>
          </div>
        </div>
      </q-card-section>

      <!-- 區塊 3: 送出 -->
      <q-card-actions align="right" class="q-pa-md">
        <q-btn
          color="primary"
          label="送出訂單"
          :loading="loading"
          class="q-px-md"
          @click="submitOrder"
        />
      </q-card-actions>
    </q-card>
  </q-page>
</template>

<script setup>
import { ref } from 'vue'
import { useQuasar } from 'quasar'
import axios from 'axios'

// 後端 API base URL
const API_BASE = 'http://localhost:8001/api'

// Quasar notify
const $q = useQuasar()

// --- 表單欄位狀態 ---
const customerName = ref('')
const customerEmail = ref('')

// 訂單品項陣列（每筆包含 product_id, quantity）
const items = ref([
  { product_id: null, quantity: 1 }
])

// 商品下拉選單內容
// 注意：value 必須對應資料庫 products.id
const productOptions = [
  { label: '黃色積木', value: 1 },
  { label: '黑色積木', value: 2 },
  { label: '輪胎',     value: 3 }
]

// --- UI 狀態 ---
const loading = ref(false)

// 回傳訊息用
const errorMessage = ref('')
const insufficientDetails = ref([])

const successMessage = ref('')
const assignedContainer = ref(null) // null 代表沒有貨箱（等待中）

// --- 方法：新增一列品項 ---
function addItem () {
  items.value.push({
    product_id: null,
    quantity: 1
  })
}

// --- 方法：刪除一列品項 ---
function removeItem (idx) {
  if (items.value.length === 1) {
    return // 保底至少一列，避免刪到空
  }
  items.value.splice(idx, 1)
}

// --- 方法：送出訂單 ---
async function submitOrder () {

  // 簡單前端驗證
  if (!customerName.value) {
    $q.notify({ type: 'negative', message: '請填寫客戶姓名' })
    return
  }

  if (
    items.value.length === 0 ||
    items.value.some(row => !row.product_id || !row.quantity || row.quantity <= 0)
  ) {
    $q.notify({ type: 'negative', message: '請確認商品與數量' })
    return
  }
  // 清空前一次的訊息
  errorMessage.value = ''
  insufficientDetails.value = []
  successMessage.value = ''
  assignedContainer.value = null

  // 要送去後端的 payload
  const payload = {
    customer_name: customerName.value,
    customer_email: customerEmail.value || null,
    items: items.value.map(row => ({
      product_id: row.product_id,
      quantity: row.quantity
    }))
  }

  loading.value = true
  let orderOk = false

  try {
    const resp = await axios.post(`${API_BASE}/orders`, payload)

    // 防禦：後端沒回東西的話，當作錯誤處理
    if (!resp || !resp.data) {
      throw new Error('後端沒有回傳資料')
    }

    // 走到這裡表示確實有成功
    orderOk = true

    // 成功訊息
    successMessage.value =
      `訂單建立成功 (單號 ${resp.data.order_number})，狀態：${resp.data.status}`

    assignedContainer.value =
      (resp.data.container && resp.data.container.id)
        ? resp.data.container.id
        : null

    // 清空表單
    customerName.value = ''
    customerEmail.value = ''
    items.value = [{ product_id: null, quantity: 1 }]

    // 正向通知
    $q.notify({
      type: 'positive',
      message: '訂單已送出',
      timeout: 1500
    })
  } catch (err) {
    console.error('提交訂單時發生例外:', err)

    // 如果後端主動回 400，代表庫存不足
    if (err.response && err.response.status === 400) {
      const detail = err.response.data.detail
      errorMessage.value = detail?.msg || '庫存不足，訂單未建立'
      insufficientDetails.value = detail?.insufficient || []

      $q.notify({
        type: 'negative',
        message: errorMessage.value,
        timeout: 2000
      })
    } else {
      // 其他錯誤（網路斷線 / 格式不符 / try 內拋錯）
      // 如果訂單其實已經成功 (orderOk=true)，就不要再嚇使用者
      if (!orderOk) {
        errorMessage.value = '系統錯誤，請稍後再試'

        $q.notify({
          type: 'negative',
          message: errorMessage.value,
          timeout: 2000
        })
      }
    }
  } finally {
    loading.value = false
  }
}
</script>

<style scoped>
.page-bg {
  background-color: #f5f6fa;
  min-height: 100vh;
}

.order-card {
  width: 100%;
  max-width: 700px;
  background: #fff;
  border-radius: 12px;
}

.banner-error {
  background: #feecec;
  border: 1px solid #f7b5b5;
  color: #b00020;
  border-radius: 8px;
  padding: 12px 16px;
  line-height: 1.4;
}

.banner-success {
  background: #e9f9ef;
  border: 1px solid #8cd9a8;
  color: #1a7f3c;
  border-radius: 8px;
  padding: 12px 16px;
  line-height: 1.4;
}

.section-block {
  padding-top: 16px;
  padding-bottom: 4px;
}

.section-header {
  line-height: 1.2;
  width: 100%;
}

.header-line {
  flex-grow: 1;
  height: 1px;
  background: #e0e0e0;
}

.item-row {
  background: #fafafa;
  border: 1px solid #e2e2e2;
  border-radius: 8px;
}
</style>
