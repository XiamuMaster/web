<template>
  <div class="home-container">
    <!-- 左侧上传区域 -->
    <div class="upload-section">
      <div class="upload-zone" @click="triggerFileInput" @dragover.prevent @drop.prevent="handleDrop">
        <i>🚗</i>
        <h3>上传图片或视频进行车牌检测</h3>
        <p>支持格式：JPG、PNG、MP4、AVI，最大文件大小：500MB</p>
        <p>或直接将文件拖放到此处</p>
      </div>

      <input type="file" ref="fileInputRef" @change="handleFileSelect" accept="image/*,video/*"
        style="display: none;">

      <div v-if="selectedFile" class="file-info">
        <span>📄 {{ selectedFile.name }}</span>
        <span style="color: #666; font-size: 12px;">
          ({{ formatFileSize(selectedFile.size) }})
        </span>
      </div>

      <!-- 控制按钮组 -->
      <div class="control-buttons">
        <button class="control-button btn-primary" @click="startWebcam" :disabled="isDetecting">
          📹 {{ isWebcamActive ? '关闭摄像头' : '开启摄像头' }}
        </button>
        <button class="control-button btn-primary" @click="startDetection"
          :disabled="!selectedFile && !isWebcamActive">
          🚀 {{ isDetecting ? '检测中...' : '开始检测' }}
        </button>
        <button class="control-button btn-secondary" @click="clearAll">
          🗑️ 清空
        </button>
        <button class="control-button btn-secondary" @click="exportResults" :disabled="!detectionResults.length">
          📥 下载结果图片
        </button>
      </div>

      <!-- 预览区域 -->
      <div class="preview-section">
        <div class="preview-header">
          <h3>预览</h3>
          <div style="display: flex; align-items: center; gap: 10px;">
            <span v-if="isDetecting" style="color: #667eea; font-size: 12px;">
              正在检测中...
            </span>
            <span v-if="isWebcamActive" style="color: #4CAF50; font-size: 12px;">
              📹 摄像头运行中
            </span>
          </div>
        </div>
        <div class="preview-container">
          <!-- 加载状态 -->
          <div v-if="isDetecting" class="loading-overlay">
            <div class="loading-spinner"></div>
            <p style="margin-top: 10px; color: #666;">正在检测中...</p>
          </div>

          <!-- 摄像头预览 -->
          <video v-else-if="isWebcamActive && !selectedFile" ref="webcamVideoRef" autoplay
            style="width: 100%; height: 100%; object-fit: cover;"></video>

          <!-- 图片预览 -->
          <img v-else-if="selectedFile && selectedFile.type.includes('image')" :src="previewUrl" alt="预览图片"
            class="preview-image">

          <!-- 视频预览 -->
          <video v-else-if="(selectedFile && selectedFile.type.includes('video')) || isResultVideo" 
            ref="previewVideoRef"
            :src="previewUrl" 
            class="preview-video"
            @timeupdate="updateVideoProgress"
            @loadedmetadata="updateVideoDuration"
            @error="handleVideoError"
            @canplay="handleVideoReady"
            controls
            autoplay>
            您的浏览器不支持视频播放
          </video>

          <!-- 没有选择文件 -->
          <div v-else style="color: #999; text-align: center;">
            <p>请上传文件或开启摄像头进行检测</p>
          </div>
        </div>

        <!-- 视频控制（仅当上传视频时显示） -->
        <div v-if="selectedFile && selectedFile.type.includes('video')" class="video-controls">
          <button class="play-button" @click="toggleVideoPlayback">
            {{ isVideoPlaying ? '⏸️' : '▶️' }}
          </button>
          <div class="progress-bar" @click="seekVideo">
            <div class="progress-fill" :style="{ width: videoProgress + '%' }"></div>
          </div>
          <div class="time-display">
            {{ formatTime(videoCurrentTime) }} / {{ formatTime(videoDuration) }}
          </div>
        </div>

        <!-- 视频错误提示 -->
        <div v-if="videoError" style="margin-top: 10px; padding: 10px; background: #ffebee; border-radius: 4px; color: #c62828; font-size: 14px;">
          ⚠️ 视频加载失败: {{ videoError }}
        </div>
      </div>
    </div>

    <!-- 右侧结果区域 -->
    <div class="result-section">
      <div class="result-header">
        <h3>检测结果</h3>
        <span style="color: #667eea; font-size: 12px;">
          置信度阈值：{{ confidenceThreshold.toFixed(2) }}
        </span>
      </div>

      <!-- 统计信息 -->
      <div class="stats-card">
        <div class="stat-item">
          <span class="stat-label">检测车牌数</span>
          <span class="stat-value">{{ detectionResults.length }}</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">平均置信度</span>
          <span class="stat-value">
            {{ averageConfidence.toFixed(3) }}
          </span>
        </div>
        <div class="stat-item">
          <span class="stat-label">处理时间</span>
          <span class="stat-value">{{ processingTime }}ms</span>
        </div>
      </div>

      <!-- 检测结果列表 -->
      <div class="detection-list">
        <div v-for="(result, index) in detectionResults" :key="index" class="detection-item">
          <div class="detection-color" :style="{ backgroundColor: getColorByConfidence(result.confidence) }"></div>
          <div>
            <div style="font-weight: 500;">
              {{ result.class }} ({{ result.confidence.toFixed(3) }})
            </div>
            <div style="font-size: 12px; color: #666;">
              位置: [{{ result.bbox[0] }}, {{ result.bbox[1] }}, {{ result.bbox[2] }}, {{ result.bbox[3] }}]
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';
import { ElMessage } from 'element-plus';
import axios from 'axios';

