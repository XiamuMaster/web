<template>
  <div id="app">
    <div class="star-bg">
      <div class="app-container">
        <!-- 导航栏 -->
        <nav class="navbar">
          <div class="logo">
            <h1>🔍 YOLO26 智能物品识别系统</h1>
          </div>
          <div class="nav-links">
            <router-link to="/" :class="{ active: $route.name === 'home' }">首页</router-link>
            <router-link to="/license-plate" :class="{ active: $route.name === 'license-plate' }">车牌检测</router-link>
            <router-link to="/history" :class="{ active: $route.name === 'history' }">检测历史</router-link>
            <router-link to="/about" :class="{ active: $route.name === 'about' }">关于系统</router-link>
          </div>
        </nav>

        <!-- 主内容区 -->
        <router-view />
      </div>
    </div>

  </div>
</template>

<script setup>
import { onMounted } from 'vue';

onMounted(() => {
  // 确保历史记录数据初始化
  if (!localStorage.getItem('yolo_detection_history')) {
    localStorage.setItem('yolo_detection_history', '[]');
  }
});
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Segoe UI', 'Microsoft YaHei', sans-serif;
  min-height: 100vh;
  margin: 0; /* 清除默认边距，避免滚动条异常 */
  background: #000; /* 把黑色背景移到body，避免star-bg的高度限制 */
}

/* 星空背景容器（核心修改：移除height和overflow限制） */
.star-bg {
  position: relative;
  width: 100%;
  min-height: 100vh; /* 改用min-height，适配内容高度 */
  overflow: visible; /* 允许内容溢出，显示拖拽框 */
}

/* 星星样式（新增z-index，避免遮挡交互元素） */
.star-bg::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: radial-gradient(4px 4px at 20px 30px, #ffccd5, transparent),
    radial-gradient(4px 4px at 40px 70px, #ff99cc, transparent),
    radial-gradient(4px 4px at 50px 160px, #ff8fab, transparent),
    radial-gradient(4px 4px at 90px 40px, #e88ea0, transparent),
    radial-gradient(4px 4px at 130px 80px, #ffccd5, transparent),
    radial-gradient(4px 4px at 160px 120px, #ff99cc, transparent);
  background-repeat: repeat;
  background-size: 200px 200px;
  opacity: 0;
  animation: starTwinkle 4s linear infinite;
  z-index: -1;
  pointer-events: none;
}

/* 第二层星星：把1px改成3px（星星变大） */
.star-bg::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: radial-gradient(3px 3px at 30px 140px, #ffccd5, transparent),
    radial-gradient(3px 3px at 100px 190px, #ff99cc, transparent),
    radial-gradient(3px 3px at 150px 50px, #ff8fab, transparent);
  background-repeat: repeat;
  background-size: 250px 250px;
  opacity: 0;
  animation: starTwinkle 5s linear infinite 1s;
  z-index: -1;
  pointer-events: none;
}

/* 星星闪烁动画 */
@keyframes starTwinkle {
  0%, 100% {
    opacity: 0;
  }
  50% {
    opacity: 1;
  }
}
#app {
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
.nav-links a.active,
.nav-links a.router-link-active {
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .navbar {
    flex-direction: column;
    gap: 15px;
  }

  .nav-links {
    gap: 10px;
  }

  .app-container {
    padding: 10px;
  }
}
</style>
