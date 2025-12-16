<template>
  <q-layout view="hHh lpR fFf">

    <!-- Header -->
    <q-header elevated>
      <q-toolbar>
        <q-toolbar-title>智慧倉儲管理系統</q-toolbar-title>

        <!-- 後端狀態顯示 -->
        <q-btn
          dense
          flat
          icon="cloud"
          :color="apiConnected ? 'green' : 'red'"
          label="後端連線"
        />
      </q-toolbar>
    </q-header>

    <!-- 左側 Drawer -->
    <q-drawer show-if-above v-model="drawerOpen" bordered>
      <q-list>
        <q-item-label header>功能選單</q-item-label>

        <q-item clickable v-ripple @click="goToPage('/')">
          <q-item-section avatar><q-icon name="home" /></q-item-section>
          <q-item-section>首頁</q-item-section>
        </q-item>

        <q-item clickable v-ripple @click="goToPage('/order')">
          <q-item-section avatar><q-icon name="add_shopping_cart" /></q-item-section>
          <q-item-section>新增訂單</q-item-section>
        </q-item>
      </q-list>
    </q-drawer>

    <!-- Page Container -->
    <q-page-container>
      <router-view />
    </q-page-container>

  </q-layout>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import axios from 'axios'

const drawerOpen = ref(true)
const apiConnected = ref(false)
const router = useRouter()

const checkApi = async () => {
  try {
    await axios.get('http://127.0.0.1:8001/api/inventory', { timeout: 2000 })
    apiConnected.value = true
  } catch {
    apiConnected.value = false
  }
}

const goToPage = (path) => router.push(path)

onMounted(() => {
  checkApi()
  setInterval(checkApi, 5000)
})
</script>

<style scoped>
.q-toolbar-title {
  font-weight: bold;
  letter-spacing: 1px;
}

.q-drawer {
  background-color: #f9f9f9;
}

.q-item:hover {
  background-color: #e0e0e0;
}
</style>
