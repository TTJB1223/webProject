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
const messageList = ref([]);
const inputMsg = ref('');
const status = ref('connecting');
const statusText = ref('正在连接 WebSocket...');

// --------------------------
// 2. WebSocket 核心变量
// --------------------------
let ws = null;
let reconnectTimer = null;
// 关键修改：将 ws:// 改为 wss://（需确保服务端配置了WSS）
// 注意：这里需要替换成你自己的域名（不能用IP），因为SSL证书绑定的是域名
const WS_URL = 'wss://你的域名:8800'; // 例如：wss://demo.xxx.com:8800

// --------------------------
// 3. 工具函数：格式化时间
// --------------------------
const formatTime = () => {
  const now = new Date();
  return `${now.getHours()}:${String(now.getMinutes()).padStart(2, '0')}`;
};

// --------------------------
// 4. WebSocket 核心逻辑（增加容错）
// --------------------------
const initWebSocket = () => {
  if (reconnectTimer) clearTimeout(reconnectTimer);

  // 增加浏览器兼容性检查
  if (!window.WebSocket) {
    status.value = 'error';
    statusText.value = '❌ 你的浏览器不支持WebSocket';
    return;
  }

  try {
    ws = new WebSocket(WS_URL);

    ws.onopen = () => {
      status.value = 'open';
      statusText.value = '✅ 连接成功，可以发送消息';
      console.log('WebSocket 连接成功');
    };

    ws.onmessage = (event) => {
      messageList.value.push({
        content: event.data,
        isSent: false,
        time: formatTime()
      });
      console.log('收到消息：', event.data);
    };

    ws.onerror = (error) => {
      status.value = 'error';
      statusText.value = '❌ 连接出错，即将重连';
      console.error('WebSocket 错误：', error);
    };

    ws.onclose = (event) => {
      status.value = 'closed';
      if (event.code === 1000) {
        statusText.value = '🔌 已手动关闭连接';
      } else {
        statusText.value = `🔌 连接断开（状态码：${event.code}），3秒后重连...`;
        reconnectTimer = setTimeout(initWebSocket, 3000);
      }
      console.log('WebSocket 关闭，状态码：', event.code);
    };
  } catch (e) {
    status.value = 'error';
    statusText.value = '❌ 初始化WebSocket失败';
    console.error('初始化失败：', e);
    reconnectTimer = setTimeout(initWebSocket, 3000);
  }
};

const sendMessage = () => {
  const msg = inputMsg.value.trim();
  if (!msg) return;

  if (ws && ws.readyState === WebSocket.OPEN) {
    ws.send(msg);
    messageList.value.push({
      content: msg,
      isSent: true,
      time: formatTime()
    });
    inputMsg.value = '';
    console.log('发送消息：', msg);
  } else {
    statusText.value = '⚠️ 连接未就绪，无法发送';
    alert('WebSocket 未连接，请等待重连完成！');
  }
};

const closeWebSocket = () => {
  if (ws) {
    ws.close(1000, '用户手动关闭');
    if (reconnectTimer) clearTimeout(reconnectTimer);
  }
};

// --------------------------
// 5. Vue 生命周期管理
// --------------------------
onMounted(() => {
  initWebSocket();
});

onUnmounted(() => {
  closeWebSocket();
});
</script>

<style scoped>
/* 补充样式，让页面更易读（可选） */
.status {
  text-align: center;
  padding: 8px;
  border-radius: 4px;
  margin-bottom: 10px;
}
.connecting { background: #fef3c7; color: #d97706; }
.open { background: #dcfce7; color: #16a34a; }
.error { background: #fee2e2; color: #dc2626; }
.closed { background: #e5e7eb; color: #4b5563; }

.message-list {
  height: 300px;
  border: 1px solid #e5e7eb;
  border-radius: 4px;
  padding: 10px;
  overflow-y: auto;
  margin-bottom: 10px;
}
.message-item {
  margin-bottom: 8px;
  max-width: 70%;
}
.sent {
  margin-left: auto;
}
.received {
  margin-right: auto;
}
.content {
  padding: 6px 10px;
  border-radius: 6px;
  word-break: break-all;
}
.sent .content {
  background: #3b82f6;
  color: white;
}
.received .content {
  background: #f3f4f6;
  color: #111827;
}

.input-area {
  display: flex;
  gap: 8px;
}
input {
  flex: 1;
  padding: 8px;
  border: 1px solid #e5e7eb;
  border-radius: 4px;
}
button {
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}
.send-btn {
  background: #3b82f6;
  color: white;
}
.close-btn {
  background: #ef4444;
  color: white;
}
</style>