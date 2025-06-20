<template>
  <div class="container">
    <header>
      <h1>闪卡学习应用</h1>
    </header>

    <!-- Loading state -->
    <LoadingOverlay v-if="isLoading" />

    <!-- Theme management buttons -->
    <div class="theme-management" v-if="!hasSelectedTheme">
      <button class="action-btn" @click="showNewThemeForm">上传新主题</button>
      <button v-if="hasProgress" class="continue-btn" @click="continueLastStudy">
        <span class="btn-icon">⏵</span> 继续学习
      </button>
      <button v-if="hasProgress" class="reset-btn" @click="resetProgress">
        <span class="btn-icon">↻</span> 重置进度
      </button>
    </div>

    <!-- Last study info -->
    <div v-if="hasProgress && !hasSelectedTheme" class="last-study-info">
      上次学习时间: {{ formatLastStudyTime }}
    </div>

    <!-- Custom theme card -->
    <div v-if="!hasSelectedTheme" class="custom-theme-card" @click="goToCustomTheme">
      <div class="custom-theme-icon">+</div>
      <div class="custom-theme-content">
        <h3>{{ customTheme.name }}</h3>
        <p>{{ customTheme.description }}</p>
      </div>
    </div>

    <!-- Theme selector -->
    <div v-if="!hasSelectedTheme" class="theme-selector">
      <h2>选择学习主题</h2>
      <div class="theme-cards">
        <ThemeCard v-for="theme in themes" :key="theme.id" :theme="theme" @select="goToTheme"
          @update="showUpdateThemeForm" @delete="deleteTheme" />
      </div>
    </div>

    <!-- New theme form -->
    <ThemeUploadForm v-if="showThemeUploadForm" @close="showThemeUploadForm = false" @submit="handleNewThemeUpload" />

    <!-- Update theme form -->
    <ThemeUpdateForm v-if="showThemeUpdateForm && themeToUpdate" :theme="themeToUpdate"
      @close="showThemeUpdateForm = false" @submit="handleThemeUpdate" />
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import { useThemeStore } from '../stores/theme'
import { useProgressStore } from '../stores/progress'
import LoadingOverlay from '../components/LoadingOverlay.vue'
import ThemeCard from '../components/ThemeCard.vue'
import ThemeUploadForm from '../components/ThemeUploadForm.vue'
import ThemeUpdateForm from '../components/ThemeUpdateForm.vue'

const router = useRouter()
const route = useRoute()
const themeStore = useThemeStore()
const progressStore = useProgressStore()

// Refs
const showThemeUploadForm = ref(false)
const showThemeUpdateForm = ref(false)
const themeToUpdate = ref(null)

// Computed properties
const themes = computed(() => themeStore.themes)
const isLoading = computed(() => themeStore.isLoading)
const hasSelectedTheme = computed(() => themeStore.selectedTheme !== '')
const hasProgress = computed(() => progressStore.hasProgress)
const formatLastStudyTime = computed(() => progressStore.formatLastStudyTime)

// Custom theme data
const customTheme = {
  id: 'custom',
  name: '自定义主题',
  description: '上传自己的TSV文件，创建自定义闪卡集合。',
  icon: 'upload',
  color: '#8e44ad',
  cards: null
}

// Methods
function showNewThemeForm() {
  showThemeUploadForm.value = true
}

function showUpdateThemeForm(theme) {
  themeToUpdate.value = theme
  showThemeUpdateForm.value = true
}

function goToTheme(themeId) {
  router.push({ name: 'theme', params: { id: themeId } })
}

function goToCustomTheme() {
  router.push({ name: 'custom' })
}

function continueLastStudy() {
  if (!hasProgress.value) return

  try {
    // Reload progress to get theme ID
    const progress = progressStore.loadProgress()
    if (progress && progress.themeId) {
      router.push({
        name: 'theme',
        params: { id: progress.themeId },
        query: { continueFrom: progress.cardIndex }
      })
    } else {
      alert('没有找到上次学习记录或记录已损坏')
    }
  } catch (error) {
    console.error('继续学习失败:', error)
    alert('继续学习失败，请重试')
  }
}

function resetProgress() {
  if (confirm('确定要重置学习进度吗？这将清除所有已保存的学习记录。')) {
    progressStore.resetProgress()
    alert('学习进度已重置。')
  }
}

async function handleNewThemeUpload(themeData) {
  try {
    await themeStore.saveUserTheme(themeData)
    showThemeUploadForm.value = false
    alert('主题上传成功！')
  } catch (error) {
    alert('主题上传失败: ' + error.message)
  }
}

async function handleThemeUpdate(themeData) {
  try {
    await themeStore.saveUserTheme(themeData)
    showThemeUpdateForm.value = false
    themeToUpdate.value = null
    alert('主题更新成功！')
  } catch (error) {
    alert('主题更新失败: ' + error.message)
  }
}

async function deleteTheme(themeId) {
  if (confirm('确定要删除这个主题吗？这将永久删除该主题及其所有闪卡。')) {
    try {
      await themeStore.deleteTheme(themeId)
      alert('主题已删除。')
    } catch (error) {
      alert('删除主题失败: ' + error.message)
    }
  }
}

