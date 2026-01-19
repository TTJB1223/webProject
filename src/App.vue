<template>
  <div>
    <h2 style="text-align: center; margin-bottom: 10px;">Vue3 WebSocket 极简版（无类封装）</h2>
    
    <!-- 连接状态展示 -->
    <div class="status" :class="status">{{ statusText }}</div>

    <!-- 消息列表 -->
    <div class="message-list">
      <div 
        v-for="(msg, index) in messageList" 
        :key="index" 
        class="message-item"
        :class="{ sent: msg.isSent, received: !msg.isSent }"
      >
        <div class="content">{{ msg.content }}</div>
        <small style="font-size: 10px; color: #9ca3af;">{{ msg.time }}</small>
      </div>
    </div>

    <!-- 输入和操作按钮 -->
    <div class="input-area">
      <input
        v-model="inputMsg"
        type="text"
        placeholder="输入消息按回车/点击发送"
        @keyup.enter="sendMessage"
      />
      <button class="send-btn" @click="sendMessage">发送</button>
      <button class="close-btn" @click="closeWebSocket">关闭连接</button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

// --------------------------
// 1. 响应式数据（页面展示用）
// --------------------------
// 消息列表
const messageList = ref([]);
// 输入框内容
const inputMsg = ref('');
// 连接状态（connecting/open/error/closed）
const status = ref('connecting');
// 状态提示文字
const statusText = ref('正在连接 WebSocket...');

// --------------------------
// 2. WebSocket 核心变量（非响应式，仅逻辑用）
// --------------------------
// WebSocket 实例
let ws = null;
// 重连定时器（避免重复重连）
let reconnectTimer = null;
// WebSocket 服务地址（公共回显服务，发什么返回什么）
const WS_URL = 'ws://124.222.6.60:8800';

// --------------------------
// 3. 工具函数：格式化时间
// --------------------------
const formatTime = () => {
  const now = new Date();
  return `${now.getHours()}:${String(now.getMinutes()).padStart(2, '0')}`;
};

// --------------------------
// 4. WebSocket 核心逻辑
// --------------------------
// 建立连接（核心函数）
const initWebSocket = () => {
  // 先清空之前的定时器（避免重复重连）
  if (reconnectTimer) clearTimeout(reconnectTimer);

  // 创建 WebSocket 实例（无类封装，直接创建）
  ws = new WebSocket(WS_URL);

  // 监听：连接成功
  ws.onopen = () => {
    status.value = 'open';
    statusText.value = '✅ 连接成功，可以发送消息';
    console.log('WebSocket 连接成功');
  };

  // 监听：接收消息
  ws.onmessage = (event) => {
    // 把服务端返回的消息添加到列表
    messageList.value.push({
      content: event.data,
      isSent: false, // 接收的消息
      time: formatTime()
    });
    console.log('收到消息：', event.data);
  };

  // 监听：连接错误
  ws.onerror = (error) => {
    status.value = 'error';
    statusText.value = '❌ 连接出错，即将重连';
    console.error('WebSocket 错误：', error);
  };

  // 监听：连接关闭
  ws.onclose = (event) => {
    status.value = 'closed';
    // 区分手动关闭和异常关闭
    if (event.code === 1000) {
      statusText.value = '🔌 已手动关闭连接';
    } else {
      statusText.value = `🔌 连接断开（状态码：${event.code}），3秒后重连...`;
      // 异常关闭则自动重连（3秒延迟）
      reconnectTimer = setTimeout(initWebSocket, 3000);
    }
    console.log('WebSocket 关闭，状态码：', event.code);
  };
};

// 发送消息
const sendMessage = () => {
  // 校验输入
  const msg = inputMsg.value.trim();
  if (!msg) return;

  // 检查连接状态：只有 OPEN 状态才能发送
  if (ws && ws.readyState === WebSocket.OPEN) {
    // 发送消息到服务端
    ws.send(msg);
    // 把发送的消息添加到本地列表
    messageList.value.push({
      content: msg,
      isSent: true, // 发送的消息
      time: formatTime()
    });
    // 清空输入框
    inputMsg.value = '';
    console.log('发送消息：', msg);
  } else {
    statusText.value = '⚠️ 连接未就绪，无法发送';
    alert('WebSocket 未连接，请等待重连完成！');
  }
};

// 手动关闭连接
const closeWebSocket = () => {
  if (ws) {
    // 1000 是正常关闭的状态码，用于区分手动/异常关闭
    ws.close(1000, '用户手动关闭');
    // 清空重连定时器（避免关闭后还重连）
    if (reconnectTimer) clearTimeout(reconnectTimer);
  }
};

// --------------------------
// 5. Vue 生命周期管理
// --------------------------
// 组件挂载时：初始化 WebSocket
onMounted(() => {
  initWebSocket();
});

// 组件卸载时：关闭连接 + 清空定时器（避免内存泄漏）
onUnmounted(() => {
  closeWebSocket();
});
</script>