// 响应式数据
const selectedFile = ref(null);
const previewUrl = ref('');
const isDetecting = ref(false);
const isWebcamActive = ref(false);
const isVideoPlaying = ref(false);
const confidenceThreshold = ref(0.25);
const detectionResults = ref([]);
const processingTime = ref(0);
const videoCurrentTime = ref(0);
const videoDuration = ref(0);
const videoProgress = ref(0);
const videoError = ref('');
const isResultVideo = ref(false);

// DOM 引用
const fileInputRef = ref(null);
const previewVideoRef = ref(null);
const webcamVideoRef = ref(null);

// API 基础路径
const API_BASE_URL = 'http://127.0.0.1:5001';

// 计算属性
const averageConfidence = computed(() => {
  if (detectionResults.value.length === 0) return 0;
  const sum = detectionResults.value.reduce((acc, item) => acc + item.confidence, 0);
  return sum / detectionResults.value.length;
});

// 文件选择处理
const triggerFileInput = () => {
  fileInputRef.value.click();
};

const handleFileSelect = (event) => {
  const file = event.target.files[0];
  if (file) {
    processFile(file);
  }
};

const handleDrop = (event) => {
  const file = event.dataTransfer.files[0];
  if (file) {
    processFile(file);
  }
};

const processFile = (file) => {
  selectedFile.value = file;
  if (previewUrl.value) {
    URL.revokeObjectURL(previewUrl.value);
  }
  previewUrl.value = URL.createObjectURL(file);
  detectionResults.value = [];
  videoError.value = '';
  isVideoPlaying.value = false;
  videoProgress.value = 0;
  isResultVideo.value = false;
};

// 格式辅助函数
const formatFileSize = (bytes) => {
  if (bytes === 0) return '0 Bytes';
  const k = 1024;
  const sizes = ['Bytes', 'KB', 'MB', 'GB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i];
};

const formatTime = (seconds) => {
  if (!seconds || isNaN(seconds)) return '00:00';
  const mins = Math.floor(seconds / 60);
  const secs = Math.floor(seconds % 60);
  return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
};

// 摄像头控制
const startWebcam = async () => {
  if (isWebcamActive.value) {
    if (webcamVideoRef.value && webcamVideoRef.value.srcObject) {
      webcamVideoRef.value.srcObject.getTracks().forEach(track => track.stop());
    }
    isWebcamActive.value = false;
    ElMessage({ message: '摄像头已关闭', icon: '' });
  } else {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({
        video: { width: 640, height: 480 }
      });
      if (webcamVideoRef.value) {
        webcamVideoRef.value.srcObject = stream;
        isWebcamActive.value = true;
        ElMessage({ message: '摄像头已开启，可进行检测', icon: '' });
      }
    } catch (error) {
      ElMessage({ message: '无法访问摄像头: ' + error.message, type: 'error', icon: '' });
    }
  }
};