// Lifecycle hooks
onMounted(async () => {
  try {
    await themeStore.loadAllThemes()
    progressStore.loadProgress()
  } catch (error) {
    alert('加载主题列表失败，请刷新页面重试。')
  }
})

// Watch for route changes to reload themes
watch(() => route.name, async (newRouteName) => {
  if (newRouteName === 'home') {
    try {
      await themeStore.loadAllThemes()
      progressStore.loadProgress()
    } catch (error) {
      console.error('Failed to reload themes:', error)
    }
  }
}, { immediate: true })
</script>

<style scoped>
/* Component-specific styles */
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

header {
  text-align: center;
  margin-bottom: 40px;
}

header h1 {
  font-size: 2.5rem;
  color: #333;
  margin-bottom: 10px;
}

/* Theme management styles */
.theme-management {
  display: flex;
  justify-content: center;
  margin-bottom: 30px;
  gap: 15px;
  flex-wrap: wrap;
}

.action-btn {
  padding: 12px 24px;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1rem;
  transition: all 0.3s;
  display: flex;
  align-items: center;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.action-btn:hover {
  background-color: #3e8e41;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.continue-btn {
  padding: 12px 24px;
  background-color: #2196F3;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1rem;
  transition: all 0.3s;
  display: flex;
  align-items: center;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.continue-btn:hover {
  background-color: #0b7dda;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.reset-btn {
  padding: 12px 24px;
  background-color: #9e9e9e;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1rem;
  transition: all 0.3s;
  display: flex;
  align-items: center;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.reset-btn:hover {
  background-color: #757575;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.btn-icon {
  margin-right: 8px;
  font-size: 1.1rem;
}

/* Last study info styles */
.last-study-info {
  text-align: center;
  margin-bottom: 30px;
  color: #666;
  font-style: italic;
  font-size: 1.1rem;
}

/* Custom theme card styles */
.custom-theme-card {
  max-width: 800px;
  margin: 0 auto 40px;
  background-color: #f9f0ff;
  border-radius: 15px;
  padding: 30px;
  box-shadow: 0 6px 12px rgba(142, 68, 173, 0.2);
  transition: all 0.3s;
  cursor: pointer;
  border: 2px dashed #8e44ad;
  border-left: 8px solid #8e44ad;
  display: flex;
  align-items: center;
  position: relative;
  overflow: hidden;
}

.custom-theme-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 12px 24px rgba(142, 68, 173, 0.3);
  background-color: #f5e6ff;
}

.custom-theme-card:after {
  content: '点击创建自定义闪卡';
  position: absolute;
  right: -50px;
  top: 15px;
  background-color: #8e44ad;
  color: white;
  padding: 5px 40px;
  transform: rotate(45deg);
  font-size: 0.9rem;
  font-weight: bold;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}

.custom-theme-icon {
  width: 80px;
  height: 80px;
  border-radius: 50%;
  background-color: #8e44ad;
  color: white;
  display: flex;
  justify-content: center;
  align-items: center;
  font-weight: bold;
  font-size: 2.5rem;
  margin-right: 30px;
  flex-shrink: 0;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.05);
  }
  100% {
    transform: scale(1);
  }
}

.custom-theme-content {
  flex: 1;
}

.custom-theme-content h3 {
  color: #333;
  margin-bottom: 15px;
  font-size: 2rem;
}

.custom-theme-content p {
  color: #666;
  font-size: 1.2rem;
  line-height: 1.5;
}

/* Theme selector styles */
.theme-selector {
  margin-bottom: 50px;
}

.theme-selector h2 {
  text-align: center;
  margin-bottom: 30px;
  font-size: 2rem;
  color: #333;
}

.theme-cards {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 25px;
  padding: 0 20px;
}

/* Responsive styles */
@media (max-width: 1024px) {
  .container {
    max-width: 900px;
  }
  
  .theme-cards {
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  }
}

@media (max-width: 768px) {
  .container {
    padding: 15px;
  }
  
  header h1 {
    font-size: 2rem;
  }
  
  .theme-management {
    flex-direction: column;
    align-items: center;
    gap: 10px;
  }
  
  .action-btn,
  .continue-btn,
  .reset-btn {
    width: 100%;
    max-width: 300px;
    justify-content: center;
  }
  
  .custom-theme-card {
    padding: 20px;
    margin-bottom: 30px;
  }
  
  .custom-theme-icon {
    width: 60px;
    height: 60px;
    font-size: 2rem;
    margin-right: 20px;
  }
  
  .custom-theme-content h3 {
    font-size: 1.6rem;
  }
  
  .custom-theme-content p {
    font-size: 1rem;
  }
  
  .theme-selector h2 {
    font-size: 1.8rem;
    margin-bottom: 20px;
  }
  
  .theme-cards {
    grid-template-columns: 1fr;
    padding: 0;
  }
}

@media (max-width: 480px) {
  header h1 {
    font-size: 1.8rem;
  }
  
  .custom-theme-card {
    flex-direction: column;
    text-align: center;
  }
  
  .custom-theme-icon {
    margin-right: 0;
    margin-bottom: 15px;
  }
  
  .custom-theme-content h3 {
    font-size: 1.4rem;
  }
  
  .custom-theme-content p {
    font-size: 0.9rem;
  }
}
</style>
