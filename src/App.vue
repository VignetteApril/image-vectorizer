<script setup>
import { ref, onBeforeUnmount } from 'vue'
// 引入两个核心库
import { removeBackground } from '@imgly/background-removal' // AI 抠图
import ImageTracer from 'imagetracerjs' // 矢量转换

// --- 全局状态 ---
const currentTab = ref('remove-bg') // 当前标签页: 'remove-bg' | 'vectorize'

// ==============================
// 功能模块 1: AI 抠图 (Background Remover)
// ==============================
const bgOriginalUrl = ref(null)
const bgResultUrl = ref(null)
const bgIsProcessing = ref(false)
const bgError = ref('')
const bgFileName = ref('image')

// 上传处理
const handleBgUpload = (event) => {
  const file = event.target.files[0]
  if (!file) return
  bgFileName.value = file.name.split('.')[0]
  const url = URL.createObjectURL(file)
  bgOriginalUrl.value = url
  bgResultUrl.value = null // 重置结果
  bgError.value = ''
}

// 执行抠图
const startRemoveBackground = async () => {
  if (!bgOriginalUrl.value || bgIsProcessing.value) return
  bgIsProcessing.value = true
  bgError.value = ''

  try {
    const blob = await removeBackground(bgOriginalUrl.value, {
      progress: (key, current, total) => {
        console.log(`Downloading ${key}: ${Math.round(current/total * 100)}%`)
      }
    })
    bgResultUrl.value = URL.createObjectURL(blob)
  } catch (error) {
    console.error(error)
    bgError.value = '抠图失败，请检查控制台'
  } finally {
    bgIsProcessing.value = false
  }
}

// 下载抠图结果
const downloadBgResult = () => {
  if (!bgResultUrl.value) return
  const link = document.createElement('a')
  link.href = bgResultUrl.value
  link.download = `${bgFileName.value}_nobg.png`
  link.click()
}

// === 联动功能：发送到矢量转换器 ===
const sendToVectorizer = () => {
  if (!bgResultUrl.value) return
  // 将抠图结果设置为矢量转换的原图
  vecOriginalUrl.value = bgResultUrl.value
  vecFileName.value = bgFileName.value + '_vector'
  vecSvgOutput.value = '' // 清空之前的矢量结果
  currentTab.value = 'vectorize' // 切换标签
}

// ==============================
// 功能模块 2: 转矢量图 (Vectorizer)
// ==============================
const vecOriginalUrl = ref(null)
const vecSvgOutput = ref('')
const vecIsProcessing = ref(false)
const vecFileName = ref('vector_image')

// 矢量图配置
const vecOptions = ref({
  colors: 16, // 颜色数量
  blur: 0     // 模糊度
})

// 上传处理
const handleVecUpload = (event) => {
  const file = event.target.files[0]
  if (!file) return
  vecFileName.value = file.name.split('.')[0]
  const reader = new FileReader()
  reader.onload = (e) => {
    vecOriginalUrl.value = e.target.result
    vecSvgOutput.value = ''
  }
  reader.readAsDataURL(file)
}

// 执行转换
const convertToSvg = () => {
  if (!vecOriginalUrl.value) return
  vecIsProcessing.value = true

  // ImageTracer 配置
  const options = {
    numberofcolors: vecOptions.value.colors,
    blurradius: vecOptions.value.blur,
    blurdelta: 10,
    transparent_color: null, // 是否透明
    scale: 1,
    viewbox: true
  }

  ImageTracer.imageToSVG(
    vecOriginalUrl.value,
    (svgString) => {
      vecSvgOutput.value = svgString
      vecIsProcessing.value = false
    },
    options
  )
}