// 视频控制
const toggleVideoPlayback = () => {
  if (previewVideoRef.value) {
    if (isVideoPlaying.value) {
      previewVideoRef.value.pause();
    } else {
      previewVideoRef.value.play();
    }
    isVideoPlaying.value = !isVideoPlaying.value;
  }
};

const updateVideoProgress = () => {
  if (previewVideoRef.value) {
    videoCurrentTime.value = previewVideoRef.value.currentTime;
    videoDuration.value = previewVideoRef.value.duration;
    if (videoDuration.value > 0) {
      videoProgress.value = (videoCurrentTime.value / videoDuration.value) * 100;
    }
  }
};

const updateVideoDuration = () => {
  if (previewVideoRef.value) {
    videoDuration.value = previewVideoRef.value.duration;
  }
};

const seekVideo = (event) => {
  if (previewVideoRef.value) {
    const rect = event.currentTarget.getBoundingClientRect();
    const percent = (event.clientX - rect.left) / rect.width;
    previewVideoRef.value.currentTime = percent * videoDuration.value;
  }
};

const handleVideoError = (event) => {
  const video = event.target;
  console.error('视频加载错误:', event);
  videoError.value = '视频加载失败: ' + (video.error?.message || '未知错误');
};

const handleVideoReady = () => {
  console.log('视频准备就绪');
  videoError.value = '';
};

// 检测功能
const startDetection = async () => {
  if (!selectedFile.value && !isWebcamActive.value) {
    ElMessage.warning('请先上传文件或开启摄像头');
    return;
  }
  
  isDetecting.value = true;
  detectionResults.value = [];
  videoError.value = '';

  try {
    const startTime = Date.now();
    let response;

    if (selectedFile.value) {
      const formData = new FormData();
      formData.append('file', selectedFile.value);
      formData.append('conf', confidenceThreshold.value);
      
      if (selectedFile.value.type.startsWith('image/')) {
        // 车牌检测 API
        response = await axios.post(`${API_BASE_URL}/api/detect/image`, formData, {
          headers: { 'Content-Type': 'multipart/form-data' }
        });

        if (response.data.success && response.data.result_filename) {
          const fileName = response.data.result_filename;
          previewUrl.value = `${API_BASE_URL}/api/result/${fileName}?t=${Date.now()}`;
        }
      } else if (selectedFile.value.type.startsWith('video/')) {
        // 车牌检测视频 API
        response = await axios.post(`${API_BASE_URL}/api/detect/video`, formData, {
          headers: { 'Content-Type': 'multipart/form-data' },
          timeout: 300000
        });

        if (response.data.success) {
          let fileName = response.data.result_filename;
          if (!fileName && response.data.result_path) {
            fileName = response.data.result_path.split('/').pop();
          }
          if (!fileName && response.data.output_file) {
            fileName = response.data.output_file;
          }
          
          if (fileName) {
            try {
              ElMessage.info('正在加载标注后的视频...');
              const videoResponse = await axios.get(`${API_BASE_URL}/api/result/${fileName}`, {
                responseType: 'blob'
              });
              
              const blob = videoResponse.data;
              if (previewUrl.value && previewUrl.value.startsWith('blob:')) {
                URL.revokeObjectURL(previewUrl.value);
              }
              previewUrl.value = URL.createObjectURL(blob);
              isResultVideo.value = true;
              ElMessage({ message: '视频检测完成！', icon: '' });
            } catch (error) {
              console.error('获取标注视频失败:', error);
              ElMessage({ message: '加载标注视频失败: ' + error.message, type: 'error', icon: '' });
            }
          } else {
            ElMessage({ message: '检测失败：未获取到结果文件', type: 'error', icon: '' });
          }
        }
      }
    } else if (isWebcamActive.value) {
      const video = webcamVideoRef.value;
      const canvas = document.createElement('canvas');
      canvas.width = video.videoWidth;
      canvas.height = video.videoHeight;
      const ctx = canvas.getContext('2d');
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);

      canvas.toBlob(async (blob) => {
        const formData = new FormData();
        formData.append('file', new File([blob], 'webcam.jpg', { type: 'image/jpeg' }));
        formData.append('conf', confidenceThreshold.value);

        try {
          const imgResponse = await axios.post(`${API_BASE_URL}/api/detect/`, formData, {
            headers: { 'Content-Type': 'multipart/form-data' }
          });

          processingTime.value = Date.now() - startTime;
          isDetecting.value = false;

          if (imgResponse.data.success) {
            detectionResults.value = imgResponse.data.detections.map(det => ({
              class: det.class,
              confidence: det.confidence,
              bbox: [det.bbox.x1, det.bbox.y1, det.bbox.x2, det.bbox.y2],
              detectType: 'webcam'
            }));

            saveToHistory();
            
            if (imgResponse.data.result_filename) {
              const fileName = imgResponse.data.result_filename;
              previewUrl.value = `${API_BASE_URL}/api/result/${fileName}?t=${Date.now()}`;
            }

            ElMessage({ message: `检测完成！识别出${detectionResults.value.length}个车牌`, icon: '' });
          }
        } catch (e) {
          ElMessage({ message: '检测失败: ' + e.message, type: 'error', icon: '' });
          isDetecting.value = false;
        }
      }, 'image/jpeg');
      return;
    }

    processingTime.value = Date.now() - startTime;
    isDetecting.value = false;

    if (response?.data.success) {
      detectionResults.value = response.data.detections.map(det => ({
        class: det.class,
        confidence: det.confidence,
        bbox: [det.bbox.x1, det.bbox.y1, det.bbox.x2, det.bbox.y2]
      }));

      saveToHistory();
      ElMessage({ message: `检测完成！识别出${detectionResults.value.length}个车牌`, icon: '' });
    }
  } catch (error) {
    console.error('检测失败:', error);
    ElMessage({ message: '检测失败: ' + (error.response?.data?.error || error.message), type: 'error', icon: '' });
    isDetecting.value = false;
  }
};

