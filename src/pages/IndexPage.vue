<template>
  <q-page class="dashboard-bg q-pa-md">
    <div class="row q-col-gutter-md q-mb-md">
      <div class="col-12" v-if="lowStockWarnings.length">
        <q-banner class="sys-banner-warn" rounded dense>
          <q-icon name="warning" class="q-mr-sm" />
          <div class="text-body2">以下品項庫存偏低：</div>
          <div
            v-for="item in lowStockWarnings"
            :key="item.id"
            class="text-caption q-ml-md q-mt-xs"
          >
            - {{ item.name }} 剩餘 {{ item.available }}
          </div>
        </q-banner>
      </div>
    </div>

    <div class="row q-col-gutter-lg">
      <div class="col-12 col-lg-8 column q-gutter-lg">

        <q-card class="panel-card orders-card shadow-2">
          <div class="panel-header row items-start justify-between">
            <div class="panel-title row items-center">
              <q-icon name="assignment" size="22px" class="q-mr-sm text-primary" />
              <span>訂單</span>
            </div>
            <q-btn
              dense
              flat
              color="primary"
              icon="download"
              label="匯出今日 CSV"
              :loading="downloadingReport"
              @click="downloadTodayReport"
            />
          </div>

          <q-separator />

          <div class="panel-body">

            <div class="text-body2 text-weight-bold row items-center q-mb-xs">
              <q-icon name="inventory_2" size="18px" class="q-mr-sm text-green" />
              <span>正在貨箱中的訂單</span>
              <q-badge color="green" class="text-caption q-ml-sm">
                {{ activeOrders.length }} 筆
              </q-badge>
            </div>

            <div class="orders-list-active q-mb-md">
              <template v-if="activeOrders.length">
                <q-list separator>
                  <q-item
                    v-for="order in activeOrders"
                    :key="order.id"
                  >
                    <q-item-section>
                    <div class="text-body1 text-weight-medium">
                        {{ order.order_number }} - {{ order.customer_name }}
                      </div>
                      <div class="text-caption text-grey">
                        {{ order.customer_email || '（無 Email）' }}
                      </div>
                      <div
                        v-if="order.items && order.items.length"
                        class="order-items text-caption text-grey-8 q-mt-xs"
                      >
                        <div
                          v-for="(item, idx) in order.items"
                          :key="`active-${order.id}-${idx}`"
                        >
                          - {{ item.product_name || (`商品#${item.product_id}`) }} × {{ item.quantity }}
                        </div>
                      </div>
                    </q-item-section>

                    <q-item-section side top class="text-right">
                      <q-badge
                        :color="statusColor(order.status)"
                        class="text-caption"
                      >
                        {{ order.status }}
                      </q-badge>
                    </q-item-section>
                  </q-item>
                </q-list>
              </template>

              <template v-else>
                <div class="empty-hint q-mb-md">
                  目前沒有在貨箱中的訂單
                </div>
              </template>
            </div>

            <q-separator spaced />

            <div class="text-body2 text-weight-bold row items-center q-mb-xs q-mt-sm">
              <q-icon name="pending_actions" size="18px" class="q-mr-sm text-orange" />
              <span>等待分配的訂單</span>
              <q-badge color="orange" class="text-caption q-ml-sm">
                {{ waitingOrders.length }} 筆
              </q-badge>
            </div>

            <div class="orders-list-waiting">
              <template v-if="waitingOrders.length">
                <q-list separator>
                  <q-item
                    v-for="order in waitingOrders"
                    :key="order.id"
                  >
                    <q-item-section>
                    <div class="text-body1 text-weight-medium">
                        {{ order.order_number }} - {{ order.customer_name }}
                      </div>
                      <div class="text-caption text-grey">
                        {{ order.customer_email || '（無 Email）' }}
                      </div>
                      <div
                        v-if="order.items && order.items.length"
                        class="order-items text-caption text-grey-8 q-mt-xs"
                      >
                        <div
                          v-for="(item, idx) in order.items"
                          :key="`waiting-${order.id}-${idx}`"
                        >
                          - {{ item.product_name || (`商品#${item.product_id}`) }} × {{ item.quantity }}
                        </div>
                      </div>
                    </q-item-section>

                    <q-item-section side top class="text-right">
                      <q-badge
                        :color="statusColor(order.status)"
                        class="text-caption"
                      >
                        {{ order.status }}
                      </q-badge>
                    </q-item-section>
                  </q-item>
                </q-list>
              </template>

              <template v-else>
                <div class="empty-hint">
                  目前沒有等待中的訂單
                </div>
              </template>
            </div>
          </div>
        </q-card>

        <q-card class="panel-card shadow-2">
          <div class="panel-header">
            <div class="panel-title row items-center">
              <q-icon name="smart_toy" size="22px" class="q-mr-sm text-primary" />
              <span>最新影像偵測</span>
            </div>
            <div class="panel-subtitle">
              來自即時攝影機 / YOLO 模組
            </div>
          </div>

          <q-separator />

          <div class="panel-body scroll-area">
            <template v-if="detections.length">
              <div
                v-for="det in detections.slice(0, 1)"
                :key="det.id"
                class="detect-row"
              >
                <div class="text-caption text-grey-7">
                  <b>#{{ det.id }}</b>?{{ det.source_id }}?{{ det.timestamp }}
                </div>
                <div class="row q-col-gutter-sm text-caption q-mt-xs">
                  <div class="col-6">
                    frame: <span class="text-weight-medium">{{ det.frame_id }}</span>
                  </div>
                  <div class="col-6">
                    ???? <span class="text-weight-medium">{{ det.image_width }}?{{ det.image_height }}</span>
                  </div>
                </div>
                <div
                  v-if="det.classes && det.classes.length"
                  class="q-mt-xs row items-center q-col-gutter-xs"
                >
                  <q-chip
                    v-for="(c, idx) in det.classes"
                    :key="`cls-${det.id}-${idx}`"
                    size="sm"
                    color="primary"
                    text-color="white"
                    class="text-caption"
                  >
                    {{ c.cls_name || '??' }}<span v-if="c.conf != null"> ({{ (c.conf).toFixed(2) }})</span>
                  </q-chip>
                </div>
                <div v-else class="text-caption text-grey-6 q-mt-xs">
                  ???
                </div>
                <div v-if="det.frame_base64" class="q-mt-sm">
                  <img
                    :src="`data:image/jpeg;base64,${det.frame_base64}`"
                    alt="detection"
                    style="max-width: 100%; border-radius: 6px; border: 1px solid #e0e0e0;"
                  />
                </div>
                <div class="text-caption text-grey-6 q-mt-xs">
                  {{ det.created_at }}
                </div>
              </div>
            </template>

            <template v-else>
              <div class="empty-hint">
                暫無偵測資料
              </div>
            </template>
          </div>
        </q-card>
      </div>

      <div class="col-12 col-lg-4 column q-gutter-lg">

        <q-card class="panel-card shadow-2 right-panel-card">
          <div class="panel-header">
            <div class="panel-title row items-center">
              <q-icon name="inventory" size="22px" class="q-mr-sm text-primary" />
              <span>庫存狀態</span>
            </div>
            <div class="panel-subtitle">
              可用數量低於安全值會標橘
            </div>
          </div>

          <q-separator />

          <div class="panel-body scroll-area">
            <template v-if="inventory.length">
              <q-list separator>
                <q-item
                  v-for="inv in inventory"
                  :key="inv.id"
                >
                  <q-item-section>
                    <div class="text-body2 text-weight-medium">
                      {{ inv.name }}
                    </div>
                    <div class="text-caption text-grey">
                      總數：{{ inv.quantity }}
                    </div>
                  </q-item-section>

                  <q-item-section side class="text-right">
                    <q-badge
                      :color="inv.available <= LOW_STOCK_THRESHOLD ? 'orange' : 'green'"
                      class="text-caption"
                    >
                      可用：{{ inv.available }}
                    </q-badge>
                  </q-item-section>
                </q-item>
              </q-list>
            </template>

            <template v-else>
              <div class="empty-hint">
                無庫存資料
              </div>
            </template>
          </div>
        </q-card>

        <q-card class="panel-card shadow-2 right-panel-card">
          <div class="panel-header">
            <div class="panel-title row items-center">
              <q-icon name="inventory_2" size="22px" class="q-mr-sm text-primary" />
              <span>貨箱狀態</span>
            </div>
            <div class="panel-subtitle">
              1 ~ 9 號貨箱即時狀態
            </div>
          </div>

          <q-separator />

          <div class="panel-body">
            <div class="row q-col-gutter-sm">
              <div
                v-for="box in containers"
                :key="box.id"
                class="col-4"
              >
                <div
                  class="bin-tile text-center"
                  :class="{
                    'bin-free': box.status === '空閒' || box.status === 'idle',
                    'bin-busy': box.status === '使用中' || box.status === 'busy'
                  }"
                >
                  <div class="text-subtitle2 text-weight-bold">
                    {{ formatContainerLabel(box.container_number) }}
                  </div>
                  <div class="text-caption">
                    {{ box.status }}
                  </div>
                  <div
                    v-if="ordersByContainer[box.id]"
                    class="bin-order-info"
                  >
                    訂單：{{ ordersByContainer[box.id].order_number || `#${ordersByContainer[box.id].id}` }}
                    <div class="order-customer">
                      {{ ordersByContainer[box.id].customer_name || '未指定客戶' }}
                    </div>
                  </div>
                </div>
              </div>
            </div>

            <div v-if="!containers.length" class="empty-hint q-mt-md">
              <q-icon name="link_off" size="24px" class="q-mb-xs" /><br>
              硬體未連線 / 無數據
            </div>
          </div>
        </q-card>
      </div>
    </div>
  </q-page>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, computed } from 'vue'
