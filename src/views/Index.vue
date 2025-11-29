
<template>
  <div class="chat-panel">
    <div class="chat-title">大新的神秘小ai</div>
    <div class="message-panel" ref="messagePanel">
      <div class="message-list">
        <div
          v-for="(item, index) in messages"
          :key="index"
          :class="['message-item', item.sender]"
          :id="'item' + index"
        >
          <template v-if="item.sender === 'user'">
            <div class="user-icon">我</div>
          </template>
          <template v-else>
            <div class="user-icon">AI</div>
          </template>
          <div class="message-content">
            <span v-if="item.sender === 'bot'">
              <span
                v-for="(part, idx) in parseMessageContent(item.content)"
                :key="idx"
              >
                <span v-if="part.isThink && part.text.trim()" class="think-content">
                  <span v-html="renderMarkdown(part.text)"></span>
                </span>
                <span v-else>
                  <span v-html="renderMarkdown(part.text)"></span>
                </span>
                <br v-if="part.isThink && part.text.trim()">
              </span>
              </span>
              <span v-else class="content-inner">{{ item.content }}</span>
          </div>
        </div>
        <div v-if="loading" class="loading-indicator">
          <div class="loading-dots">
            <span></span>
            <span></span>
            <span></span>
          </div>
        </div>
      </div>
    </div>
    <div class="send-panel">
      <el-form-item label="" prop="content">
        <el-input
          type="textarea"
          :rows="3"
          clearable
          placeholder="请输入你想问的问题"
          v-model="inputmessage"
          @keyup="keySend"
        ></el-input>
      </el-form-item>
      <el-form-item label="" prop="" class="send-btn">
        <el-button type="primary" @click="sendMessage" :loading="loading">
          发送(ctrl+enter)
        </el-button>
        <el-button type="primary" @click="deleteMessage">
          删除所有会话
        </el-button>
      </el-form-item>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick, onMounted, watch } from 'vue'
import { marked } from 'marked'
import axios from 'axios'

const messagePanel = ref(null)
const messages = ref([])
const inputmessage = ref('')
const loading = ref(false)

const renderMarkdown = (content) => {
  return marked(content)
}

const parseMessageContent = (content) => {
  const parts = []
  let remaining = content

  while (remaining.includes("<think>") && remaining.includes("</think>")) {
    const thinkStart = remaining.indexOf("<think>")
    const thinkEnd = remaining.indexOf("</think>") + "</think>".length

    if (thinkStart > 0) {
      parts.push({ text: remaining.slice(0, thinkStart), isThink: false })
    }

    const thinkContent = remaining.slice(thinkStart + "<think>".length, thinkEnd - "</think>".length)
    parts.push({ text: thinkContent, isThink: true })
    
    remaining = remaining.slice(thinkEnd)
  }
  
  if (remaining) {
    parts.push({ text: remaining, isThink: false })
  }
  return parts
}

const scrollToBottom = () => {
  nextTick(() => {
    if (messagePanel.value) {
      messagePanel.value.scrollTop = messagePanel.value.scrollHeight
  }})

}

// 监听messages变化，自动滚动到底部
watch(messages, () => {
  scrollToBottom()
}, { deep: true, flush: 'post' })

const loadHistory = async () => {
  try {
    const response = await axios.get('/api/messages')
    messages.value = response.data.sort((a, b) => new Date(a.create_at) - new Date(b.create_at))
  } catch (error) {
    console.error('获取历史失败:', error)
  }
}

const keySend = (event) => {
  if (!(event.ctrlKey && event.key === 'Enter')) {
    return
  }
  sendMessage()
}

const deleteMessage = async () => {
  axios.delete('/api/delete');
  messages.value = [];
  alert('删除成功');
  await loadHistory();
}

const sendMessage = async () => {
  if (!inputmessage.value?.trim() || loading.value) return

  const userText = inputmessage.value.trim()
  
  const userMsg = {
    sender: "user",
    content: userText,
    create_at: new Date().toISOString()
  }

  messages.value.push(userMsg)
  inputmessage.value = ""

  const botMsg = {
    sender: "bot",
    content: "",
    create_at: new Date().toISOString()
  }

  messages.value.push(botMsg)
  loading.value = true

  scrollToBottom()

  try {
    const eventSource = new EventSource(`/api/stream?prompt=${encodeURIComponent(userText)}`)
    
    eventSource.onmessage = (event) => {
      if (event.data) {
        botMsg.content += event.data
        messages.value = [...messages.value]
        scrollToBottom()
      }
    }

    eventSource.onerror = (error) => {
      console.error('EventSource错误:', error)
      eventSource.close()
      loading.value = false
    }

    setTimeout(() => {
      if (loading.value) {
        eventSource.close()
        loading.value = false
      }
    }, 30000)

  } catch (error) {
    console.error('发送消息失败:', error)
    loading.value = false
  }
}

onMounted(() => {
  loadHistory()
  scrollToBottom()
})
</script>

<style scoped>
.chat-panel {
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: #f5f7fa;
}

.chat-title {
  padding: 1rem 2rem;
  background: white;
  border-bottom: 1px solid #e4e7ed;
  font-size: 1.25rem;
  font-weight: 600;
  color: #303133;
}

.message-panel {
  flex: 1;
  overflow-y: auto;
  padding: 1rem;
}

.message-list {
  max-width: 800px;
  margin: 0 auto;
}

.message-item {
  display: flex;
  margin-bottom: 1rem;
  align-items: flex-start;
}

.message-item.user {
  flex-direction: row-reverse;
}

.user-icon {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  margin: 0 0.5rem;
  flex-shrink: 0;
}

.message-item.user .user-icon {
  background: linear-gradient(135deg, #409eff, #67c23a);
  color: white;
}

.message-item.bot .user-icon {
  background: linear-gradient(135deg, #e6a23c, #f56c6c);
  color: white;
}

.message-content {
  max-width: 70%;
  background: white;
  padding: 0.75rem 1rem;
  border-radius: 0.75rem;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
  word-wrap: break-word;
}

.message-item.user .message-content {
  background: linear-gradient(135deg, #409eff, #67c23a);
  color: white;
}

.think-content {
  background: #f0f9ff;
  padding: 0.5rem;
  border-radius: 0.375rem;
  display: block;
  margin-bottom: 0.5rem;
  border-left: 3px solid #409eff;
}

.content-inner {
  line-height: 1.5;
}

.loading-indicator {
  display: flex;
  justify-content: center;
  padding: 1rem;
}

.loading-dots {
  display: flex;
  gap: 0.25rem;
}

.loading-dots span {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #409eff;
  animation: dot-pulse 1.4s ease-in-out infinite both;
}

.loading-dots span:nth-child(2) {
  animation-delay: 0.2s;
}

.loading-dots span:nth-child(3) {
  animation-delay: 0.4s;
}

@keyframes dot-pulse {
  0%, 80%, 100% {
    transform: scale(0.8);
    opacity: 0.5;
  }
  40% {
    transform: scale(1);
    opacity: 1;
  }
}

.send-panel {
  background: white;
  padding: 1rem 2rem;
  border-top: 1px solid #e4e7ed;
}

.send-btn {
  margin-top: 0.5rem;
}
</style>