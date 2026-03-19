<template>
  <div class="about-container">
    <div class="about-content">
      <div class="about-header">
        <div class="system-icon">🔍</div>
        <h1>YOLO26 智能物品识别系统</h1>
        <p class="version">v1.0.0</p>
      </div>
      <div class="feature-section">
        <h2>系统简介</h2>
        <p>本系统基于 YOLOv26 深度学习模型，能够快速准确地识别图片和视频中的各种物品。采用先进的目标检测算法，提供实时、高精度的物体识别服务。</p>
      </div>

      <div class="feature-section">
        <h2>主要特性</h2>
        <div class="features-grid">
          <div class="feature-card">
            <div class="feature-icon">📸</div>
            <h3>图片检测</h3>
            <p>支持 JPG、PNG 等常见图片格式的智能识别</p>
          </div>
          <div class="feature-card">
            <div class="feature-icon">🎬</div>
            <h3>视频检测</h3>
            <p>支持 MP4、AVI 等视频格式的实时分析</p>
          </div>
          <div class="feature-card">
            <div class="feature-icon">📹</div>
            <h3>摄像头检测</h3>
            <p>支持摄像头实时画面检测与分析</p>
          </div>
          <div class="feature-card">
            <div class="feature-icon">⚡</div>
            <h3>高效快速</h3>
            <p>优化的算法确保快速响应和实时处理</p>
          </div>
          <div class="feature-card">
            <div class="feature-icon">📊</div>
            <h3>结果可视化</h3>
            <p>直观展示检测结果、置信度和统计信息</p>
          </div>
          <div class="feature-card">
            <div class="feature-icon">💾</div>
            <h3>历史记录</h3>
            <p>自动保存检测历史，支持快速回顾</p>
          </div>
        </div>
      </div>

      <div class="feature-section">
        <h2>技术栈</h2>
        <div class="tech-tags">
          <span class="tech-tag">Vue 3</span>
          <span class="tech-tag">Element Plus</span>
          <span class="tech-tag">Axios</span>
          <span class="tech-tag">YOLOv26</span>
          <span class="tech-tag">Python Flask</span>
        </div>
      </div>

      <div class="feature-section">
        <h2>使用说明</h2>
        <ol class="instructions">
          <li>上传图片或视频文件，或直接开启摄像头</li>
          <li>点击"开始检测"按钮进行识别</li>
          <li>查看检测结果和统计信息</li>
          <li>支持导出检测结果为 JSON 格式</li>
          <li>在"检测历史"页面查看之前的检测记录</li>
        </ol>
      </div>

      <div class="feature-section">
        <h2>注意事项</h2>
        <ul class="notes">
          <li>建议上传清晰度较高的图片以获得更准确的检测效果</li>
          <li>视频文件大小建议不超过 500MB</li>
          <li>首次使用摄像头需要浏览器授权</li>
          <li>检测结果仅供参考，具体应用请结合实际场景</li>
        </ul>
      </div>
    </div>

    <div class="about-footer">
      <p>&copy; 2026 YOLO26 智能物品识别系统. All rights reserved.</p>
    </div>
  </div>
</template>

<script setup>
// 静态页面，无需脚本逻辑
</script>

<style scoped>
.about-container {
  padding: 0;
  max-width: 1400px;
  margin: 0 auto;
}

.about-header {
  text-align: center;
  padding: 40px 20px;
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.1), rgba(118, 75, 162, 0.1));
  border-radius: 16px;
}

.system-icon {
  font-size: 64px;
  margin-bottom: 20px;
}

.about-header h1 {
  font-size: 36px;
  color: #333;
  margin-bottom: 10px;
  font-weight: 700;
}

.version {
  color: #667eea;
  font-weight: 600;
  font-size: 16px;
}

.about-content {
  display: flex;
  flex-direction: column;
  gap: 40px;
  background: #ffffff;
  border-radius: 16px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.06);
  padding: 20px;
}

.feature-section h2 {
  font-size: 24px;
  color: #333;
  margin-bottom: 20px;
  font-weight: 700;
  border-left: 4px solid #667eea;
  padding-left: 15px;
}

.feature-section > p {
  color: #666;
  line-height: 1.8;
  font-size: 15px;
}

.features-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 20px;
}

.feature-card {
  background: #f8f9fa;
  border-radius: 12px;
  padding: 25px;
  text-align: center;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
  transition: all 0.3s ease;
}

.feature-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.12);
}

.feature-icon {
  font-size: 40px;
  margin-bottom: 15px;
}

.feature-card h3 {
  font-size: 18px;
  color: #333;
  margin-bottom: 10px;
  font-weight: 600;
}

.feature-card p {
  color: #666;
  font-size: 14px;
  line-height: 1.5;
}

.tech-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
}

.tech-tag {
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
  padding: 8px 20px;
  border-radius: 20px;
  font-weight: 500;
  font-size: 14px;
}

.instructions, .notes {
  color: #666;
  line-height: 2;
  font-size: 15px;
}

.instructions li, .notes li {
  margin: 10px 0;
}

.instructions {
  padding-left: 20px;
}

.notes {
  list-style: none;
}

.notes li::before {
  content: "•";
  color: #667eea;
  font-weight: bold;
  display: inline-block;
  width: 1em;
  margin-left: -1em;
}

.about-footer {
  margin-top: 50px;
  text-align: center;
  padding-top: 30px;
  border-top: 1px solid #e0e0e0;
  color: #999;
  font-size: 14px;
}
</style>