// 根据置信度获取颜色
const getColorByConfidence = (confidence) => {
  if (confidence > 0.8) return '#4CAF50';
  if (confidence > 0.6) return '#FF9800';
  return '#F44336';
};

// 保存历史记录
const saveToHistory = () => {
  const historyItem = {
    id: Date.now(),
    fileName: selectedFile.value ? selectedFile.value.name : '摄像头检测',
    timestamp: Date.now(),
    results: [...detectionResults.value],
    processingTime: processingTime.value,
    detectType: selectedFile.value ? 'file' : 'webcam',
    detectionMode: 'license_plate' // 标记为车牌检测
  };

  const savedHistory = JSON.parse(localStorage.getItem('yolo_detection_history') || '[]');
  savedHistory.unshift(historyItem);

  if (savedHistory.length > 20) {
    savedHistory.length = 20;
  }

  localStorage.setItem('yolo_detection_history', JSON.stringify(savedHistory));
};

// 清空所有
const clearAll = () => {
  if (selectedFile.value) {
    if (previewUrl.value && previewUrl.value.startsWith('blob:')) {
      URL.revokeObjectURL(previewUrl.value);
    }
  }
  selectedFile.value = null;
  previewUrl.value = '';
  detectionResults.value = [];
  isVideoPlaying.value = false;
  videoProgress.value = 0;
  videoError.value = '';
  isWebcamActive.value = false;
  isResultVideo.value = false;
  
  if (webcamVideoRef.value && webcamVideoRef.value.srcObject) {
    webcamVideoRef.value.srcObject.getTracks().forEach(track => track.stop());
  }
  
  ElMessage({ message: '已清空所有内容', icon: '' });
};

// 导出结果（下载标注后的图片）
const exportResults = async () => {
  if (!previewUrl.value) {
    ElMessage({ message: '暂无可下载的检测结果图片', type: 'error', icon: '' });
    return;
  }

  try {
    const baseName = selectedFile.value
      ? selectedFile.value.name.replace(/\.[^.]+$/, '')
      : 'webcam';
    const downloadName = `${baseName}_plate_result_${Date.now()}.jpg`;

    let blobUrl = previewUrl.value;
    let needRevoke = false;
    if (!previewUrl.value.startsWith('blob:')) {
      const resp = await axios.get(previewUrl.value, { responseType: 'blob' });
      blobUrl = URL.createObjectURL(resp.data);
      needRevoke = true;
    }

    const a = document.createElement('a');
    a.href = blobUrl;
    a.download = downloadName;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    if (needRevoke) URL.revokeObjectURL(blobUrl);

    ElMessage({ message: '图片已下载', icon: '' });
  } catch (e) {
    ElMessage({ message: '下载失败: ' + e.message, type: 'error', icon: '' });
  }
};

