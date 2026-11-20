<script setup>
import { reactive, ref, onMounted, onUnmounted } from 'vue'

const cellSize = 36

const map = reactive({
  width: 16,
  height: 12,
  obstacles: [],
})

const robot = reactive({
  x: 1,
  y: 1,
  heading: 90,
  status: 'idle',
})

const control = reactive({
  busy: false,
  message: '',
})
const sensorBlocked = ref(false)
const sensorDistance = ref(null)

const fetchSensorData = async () => {
try {
const res = await fetch("http://127.0.0.1:8000/api/sensors/front")
const data = await res.json()
sensorBlocked.value = data.blocked
sensorDistance.value = data.distance
} catch { /* 后端未起：保持上次值 */ }
}

const sensorTimer = setInterval(fetchSensorData, 500)

async function sendAction(action) {
  if (control.busy) return

  control.busy = true
  control.message = ''

  try {
    const response = await fetch(
      'http://127.0.0.1:8000/api/robot/move',
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ action }),
      },
    )

    const data = await response.json()

    if (response.ok) {
      Object.assign(robot, data)
    } else {
      control.message = data.error?.message || '操作失败'

      // 拦截响应只有错误信息，需要另查机器人最新状态
      const stateResponse = await fetch(
        'http://127.0.0.1:8000/api/robot',
      )
      if (!stateResponse.ok) {
        throw new Error('状态读取失败')
      }
      Object.assign(robot, await stateResponse.json())
    }
  } catch (error) {
    console.error(error)
    control.message = '连接后端失败，请检查后端终端'
  } finally {
    control.busy = false
  }
}

function moveForward() {
  return sendAction('forward')
}

function moveBack() {
  return sendAction('back')
}

function turnLeft() {
  return sendAction('turn_left')
}

function turnRight() {
  return sendAction('turn_right')
}

async function loadData() {
  try {
    const mapResponse = await fetch('http://127.0.0.1:8000/api/map')
    if (!mapResponse.ok) {
      throw new Error('地图读取失败')
    }
    const mapData = await mapResponse.json()
    Object.assign(map, mapData)

    const robotResponse = await fetch('http://127.0.0.1:8000/api/robot')
    if (!robotResponse.ok) {
      throw new Error('机器人状态读取失败')
    }
    const robotData = await robotResponse.json()
    Object.assign(robot, robotData)
  } catch (error) {
    console.error(error)
    alert('读取后端失败，请检查后端是否正在运行')
  }
}

let timer = null
let polling = false
let disposed = false

async function refreshRobot() {
  if (polling) return

  polling = true

  try {
    const response = await fetch(
      'http://127.0.0.1:8000/api/robot',
    )

    if (!response.ok) {
      throw new Error(`机器人状态读取失败：${response.status}`)
    }

    const data = await response.json()

    if (!disposed) {
      Object.assign(robot, data)
    }
  } catch (error) {
    console.error('轮询失败', error)
  } finally {
    polling = false
  }
}

onMounted(async () => {
  await loadData()

  if (!disposed) {
    timer = setInterval(refreshRobot, 500)
  }
})

onUnmounted(() => {
  disposed = true
  clearInterval(timer)
})
const patrolPoints = [[4, 8], [12, 3], [14, 10]]
const patrolStatus = ref('idle')
const patrolMessage = ref('')
const patrolProgress = ref({ active: false, total: 3, current: 0, done: false })
let patrolTimer = null
let patrolStartedAt = 0

function mockGetPatrolProgress() {
  const total = patrolProgress.value.total
  const current = Math.min(total, Math.floor((Date.now() - patrolStartedAt) / 2000))
  const done = current === total

  return { active: !done, total, current, done }
}

function stopPatrolPolling() {
  clearInterval(patrolTimer)
  patrolTimer = null
}

function startPatrolPolling(total) {
  stopPatrolPolling()
  patrolProgress.value = { active: true, total, current: 0, done: false }

  let pending = false

  const timer = setInterval(async () => {
    if (pending) return
    pending = true

    try {
      const response = await fetch('http://127.0.0.1:8000/api/patrol/progress')
      if (!response.ok) throw new Error(`HTTP ${response.status}`)

      const progress = await response.json()
      if (patrolTimer !== timer) return

      patrolProgress.value = progress

      if (!progress.active) {
        patrolStatus.value = 'idle'
        stopPatrolPolling()
      }
    } catch (error) {
      console.error('读取巡检进度失败', error)
    } finally {
      pending = false
    }
  }, 500)

  patrolTimer = timer
}

