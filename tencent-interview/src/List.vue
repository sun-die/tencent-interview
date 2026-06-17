<template>
  <div class="list-container">
    <div
      v-for="(item, index) in list"
      :key="index"
      class="list-item"
      :style="{ backgroundColor: item.background }"
    >
      序号：{{ index }}
    </div>
    <div v-if="loading" class="loading">Loading ...</div>
    <div v-else-if="isStopped" class="loading">已加载全部，共 50 条</div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
// 注意：Vue 核心没有 throttle，此处使用 VueUse 或 自行实现
// 为了零依赖，这里手动实现一个简单节流函数
function throttle<T extends (...args: any[]) => any>(fn: T, delay: number) {
  let timer: ReturnType<typeof setTimeout> | null = null
  return function (this: any, ...args: Parameters<T>) {
    if (timer) return
    timer = setTimeout(() => {
      fn.apply(this, args)
      timer = null
    }, delay)
  }
}

interface ListItem {
  background: string
}

// ---------- 1. 状态定义 ----------
const list = ref<ListItem[]>([{ background: 'rgb(233,32,38)' }])
const loading = ref(false)
const isStopped = ref(false)
// 新增：冷却标志，防止加载完成后立即再次触发
let cooldown = false

// ---------- 2. 模拟请求 ----------
const randomRgb = (): string => {
  const r = Math.floor(Math.random() * 256)
  const g = Math.floor(Math.random() * 256)
  const b = Math.floor(Math.random() * 256)
  return `rgb(${r}, ${g}, ${b})`
}

const getList = (num: number): Promise<ListItem[]> => {
  return new Promise((resolve) => {
    const delay = Math.random() * 5000
    setTimeout(() => {
      resolve(Array.from({ length: num }, () => ({ background: randomRgb() })))
    }, delay)
  })
}

// ---------- 3. 加载更多（核心） ----------
const loadMore = async () => {
  // 防止并发、防止超出、防止冷却期
  if (loading.value || isStopped.value || cooldown) return
  // 已满 50 个停止
  if (list.value.length >= 50) {
    isStopped.value = true
    return
  }

  loading.value = true
  try {
    const newItems = await getList(10)
    list.value.push(...newItems)
    if (list.value.length >= 50) {
      isStopped.value = true
    }
    // 加载完成后，开启 300ms 冷却期，避免立即触发
    cooldown = true
    setTimeout(() => {
      cooldown = false
    }, 300)
  } finally {
    loading.value = false
  }
}

// ---------- 4. 滚动检测（优化后的百分比计算） ----------
const getScrollPercent = (): number => {
  const scrollTop = window.scrollY || document.documentElement.scrollTop
  const scrollHeight = document.documentElement.scrollHeight
  const clientHeight = window.innerHeight
  const maxScroll = scrollHeight - clientHeight
  if (maxScroll <= 0) return 0
  // 保留两位小数，防止浮点数误差
  return Math.round((scrollTop / maxScroll) * 100) / 100
}

// 使用节流：限制检测频率为 200ms
const handleScroll = throttle(() => {
  // 条件 1：已停止 -> 无操作
  if (isStopped.value) return
  // 条件 2：当前滚动百分比 大于 50% 且 未在加载中 且 不在冷却期
  if (getScrollPercent() > 0.5 && !loading.value && !cooldown) {
    loadMore()
  }
}, 200)

// ---------- 5. 生命周期 ----------
onMounted(async () => {
  // 初始化加载第一批
  await loadMore()
  window.addEventListener('scroll', handleScroll)
})

onUnmounted(() => {
  window.removeEventListener('scroll', handleScroll)
})
</script>

<style scoped>
.list-item {
  width: 100%;
  height: 500px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24px;
  color: #fff;
  border-bottom: 1px solid rgba(0,0,0,0.1);
}
.loading {
  position: fixed;
  bottom: 0;
  left: 0; right: 0;
  padding: 12px;
  background: rgba(0,0,0,0.8);
  color: #fff;
  text-align: center;
}
</style>