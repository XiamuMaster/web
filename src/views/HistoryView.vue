<template>
  <div class="history-container">
    <div v-if="detectionHistory.length === 0" class="empty-state">
      <i>📋</i>
      <p>暂无检测历史</p>
    </div>
    <div v-else class="history-wrapper">
      <h2 class="page-title">检测历史</h2>
      <div class="history-list">
      <div 
        v-for="(history, index) in detectionHistory" 
        :key="history.id" 
        class="history-item"
        @click="loadHistoryItem(history)"
      >
        <div class="history-icon">
          {{ history.detectType === 'file' ? '📄' : '📹' }}
        </div>
        <div class="history-content">
          <div class="history-header">
            <span class="history-filename">{{ history.fileName }}</span>
            <span class="detection-mode-badge" :class="history.detectionMode">
              {{ getDetectionModeLabel(history.detectionMode) }}
            </span>
          </div>
          <div class="history-meta">
            {{ formatDate(history.timestamp) }} · 检测到 {{ history.results.length }} 个物品
          </div>
          <div class="history-stats">
            <span class="stat-badge">耗时: {{ history.processingTime }}ms</span>
          </div>
        </div>
        <div class="history-arrow">→</div>
      </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { ElMessage } from 'element-plus';

const detectionHistory = ref([]);

const formatDate = (timestamp) => {
  return new Date(timestamp).toLocaleString('zh-CN', {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit'
  });
};

const getDetectionModeLabel = (mode) => {
  if (mode === 'license_plate') return '车牌检测';
  return '普通检测';
};

const loadHistoryItem = (history) => {
  // 通过事件向父组件传递选中的历史记录
  // 这里需要将数据保存到 localStorage，然后在 HomeView 中读取
  localStorage.setItem('yolo_selected_history', JSON.stringify(history));
  // 触发自定义事件
  window.dispatchEvent(new CustomEvent('load-history', { detail: history }));
  ElMessage({ message: `已加载历史记录: ${history.fileName}`, icon: '' });
};

onMounted(() => {
  const savedHistory = localStorage.getItem('yolo_detection_history');
  if (savedHistory) {
    try {
      detectionHistory.value = JSON.parse(savedHistory);
    } catch (e) {
      console.error('加载历史记录失败:', e);
    }
  }
});
</script>

<style scoped>
.history-container {
  padding: 0;
}

.page-title {
  font-size: 28px;
  color: #333;
  margin: 0 0 20px 0;
  font-weight: 700;
}

.empty-state {
  text-align: center;
  padding: 80px 20px;
  color: #999;
}

.empty-state i {
  font-size: 64px;
  margin-bottom: 20px;
  display: block;
}

.empty-state p {
  font-size: 16px;
}

.history-wrapper {
  background: #ffffff;
  border-radius: 16px;
  padding: 24px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.06);
}

.history-list {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.history-item {
  background: white;
  border-radius: 12px;
  padding: 20px;
  display: flex;
  align-items: center;
  gap: 20px;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
}

.history-item:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.12);
  background: #f8fafc;
}

.history-icon {
  font-size: 32px;
  width: 50px;
  height: 50px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #667eea, #764ba2);
  border-radius: 10px;
}

.history-content {
  flex: 1;
}

.history-header {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 5px;
}

.history-filename {
  font-weight: 600;
  color: #333;
  font-size: 16px;
}

.detection-mode-badge {
  font-size: 11px;
  padding: 3px 8px;
  border-radius: 10px;
  font-weight: 500;
}

.detection-mode-badge {
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
}

.detection-mode-badge.license_plate {
  background: linear-gradient(135deg, #FF6B6B, #FF8E53);
  color: white;
}

.history-meta {
  color: #666;
  font-size: 13px;
  margin-bottom: 8px;
}

.history-stats {
  display: flex;
  gap: 10px;
}

.stat-badge {
  background: linear-gradient(135deg, #f0f4ff, #e3f2fd);
  color: #667eea;
  padding: 4px 12px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 500;
}

.history-arrow {
  font-size: 24px;
  color: #999;
  transition: transform 0.3s ease;
}

.history-item:hover .history-arrow {
  transform: translateX(5px);
  color: #667eea;
}
</style>
