<template>
  <div id="app">
    <div class="app-container">
      <!-- 导航栏 -->
      <nav class="navbar">
        <div class="logo">
          <h1>🔍 YOLO26 智能物品识别系统</h1>
        </div>
        <div class="nav-links">
          <a href="#" :class="{ active: activeTab === 'home' }" @click="activeTab = 'home'">首页</a>
          <a href="#" :class="{ active: activeTab === 'history' }" @click="activeTab = 'history'">检测历史</a>
          <a href="#" :class="{ active: activeTab === 'about' }" @click="activeTab = 'about'">关于系统</a>
        </div>
      </nav>

      <!-- 主内容区 -->
      <div v-if="activeTab === 'home'" class="main-content">
        <!-- 左侧上传区域 -->
        <div class="upload-section">
          <div class="upload-zone" @click="triggerFileInput" @dragover.prevent @drop.prevent="handleDrop">
            <i>📁</i>
            <h3>上传图片或视频进行检测</h3>
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
              📥 导出结果
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
              <video v-else-if="selectedFile && selectedFile.type.includes('video')" 
                ref="previewVideoRef"
                :src="previewUrl" 
                class="preview-video"
                @timeupdate="updateVideoProgress"
                @loadedmetadata="updateVideoDuration"
                @error="handleVideoError"
                @canplay="handleVideoReady"
                controls>
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
              <span class="stat-label">检测物品数</span>
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

      <!-- 历史记录页面 -->
      <div v-if="activeTab === 'history'" class="upload-section">
        <h2 style="margin-bottom: 20px;">检测历史</h2>
        <div v-for="(history, index) in detectionHistory" :key="index" class="detection-item"
          @click="loadHistoryItem(history)" style="cursor: pointer;">
          <div style="flex: 1;">
            <div style="font-weight: 500;">{{ history.fileName }}</div>
            <div style="color: #666; font-size: 12px;">
              {{ formatDate(history.timestamp) }} | 检测到 {{ history.results.length }} 个物品
            </div>
          </div>
        </div>
      </div>

      <!-- 关于页面 -->
      <div v-if="activeTab === 'about'" class="upload-section">
        <h2 style="margin-bottom: 20px;">关于系统</h2>
        <div style="line-height: 1.6; color: #666;">
          <p>本系统基于 YOLOv11 深度学习模型，能够快速准确地识别图片和视频中的各种物品。</p>
          <p>主要特性：</p>
          <ul style="margin-left: 20px; margin-top: 10px;">
            <li>支持图片和视频上传检测</li>
            <li>支持摄像头实时检测</li>
            <li>高精度的物品识别</li>
            <li>现代化的用户界面</li>
            <li>检测结果可视化展示</li>
          </ul>
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
const activeTab = ref('home');
const selectedFile = ref(null);
const previewUrl = ref('');
const isDetecting = ref(false);
const isWebcamActive = ref(false);
const isVideoPlaying = ref(false);
const confidenceThreshold = ref(0.25);
const detectionResults = ref([]);
const processingTime = ref(0);
const detectionHistory = ref([]);
const videoCurrentTime = ref(0);
const videoDuration = ref(0);
const videoProgress = ref(0);
const videoError = ref('');

// DOM 引用
const fileInputRef = ref(null);
const previewVideoRef = ref(null);
const webcamVideoRef = ref(null);

// API 基础路径
const API_BASE_URL = 'http://127.0.0.1:5000';

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
  // 使用 URL.createObjectURL 创建本地预览URL
  // 这样可以避免跨域问题，并且支持所有视频格式
  if (previewUrl.value) {
    URL.revokeObjectURL(previewUrl.value);
  }
  previewUrl.value = URL.createObjectURL(file);
  detectionResults.value = [];
  videoError.value = '';
  isVideoPlaying.value = false;
  videoProgress.value = 0;
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

const formatDate = (timestamp) => {
  return new Date(timestamp).toLocaleString('zh-CN');
};