import axios from 'axios'

// 後端 API base URL
const API_BASE = 'http://localhost:8001/api'

// ---- 狀態 ----
const ordersToday = ref([])
const inventory = ref([])
const containers = ref([])
const detections = ref([])
const downloadingReport = ref(false)

const LOW_STOCK_THRESHOLD = 5
const lowStockWarnings = ref([])

// 狀態顏色 Helper
function statusColor (status) {
  if (status === '等待中' || status === 'queued') return 'orange'
  if (status === '正在執行中' || status === 'running') return 'green'
  if (status === '新建' || status === 'new') return 'blue'
  return 'grey'
}

function formatContainerLabel (containerNumber) {
  if (containerNumber == null || containerNumber === '') return '(?)'
  const normalized = String(containerNumber).trim()
  const tupleMatch = normalized.match(/^\(?\s*(\d+)\s*,\s*(\d+)\s*\)?$/)
  if (tupleMatch) return `(${tupleMatch[1]},${tupleMatch[2]})`

  const num = Number(normalized)
  if (Number.isFinite(num) && num > 0) {
    const row = Math.floor((num - 1) / 3) + 1
    const col = ((num - 1) % 3) + 1
    return `(${row},${col})`
  }

  return `(${normalized})`
}

// 依照 status 分組 (Computed)
const activeOrders = computed(() =>
  ordersToday.value.filter(o => o.status === '正在執行中' || o.status === 'running' || o.container_id != null)
)
const waitingOrders = computed(() =>
  ordersToday.value.filter(o => o.status === '等待中' || o.status === 'queued')
)
const ordersByContainer = computed(() => {
  const map = {}
  for (const order of ordersToday.value) {
    if (order && order.container_id != null) {
      map[order.container_id] = order
    }
  }
  return map
})

