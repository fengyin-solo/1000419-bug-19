<template>
  <section class="page" data-module="plant">
    <header class="page-head">
      <div>
        <h2>厂区单元管理</h2>
        <p class="page-desc">维护工艺单元，围绕单元编码、单元名称、处理工艺、设计处理量做登记、筛选与状态流转。</p>
      </div>
      <div class="page-actions">
        <button class="btn primary" type="button" @click="openCreate">登记工艺单元</button>
        <button class="btn" type="button" @click="exportRows">导出厂区单元清单</button>
      </div>
    </header>

    <div class="stat-row">
      <article v-for="item in stats" :key="item.label" class="stat-card">
        <span class="stat-label">{{ item.label }}</span>
        <strong class="stat-value">{{ item.value }}</strong>
      </article>
    </div>

    <form class="filter-bar" @submit.prevent="applyFilters">
      <label v-for="field in filterFields" :key="field" class="filter-item">
        <span>{{ field }}</span>
        <input v-model="filters[field]" :placeholder="`按${field}检索`" />
      </label>
      <button class="btn" type="submit">查询</button>
      <button class="btn ghost" type="button" @click="resetFilters">重置条件</button>
    </form>

    <table class="data-table">
      <thead>
        <tr>
          <th v-for="column in columns" :key="column">{{ column }}</th>
          <th>可执行动作</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="row in rows" :key="String(row.id)">
          <td v-for="column in columns" :key="column">{{ row[column] ?? '—' }}</td>
          <td class="row-actions">
            <button
              v-for="action in actions"
              :key="action"
              class="link"
              type="button"
              @click="runAction(action, row)"
            >
              {{ action }}
            </button>
          </td>
        </tr>
        <tr v-if="!rows.length">
          <td :colspan="columns.length + 1" class="empty-state">暂无厂区单元数据，可先登记工艺单元</td>
        </tr>
      </tbody>
    </table>

    <footer class="page-foot">
      <span>共 {{ total }} 条厂区单元记录</span>
      <span v-if="errorMessage" class="error-text">{{ errorMessage }}</span>
    </footer>
  </section>
</template>

<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'

import { request } from '@/api/client'

type Row = Record<string, string | number | null>

const ENDPOINT = '/api/plant'
const columns = ["单元编码", "单元名称", "处理工艺", "设计处理量", "实际处理量", "运行班组", "投运日期", "单元状态"]
const actions = ["完成调试", "安排减量", "停用单元"]
const statuses = ["待调试", "正常运行", "减量运行", "已停用"]
const stats = [{"label": "运行单元", "value": 0}, {"label": "减量运行单元", "value": 0}, {"label": "设计处理总量", "value": 0}]

// 前端筛选项与后端查询参数的对应关系，列表、重置、导出都走这一份
const filterParams: Record<string, string> = {
  单元编码: 'code',
  单元名称: 'name',
  处理工艺: 'process',
}
const filterFields = Object.keys(filterParams)

const route = useRoute()
const router = useRouter()

const rows = ref<Row[]>([])
const total = ref(0)
const errorMessage = ref('')
const filters = ref<Record<string, string>>(Object.fromEntries(filterFields.map((field) => [field, ''])))

function syncFiltersFromRoute() {
  for (const field of filterFields) {
    const value = route.query[filterParams[field]]
    filters.value[field] = typeof value === 'string' ? value : ''
  }
}

// 列表与导出共用同一套条件，保证列表页定位到的结果与导出完全一致
function activeParams() {
  const params: Record<string, string> = {}
  for (const field of filterFields) {
    const value = filters.value[field]?.trim() ?? ''
    if (value) {
      params[filterParams[field]] = value
    }
  }
  return params
}

function buildQuery() {
  return new URLSearchParams(activeParams()).toString()
}

function applyFilters() {
  void router.replace({ query: activeParams() })
}

function resetFilters() {
  for (const field of filterFields) {
    filters.value[field] = ''
  }
  void router.replace({ query: {} })
}

function exportRows() {
  window.open(`${ENDPOINT}/export?${buildQuery()}`, '_blank')
}

function openCreate() {
  errorMessage.value = '工艺单元登记入口尚未接入审批流'
}

async function runAction(action: string, row: Row) {
  errorMessage.value = ''
  try {
    const response = await request(`${ENDPOINT}/${row.id}/actions`, {
      method: 'POST',
      body: JSON.stringify({ action }),
    })
    if (!response.ok) {
      throw new Error('厂区单元动作未生效，请稍后重试')
    }
    await reload()
  } catch (error) {
    errorMessage.value = error instanceof Error ? error.message : '厂区单元操作失败'
  }
}

async function reload() {
  errorMessage.value = ''
  syncFiltersFromRoute()
  const query = buildQuery()
  try {
    const response = await request(`${ENDPOINT}?${query}`)
    if (!response.ok) {
      throw new Error('工艺单元列表读取失败')
    }
    const payload = await response.json()
    rows.value = payload.items ?? []
    total.value = payload.total ?? rows.value.length
  } catch (error) {
    errorMessage.value = error instanceof Error ? error.message : '厂区单元列表读取失败'
  }
}

// 地址栏查询参数变化（查询、重置、前进/后退、刷新进入）都按同一入口重新拉取
watch(() => route.query, () => {
  void reload()
}, { deep: true })

onMounted(reload)
</script>