onUnmounted(stopPatrolPolling)
async function startPatrol() {
  patrolMessage.value = ''

  try {
    const response = await fetch('http://127.0.0.1:8000/api/patrol/start', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ points: patrolPoints }),
    })
    const data = await response.json()

    if (!response.ok) {
      patrolMessage.value = data.error?.message ?? `启动失败（HTTP ${response.status}）`
      return
    }

    patrolStatus.value = data.status
    startPatrolPolling(data.total)
  } catch (error) {
    console.error('启动巡检失败', error)
    patrolMessage.value = '启动巡检失败，请检查后端是否已启动'
  }
}
async function stopPatrol() {
  patrolMessage.value = ''

  try {
    const response = await fetch('http://127.0.0.1:8000/api/patrol/stop', {
      method: 'POST',
    })

    if (!response.ok) {
      const data = await response.json().catch(() => ({}))
      patrolMessage.value =
        data.error?.message ?? `停止失败（HTTP ${response.status}）`
      return
    }

    stopPatrolPolling()
    patrolStatus.value = 'idle'
    patrolProgress.value = {
      ...patrolProgress.value,
      active: false,
      done: false,
    }
  } catch (error) {
    console.error('停止巡检失败', error)
    patrolMessage.value = '停止巡检失败，请检查后端连接'
  }
}
onUnmounted(() => clearInterval(sensorTimer))
</script>

<template>
  <main>
    <h1>智能巡检机器人</h1>
   <p>后端接口联调 · 手动控制</p>

    <svg
      :viewBox="`0 0 ${map.width * cellSize} ${map.height * cellSize}`"
      aria-label="巡检地图"
      role="img"
      style="display: block; width: 100%; max-width: 576px; background: white; border: 2px solid #334155;"
    >
      <!-- 竖线：16 列需要 17 条边线 -->
      <line
        v-for="n in map.width + 1"
        :key="`column-${n}`"
        :x1="(n - 1) * cellSize"
        y1="0"
        :x2="(n - 1) * cellSize"
        :y2="map.height * cellSize"
        stroke="#cbd5e1"
      />

      <!-- 横线：12 行需要 13 条边线 -->
      <line
        v-for="n in map.height + 1"
        :key="`row-${n}`"
        x1="0"
        :y1="(n - 1) * cellSize"
        :x2="map.width * cellSize"
        :y2="(n - 1) * cellSize"
        stroke="#cbd5e1"
      />

       <rect
  v-for="obstacle in map.obstacles"
  :key="obstacle.join(',')"
  :x="obstacle[0] * cellSize"
  :y="obstacle[1] * cellSize"
  :width="cellSize"
  :height="cellSize"
  fill="#64748b"
  />

      <!-- 机器人：放在对应格子内，四周留一点空隙 -->
      <rect
        :x="robot.x * cellSize + 4"
        :y="robot.y * cellSize + 4"
        :width="cellSize - 8"
        :height="cellSize - 8"
        rx="6"
        fill="#f97316"
      />
      <text
  :x="(robot.x + 0.5) * cellSize"
  :y="(robot.y + 0.5) * cellSize"
  text-anchor="middle"
  dominant-baseline="central"
  font-size="20"
  fill="white"
>
  {{ { 0: '↑', 90: '→', 180: '↓', 270: '←' }[robot.heading] }}
</text>

    </svg>
   <button :disabled="control.busy || patrolStatus === 'patrolling' || robot.status === 'patrolling'" @click="turnLeft">左转</button>
<button :disabled="control.busy || patrolStatus === 'patrolling' || robot.status === 'patrolling'" @click="moveForward">前进</button>
<button :disabled="control.busy || patrolStatus === 'patrolling' || robot.status === 'patrolling'"@click="moveBack">后退</button>
<button :disabled="control.busy || patrolStatus === 'patrolling' || robot.status === 'patrolling'"@click="turnRight">右转</button>

<p role="status">{{ control.message }}</p>
    <p>
      位置：({{ robot.x }}, {{ robot.y }})
      · 朝向：{{ robot.heading }}°
      · 状态：{{ robot.status }}
    </p>
    <section>
  <h2>自动巡检</h2>
  <p>巡检点：(4,8) → (12,3) → (14,10)</p>

  <button type="button" @click="startPatrol">开始巡检</button>
<p>状态：{{ patrolStatus === 'patrolling' ? '巡检中' : '空闲' }}</p>
<p v-if="patrolMessage" role="alert">{{ patrolMessage }}</p>
  <button
  type="button"
  :disabled="patrolStatus !== 'patrolling'"
  @click="stopPatrol"
>
  停止巡检
</button>

 <p>
  <label for="patrol-progress">
    巡检进度：{{ patrolProgress.current }} / {{ patrolProgress.total }}
  </label>
  <progress
    id="patrol-progress"
    :value="patrolProgress.current"
    :max="patrolProgress.total"
  />
  <span v-if="patrolProgress.done">巡检完成</span>
</p>
</section>
    <div class="sensor-box">
  <p>前方传感器</p >
  <p>blocked：{{ sensorBlocked }}</p >
  <p>distance：{{ sensorDistance }}</p >
</div>
  </main>
</template>
<style scoped>
.sensor-box {
  border: 1px solid #222;
  padding: 8px 12px;
  margin-top: 8px;
  display: inline-block;
}
</style>