/**
 * 從後端抓資料 - 修正版
 * 使用 Promise.allSettled 平行處理，避免單一硬體 API 失敗導致全部卡死
 */
async function fetchData () {
  // 設定 Timeout，避免硬體斷線時請求卡住太久 (例如 2秒)
  const config = { timeout: 2000 }

  // 定義所有請求，但不立即 await
  const pOrders = axios.get(`${API_BASE}/orders/unfinished`, config)
  const pInv = axios.get(`${API_BASE}/inventory`, config)
  const pBox = axios.get(`${API_BASE}/containers`, config)
  const pDet = axios.get(`${API_BASE}/detections/latest`, { params: { limit: 10 }, ...config })

  // 同時發送並等待全部結果 (不管是成功還是失敗)
  const results = await Promise.allSettled([pOrders, pInv, pBox, pDet])
  const [resOrders, resInv, resBox, resDet] = results

  // 1. 處理訂單
  if (resOrders.status === 'fulfilled') {
    ordersToday.value = resOrders.value.data || []
  } else {
    console.warn('訂單資料載入失敗:', resOrders.reason)
  }

  // 2. 處理庫存
  if (resInv.status === 'fulfilled') {
    inventory.value = resInv.value.data || []
    
    // 更新低庫存警告
    lowStockWarnings.value = inventory.value
      .filter(item => item.available <= LOW_STOCK_THRESHOLD)
      .map(item => ({
        id: item.id,
        name: item.name,
        available: item.available
      }))
  } else {
    console.warn('庫存資料載入失敗:', resInv.reason)
  }

  // 3. 處理貨箱 (硬體)
  if (resBox.status === 'fulfilled') {
    containers.value = resBox.value.data || []
  } else {
    // 這裡通常是硬體斷線，我們可以只記錄 Log，不清除畫面也不報錯
    console.warn('貨箱 API 連線失敗 (硬體可能離線):', resBox.reason)
    // 選擇性：如果要明確清空硬體狀態，可以取消下面註解
    // containers.value = []
  }

  // 4. 處理偵測
  if (resDet.status === 'fulfilled') {
    detections.value = resDet.value.data || []
  } else {
    console.warn('偵測資料載入失敗:', resDet.reason)
  }
}