// 监听从历史记录页面加载的事件
onMounted(() => {
  const handler = (e) => {
    const history = e.detail;
    detectionResults.value = history.results;
    processingTime.value = history.processingTime;
    ElMessage.info(`已加载历史记录: ${history.fileName}`);
  };
  window.addEventListener('load-history', handler);
  
  // 清理函数
  onUnmounted(() => {
    window.removeEventListener('load-history', handler);
  });
});

// 组件卸载时清理
onUnmounted(() => {
  if (webcamVideoRef.value && webcamVideoRef.value.srcObject) {
    webcamVideoRef.value.srcObject.getTracks().forEach(track => track.stop());
  }
  
  if (previewUrl.value && previewUrl.value.startsWith('blob:')) {
    URL.revokeObjectURL(previewUrl.value);
  }
});
</script>

<style scoped>
.home-container {
  display: grid;
  grid-template-columns: 3fr 1fr;
  gap: 25px;
}

/* 上传区域 */
.upload-section {
  background: white;
  border-radius: 12px;
  padding: 30px;
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.12);
}

.upload-zone {
  border: 3px dashed #d1d9e6;
  border-radius: 12px;
  padding: 60px 20px;
  text-align: center;
  cursor: pointer;
  transition: all 0.3s ease;
  background: #f8fafc;
  margin-bottom: 20px;
}

.upload-zone:hover {
  border-color: #667eea;
  background: #f0f4ff;
}

.upload-zone i {
  font-size: 48px;
  color: #667eea;
  margin-bottom: 15px;
}

.file-info {
  display: flex;
  align-items: center;
  gap: 10px;
  background: #f0f4ff;
  padding: 12px 20px;
  border-radius: 8px;
  margin: 20px 0;
}

/* 控制按钮组 */
.control-buttons {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
  margin: 20px 0;
}

.control-button {
  padding: 12px 24px;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  gap: 8px;
}

.btn-primary {
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
}

.btn-secondary {
  background: #f0f4ff;
  color: #667eea;
}

.control-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(102, 126, 234, 0.3);
}

.control-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* 预览区域 */
.preview-section {
  background: white;
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.12);
  margin-top: 20px;
}

.preview-header {
  margin-bottom: 15px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.preview-container {
  width: 100%;
  height: 400px;
  background: #f8fafc;
  border-radius: 8px;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
}

.preview-image,
.preview-video {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
}

/* 视频控制栏 */
.video-controls {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-top: 15px;
}

.play-button {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: #667eea;
  border: none;
  color: white;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 16px;
}

.progress-bar {
  flex: 1;
  height: 6px;
  background: #e0e0e0;
  border-radius: 3px;
  overflow: hidden;
  cursor: pointer;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #667eea, #764ba2);
  width: 0%;
  transition: width 0.1s linear;
}

.time-display {
  font-size: 12px;
  color: #666;
  min-width: 100px;
  text-align: right;
}

/* 结果展示区 */
.result-section {
  background: white;
  border-radius: 12px;
  padding: 25px;
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.12);
}

.result-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.result-header h3 {
  color: #333;
  font-size: 18px;
}

.stats-card {
  background: linear-gradient(135deg, #f0f4ff, #e3f2fd);
  padding: 15px;
  border-radius: 8px;
  margin-bottom: 20px;
}

.stat-item {
  display: flex;
  justify-content: space-between;
  margin: 8px 0;
}

.stat-label {
  color: #666;
}

.stat-value {
  color: #667eea;
  font-weight: 600;
}

.detection-list {
  max-height: 400px;
  overflow-y: auto;
}

.detection-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  background: #f8fafc;
  border-radius: 8px;
  margin-bottom: 8px;
}

.detection-color {
  width: 8px;
  height: 8px;
  border-radius: 50%;
}

/* 加载动画 */
.loading-spinner {
  width: 50px;
  height: 50px;
  border: 3px solid #f0f0f0;
  border-top: 3px solid #667eea;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

.loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(255, 255, 255, 0.8);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  z-index: 10;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .home-container {
    grid-template-columns: 1fr;
  }

  .control-buttons {
    flex-direction: column;
  }
}
</style>