// 下载 SVG
const downloadSvg = () => {
  if (!vecSvgOutput.value) return
  const blob = new Blob([vecSvgOutput.value], { type: 'image/svg+xml;charset=utf-8' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.href = url
  link.download = `${vecFileName.value}.svg`
  link.click()
  URL.revokeObjectURL(url)
}

// --- 清理 ---
onBeforeUnmount(() => {
  // 简单清理，实际项目中建议完整跟踪所有 URL
  if (bgOriginalUrl.value) URL.revokeObjectURL(bgOriginalUrl.value)
  if (bgResultUrl.value) URL.revokeObjectURL(bgResultUrl.value)
})
</script>

<template>
  <div class="container">
    <header>
      <h1>Web 图片工作台</h1>
      <p class="subtitle">纯本地运行 | AI 抠图 + SVG 转换</p>
    </header>

    <div class="tabs">
      <button
        :class="['tab-btn', { active: currentTab === 'remove-bg' }]"
        @click="currentTab = 'remove-bg'"
      >
        🖌️ AI 自动抠图
      </button>
      <button
        :class="['tab-btn', { active: currentTab === 'vectorize' }]"
        @click="currentTab = 'vectorize'"
      >
        📐 转矢量图 (SVG)
      </button>
    </div>

    <div class="workspace">

      <div v-show="currentTab === 'remove-bg'" class="module-box">
        <div class="toolbar">
          <input type="file" accept="image/*" @change="handleBgUpload" :disabled="bgIsProcessing" />
          <button class="action-btn" @click="startRemoveBackground" :disabled="!bgOriginalUrl || bgIsProcessing">
            {{ bgIsProcessing ? 'AI 计算中...' : '开始抠图' }}
          </button>
        </div>

        <div class="preview-grid" v-if="bgOriginalUrl">
          <div class="card">
            <span class="label">原图</span>
            <img :src="bgOriginalUrl" />
          </div>
          <div class="card checkerboard">
            <span class="label">结果</span>
            <img v-if="bgResultUrl" :src="bgResultUrl" />
            <div v-else class="placeholder">{{ bgIsProcessing ? '处理中...' : '等待开始' }}</div>
          </div>
        </div>

        <div class="result-actions" v-if="bgResultUrl">
          <button class="download-btn" @click="downloadBgResult">⬇️ 下载 PNG</button>
          <button class="link-btn" @click="sendToVectorizer">➡️ 发送到矢量转换器</button>
        </div>
        <p v-if="bgError" class="error">{{ bgError }}</p>
      </div>

      <div v-show="currentTab === 'vectorize'" class="module-box">
        <div class="toolbar">
          <input type="file" accept="image/*" @change="handleVecUpload" />

          <div class="settings">
            <label>色彩数: <input type="number" v-model="vecOptions.colors" min="2" max="64"></label>
          </div>

          <button class="action-btn" @click="convertToSvg" :disabled="!vecOriginalUrl || vecIsProcessing">
            {{ vecIsProcessing ? '转换中...' : '转为 SVG' }}
          </button>
        </div>

        <div class="preview-grid" v-if="vecOriginalUrl">
          <div class="card">
            <span class="label">位图输入</span>
            <img :src="vecOriginalUrl" />
          </div>
          <div class="card vector-bg">
            <span class="label">SVG 预览</span>
            <div v-if="vecSvgOutput" v-html="vecSvgOutput" class="svg-container"></div>
            <div v-else class="placeholder">等待转换</div>
          </div>
        </div>

        <div class="result-actions" v-if="vecSvgOutput">
          <button class="download-btn" @click="downloadSvg">⬇️ 下载 SVG 文件</button>
        </div>
      </div>

    </div>
  </div>
</template>

<style scoped>
/* 全局样式 */
.container {
  max-width: 900px;
  margin: 0 auto;
  padding: 2rem;
  font-family: 'Segoe UI', sans-serif;
  color: #333;
}

header { text-align: center; margin-bottom: 2rem; }
h1 { color: #2c3e50; margin: 0; }
.subtitle { color: #7f8c8d; margin-top: 0.5rem; }

/* 标签页样式 */
.tabs {
  display: flex;
  justify-content: center;
  gap: 1rem;
  margin-bottom: 1.5rem;
}

.tab-btn {
  padding: 0.8rem 1.5rem;
  border: none;
  background: #e0e0e0;
  border-radius: 8px;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.2s;
  font-weight: bold;
  color: #555;
}

.tab-btn.active {
  background: #42b983;
  color: white;
  transform: translateY(-2px);
  box-shadow: 0 4px 6px rgba(66, 185, 131, 0.3);
}

/* 工作区模块 */
.module-box {
  background: white;
  padding: 1.5rem;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.08);
  border: 1px solid #eee;
  animation: fadeIn 0.3s ease;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

/* 工具栏 */
.toolbar {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
  align-items: center;
  justify-content: center;
  margin-bottom: 1.5rem;
  background: #f8f9fa;
  padding: 1rem;
  border-radius: 8px;
}

.settings {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.9rem;
}
.settings input { width: 50px; padding: 4px; }

/* 按钮组 */
.action-btn {
  background: #2c3e50;
  color: white;
  border: none;
  padding: 0.6rem 1.2rem;
  border-radius: 4px;
  cursor: pointer;
}
.action-btn:disabled { background: #95a5a6; cursor: not-allowed; }

.download-btn {
  background: #42b983;
  color: white;
  border: none;
  padding: 0.8rem 2rem;
  border-radius: 6px;
  cursor: pointer;
  font-weight: bold;
}

.link-btn {
  background: #3498db;
  color: white;
  border: none;
  padding: 0.8rem 2rem;
  border-radius: 6px;
  cursor: pointer;
  font-weight: bold;
}

.result-actions {
  display: flex;
  justify-content: center;
  gap: 1rem;
  margin-top: 1.5rem;
}

/* 预览网格 */
.preview-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1.5rem;
}

.card {
  position: relative;
  background: #f0f0f0;
  height: 300px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  border: 2px dashed #ddd;
}

.label {
  position: absolute;
  top: 10px;
  left: 10px;
  background: rgba(0,0,0,0.6);
  color: white;
  padding: 2px 8px;
  border-radius: 4px;
  font-size: 0.8rem;
  z-index: 10;
}

img, .svg-container {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
}

/* 特殊背景 */
.checkerboard {
  background-image: linear-gradient(45deg, #ccc 25%, transparent 25%),
    linear-gradient(-45deg, #ccc 25%, transparent 25%),
    linear-gradient(45deg, transparent 75%, #ccc 75%),
    linear-gradient(-45deg, transparent 75%, #ccc 75%);
  background-size: 20px 20px;
  background-color: white;
}

.vector-bg { background: white; }

.error { color: red; text-align: center; margin-top: 1rem; }
.placeholder { color: #aaa; }

@media (max-width: 600px) {
  .preview-grid { grid-template-columns: 1fr; }
  .result-actions { flex-direction: column; }
}
</style>