// 摄像头控制
const startWebcam = async () => {
  if (isWebcamActive.value) {
    // 关闭摄像头
    if (webcamVideoRef.value && webcamVideoRef.value.srcObject) {
      webcamVideoRef.value.srcObject.getTracks().forEach(track => track.stop());
    }
    isWebcamActive.value = false;
    ElMessage.success('摄像头已关闭');
  } else {
    // 开启摄像头
    try {
      const stream = await navigator.mediaDevices.getUserMedia({
        video: { width: 640, height: 480 }
      });
      if (webcamVideoRef.value) {
        webcamVideoRef.value.srcObject = stream;
        isWebcamActive.value = true;
        ElMessage.success('摄像头已开启，可进行检测');
      }
    } catch (error) {
      ElMessage.error('无法访问摄像头: ' + error.message);
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
  console.error('错误码:', video.error?.code);
  console.error('错误信息:', video.error?.message);
  console.error('video.src:', video.src);
  console.error('video.readyState:', video.readyState);
  console.error('video.networkState:', video.networkState);
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
        // 图片检测
        response = await axios.post(`${API_BASE_URL}/api/detect/image`, formData, {
          headers: { 'Content-Type': 'multipart/form-data' }
        });

        if (response.data.success) {
          // 图片检测后更新为标注后的图片
          if (response.data.result_filename) {
            const fileName = response.data.result_filename;
            // 使用后端API获取标注后的图片
            previewUrl.value = `${API_BASE_URL}/api/result/${fileName}?t=${Date.now()}`;
          }
        }
      } else if (selectedFile.value.type.startsWith('video/')) {
        // 视频检测
        ElMessage.info('开始处理视频，请稍候...');
        
        response = await axios.post(`${API_BASE_URL}/api/detect/video`, formData, {
          headers: { 'Content-Type': 'multipart/form-data' },
          timeout: 300000 // 5分钟超时
        });

        if (response.data.success) {
          console.log('视频检测API返回:', response.data);
          console.log('result_filename:', response.data.result_filename);
          console.log('result_path:', response.data.result_path);
          console.log('detections count:', response.data.detections?.length);
          
          // 获取标注后的视频数据 - 尝试多种可能的字段
          let fileName = response.data.result_filename;
          
          // 如果没有 result_filename,尝试从 result_path 提取
          if (!fileName && response.data.result_path) {
            fileName = response.data.result_path.split('/').pop();
          }
          
          // 如果还是没有,尝试其他可能的字段
          if (!fileName && response.data.output_file) {
            fileName = response.data.output_file;
          }
          
          if (fileName) {
            console.log('准备获取视频,文件名:', fileName);
            try {
              ElMessage.info('正在加载标注后的视频...');
              
              // 通过axios获取视频数据
              const videoResponse = await axios.get(`${API_BASE_URL}/api/result/${fileName}`, {
                responseType: 'blob'
              });
              
              console.log('视频数据获取成功，大小:', videoResponse.data.size);
              
              // 创建下载链接
              const blob = videoResponse.data;
              const downloadUrl = URL.createObjectURL(blob);
              const a = document.createElement('a');
              a.href = downloadUrl;
              a.download = fileName;
              document.body.appendChild(a);
              a.click();
              document.body.removeChild(a);
              URL.revokeObjectURL(downloadUrl);
              
              console.log('视频已开始下载:', fileName);
              ElMessage.success('视频检测完成，已开始下载！');
              
            } catch (error) {
              console.error('获取标注视频失败:', error);
              console.error('错误详情:', error.response?.data);
              ElMessage.error('加载标注视频失败: ' + error.message);
            }
          } else {
            console.error('未找到结果文件名，返回数据:', response.data);
            ElMessage.error('检测失败：未获取到结果文件');
          }
        }
      }
    } else if (isWebcamActive.value) {
      // 摄像头检测
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
          const imgResponse = await axios.post(`${API_BASE_URL}/api/detect/image`, formData, {
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

            ElMessage.success(`检测完成！识别出${detectionResults.value.length}个物品`);
          }
        } catch (e) {
          ElMessage.error('检测失败: ' + e.message);
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
      ElMessage.success(`检测完成！识别出${detectionResults.value.length}个物品`);
    }
  } catch (error) {
    console.error('检测失败:', error);
    ElMessage.error('检测失败: ' + (error.response?.data?.error || error.message));
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
    detectType: selectedFile.value ? 'file' : 'webcam'
  };

  detectionHistory.value.unshift(historyItem);

  if (detectionHistory.value.length > 20) {
    detectionHistory.value = detectionHistory.value.slice(0, 20);
  }

  localStorage.setItem('yolo_detection_history', JSON.stringify(detectionHistory.value));
};

// 加载历史记录
const loadHistoryItem = (history) => {
  detectionResults.value = history.results;
  processingTime.value = history.processingTime;
  activeTab.value = 'home';
  ElMessage.info(`已加载历史记录: ${history.fileName}`);
};

// 清空所有
const clearAll = () => {
  if (selectedFile.value) {
    // 释放blob URL
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
  
  // 关闭摄像头
  if (webcamVideoRef.value && webcamVideoRef.value.srcObject) {
    webcamVideoRef.value.srcObject.getTracks().forEach(track => track.stop());
  }
  
  ElMessage.success('已清空所有内容');
};

// 导出结果
const exportResults = () => {
  const data = {
    detectionResults: detectionResults.value,
    timestamp: new Date().toISOString(),
    fileName: selectedFile.value ? selectedFile.value.name : '摄像头检测',
    averageConfidence: averageConfidence.value,
    processingTime: processingTime.value
  };

  const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `yolo_detection_${Date.now()}.json`;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);

  ElMessage.success('结果已导出为JSON文件');
};

// 初始化加载历史记录
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

// 组件卸载时清理
onUnmounted(() => {
  // 关闭摄像头
  if (webcamVideoRef.value && webcamVideoRef.value.srcObject) {
    webcamVideoRef.value.srcObject.getTracks().forEach(track => track.stop());
  }
  
  // 释放blob URL
  if (previewUrl.value && previewUrl.value.startsWith('blob:')) {
    URL.revokeObjectURL(previewUrl.value);
  }
});
</script>

<style scoped>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Segoe UI', 'Microsoft YaHei', sans-serif;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  min-height: 100vh;
}

.app-container {
  max-width: 1400px;
  margin: 0 auto;
  padding: 20px;
}

/* 导航栏 */
.navbar {
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  border-radius: 12px;
  padding: 15px 25px;
  margin-bottom: 25px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.logo h1 {
  background: linear-gradient(135deg, #667eea, #764ba2);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  font-size: 24px;
  font-weight: 700;
}

.nav-links {
  display: flex;
  gap: 25px;
}

.nav-links a {
  text-decoration: none;
  color: #666;
  font-weight: 500;
  padding: 8px 16px;
  border-radius: 6px;
  transition: all 0.3s ease;
}

.nav-links a:hover,
.nav-links a.active {
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
}

/* 主内容区 */
.main-content {
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

.detection-info {
  flex: 1;
}

.detection-name {
  font-weight: 500;
  color: #333;
}

.detection-confidence {
  font-size: 12px;
  color: #666;
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
  .main-content {
    grid-template-columns: 1fr;
  }

  .nav-links {
    gap: 10px;
  }

  .control-buttons {
    flex-direction: column;
  }
}
</style>