/* 輪詢機制 */
let poll = null
onMounted(() => {
  fetchData() // 第一次立即執行
  poll = setInterval(() => {
    fetchData()
  }, 1500) // 之後每 1.5 秒更新一次
})

onBeforeUnmount(() => {
  if (poll) clearInterval(poll)
})

async function downloadTodayReport () {
  const today = new Date()
  const dateStr = today.toISOString().slice(0, 10)
  downloadingReport.value = true
  try {
    const resp = await axios.get(`${API_BASE}/reports/daily/download`, {
      params: { date: dateStr },
      responseType: 'blob'
    })
    const blob = new Blob([resp.data], { type: 'text/csv' })
    const url = window.URL.createObjectURL(blob)
    const link = document.createElement('a')
    link.href = url
    link.download = `inout_${dateStr.replace(/-/g, '')}.csv`
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)
    window.URL.revokeObjectURL(url)
  } catch (err) {
    console.error('下載報表失敗', err)
  } finally {
    downloadingReport.value = false
  }
}
</script>

<style scoped>
.dashboard-bg {
  background-color: #f5f6fa;
  min-height: 100vh;
}

.sys-banner-warn {
  background: #fff7e6;
  border: 1px solid #ffd48a;
  color: #704c00;
  padding: 8px 12px;
  border-radius: 8px;
  line-height: 1.4;
}

/* 通用卡片框架 */
.panel-card {
  background: #fff;
  border-radius: 12px;
  min-height: 220px;
  display: flex;
  flex-direction: column;
}
.right-panel-card {
  min-height: 260px;
}

.orders-card {
  min-height: 460px;
  display: flex;
  flex-direction: column;
}

.panel-header {
  padding: 16px 16px 8px 16px;
}

.panel-title {
  font-size: 16px;
  font-weight: 600;
  line-height: 1.2;
  color: #1a1a1a;
}

.panel-subtitle {
  font-size: 12px;
  color: #6b6b6b;
  line-height: 1.4;
  margin-top: 4px;
}

.panel-body {
  padding: 12px 16px 16px 16px;
  flex-grow: 1;
}

.scroll-area {
  max-height: 240px;
  overflow-y: auto;
  scrollbar-width: thin;
}

.orders-list-active,
.orders-list-waiting {
  max-height: 180px;
  overflow-y: auto;
  scrollbar-width: thin;
  border: 1px solid #eee;
  border-radius: 8px;
  padding: 8px 12px;
  background: #fafafa;
}

.order-items {
  line-height: 1.3;
}

.detect-row {
  border-bottom: 1px solid #ececec;
  padding: 8px 0;
  font-size: 12px;
  line-height: 1.4;
}

.empty-hint {
  padding: 24px 12px;
  text-align: center;
  font-size: 13px;
  color: #999;
}

/* 貨箱格子 */
.bin-tile {
  border-radius: 10px;
  min-height: 80px;
  border: 1px solid #dcdcdc;
  background: #fafafa;
  display: flex;
  flex-direction: column;
  justify-content: center;
  line-height: 1.3;
  font-size: 13px;
}
.bin-free {
  background: #e9f9ef;
  border-color: #8cd9a8;
  color: #1a7f3c;
}
.bin-busy {
  background: #fff7e6;
  border-color: #ffd48a;
  color: #704c00;
}
.bin-order-info {
  margin-top: 6px;
  padding-top: 4px;
  border-top: 1px dashed #e2e2e2;
  font-size: 11px;
  line-height: 1.3;
  color: #444;
}
.bin-order-info .order-customer {
  font-size: 10px;
  color: #777;
}

.q-badge.text-caption {
  font-size: 11px;
  font-weight: 600;
}
</style>
