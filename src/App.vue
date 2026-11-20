<template>
  <div class="chat-container">
    <h2>AI 智能客服 (Vue3 + SSE)</h2>

    <div class="message-list" ref="messageListRef">
      <div
        v-for="(msg, index) in messages"
        :key="index"
        :class="['message', msg.role]"
      >
        <div class="avatar">{{ msg.role === "user" ? "我" : "AI" }}</div>
        <div class="content">
          <div v-if="msg.role === 'user'">{{ msg.content }}</div>
          <div v-else>
            <!-- 特殊工单卡片渲染 -->
            <div v-if="msg.content.includes('[工单卡片]')" class="ticket-card">
              <div class="ticket-header">
                <span>工单号：GD-20240501-001</span>
                <span class="status processing">处理中</span>
              </div>
              <div class="ticket-body">
                <p>业务类型：宽带报障</p>
                <p>当前进度：维修人员已接单，预计 2 小时内上门。</p>
              </div>
            </div>
            <!-- 正常 Markdown 渲染 -->
            <div v-else v-html="renderMarkdown(msg.content)"></div>
          </div>
        </div>
      </div>
    </div>

    <div class="input-area">
      <input
        v-model="inputText"
        @keyup.enter="sendMessage"
        placeholder="输入问题... (试试问：帮我查下工单进度)"
        :disabled="isGenerating"
      />
      <button v-if="isGenerating" @click="stopGeneration" class="stop-btn">
        停止生成
      </button>
      <button v-else @click="sendMessage" class="send-btn">发送</button>

      {{ messages }}
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, nextTick } from "vue";
import { marked } from "marked";

const inputText = ref("");
const messages = ref<{ role: "user" | "assistant"; content: string }[]>([]);
const isGenerating = ref(false);
const messageListRef = ref<HTMLElement | null>(null);

// 剪刀：用于中断请求
let abortController: AbortController | null = null;

// 【注意这里】把刚才复制的 API Key 填进来
const API_KEY = "sk-e1baae07f7584c288c96a60ba6401f52";
const API_URL = "https://api.deepseek.com/chat/completions";

const renderMarkdown = (text: string) => {
  console.log(marked.parse(text));
  return marked.parse(text);
};

const scrollToBottom = async () => {
  await nextTick();
  if (messageListRef.value) {
    messageListRef.value.scrollTop = messageListRef.value.scrollHeight;
  }
};

const sendMessage = async () => {
  if (!inputText.value.trim() || isGenerating.value) return;

  const userText = inputText.value;
  inputText.value = "";

  messages.value.push({ role: "user", content: userText });
  messages.value.push({ role: "assistant", content: "" });
  const aiMessageIndex = messages.value.length - 1;

  isGenerating.value = true;
  scrollToBottom();

  // 初始化剪刀
  abortController = new AbortController();

  try {
    // 【暗号拦截】为了演示卡片，如果包含工单二字，直接返回特殊结构
    if (userText.includes("工单")) {
      await new Promise((resolve) => setTimeout(resolve, 800)); // 模拟网络延迟
      messages.value[aiMessageIndex].content =
        "正在为您查询工单信息...\n\n[工单卡片]";
      isGenerating.value = false;
      scrollToBottom();
      return;
    }

    // 发起 SSE 请求
    const response = await fetch(API_URL, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${API_KEY}`,
      },
      body: JSON.stringify({
        model: "deepseek-chat",
        messages: [{ role: "user", content: userText }],
        stream: true, // 开启流式
      }),
      signal: abortController.signal, // 绑定剪刀
    });

    if (!response.ok) throw new Error("网络请求失败");

    const reader = response.body?.getReader();
    const decoder = new TextDecoder("utf-8");

        if (reader) {
      let buffer = '' // 核心点：新增缓冲区，用来拼接半包数据
      
      while (true) {
        const { done, value } = await reader.read()
        if (done) break

        // 1. 将新收到的二进制数据解码，并拼接到缓冲区
        buffer += decoder.decode(value, { stream: true })
        console.log(buffer,'buffer')
        
        // 2. 按 SSE 的标准分隔符 '\n\n' 切割
        const parts = buffer.split('\n\n')
         console.log(parts,'parts')
        
        // 3. 核心点：最后一部分可能是不完整的，把它留在 buffer 里，等下一次读取
        buffer = parts.pop() || '' 

        for (const part of parts) {
          const lines = part.split('\n')
          for (const line of lines) {
            if (line.startsWith('data: ') && line !== 'data: [DONE]') {
              try {
                const data = JSON.parse(line.slice(6))
                const content = data.choices[0]?.delta?.content || ''
                messages.value[aiMessageIndex].content += content
                scrollToBottom()
              } catch (e) {
                console.error('解析单行数据失败:', line, e)
              }
            }
          }
        }
      }
    }
  } catch (error: any) {
    if (error.name === "AbortError") {
      messages.value[aiMessageIndex].content += "\n\n[已停止生成]";
    } else {
      messages.value[aiMessageIndex].content = "抱歉，网络似乎出了点问题。";
    }
  } finally {
    isGenerating.value = false;
    abortController = null;
    scrollToBottom();
  }
};

const stopGeneration = () => {
  if (abortController) {
    abortController.abort(); // 咔嚓，剪断水管
  }
};
</script>

<style scoped>
.chat-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  height: 90vh;
  display: flex;
  flex-direction: column;
  background: #f5f5f5;
  border-radius: 10px;
  font-family: sans-serif;
}
h2 {
  text-align: center;
  color: #333;
}
.message-list {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
  background: white;
  border-radius: 8px;
  margin-bottom: 20px;
  box-shadow: inset 0 0 5px rgba(0, 0, 0, 0.1);
}
.message {
  display: flex;
  margin-bottom: 20px;
  gap: 12px;
}
.message.user {
  flex-direction: row-reverse;
}
.avatar {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  flex-shrink: 0;
}
.user .avatar {
  background: #409eff;
  color: white;
}
.assistant .avatar {
  background: #67c23a;
  color: white;
}
.content {
  max-width: 70%;
  padding: 12px;
  border-radius: 8px;
  line-height: 1.5;
  word-break: break-all;
}
.user .content {
  background: #e6f2ff;
}
.assistant .content {
  background: #f4f4f5;
}

/* 工单卡片样式 */
.ticket-card {
  border: 1px solid #dcdfe6;
  border-radius: 8px;
  margin-top: 10px;
  background: #fff;
  overflow: hidden;
}
.ticket-header {
  display: flex;
  justify-content: space-between;
  padding: 12px;
  background: #f8f9fa;
  border-bottom: 1px solid #ebeef5;
  font-weight: bold;
}
.ticket-body {
  padding: 12px;
  font-size: 14px;
  color: #606266;
}
.status.processing {
  color: #e6a23c;
  font-weight: normal;
}

.input-area {
  display: flex;
  gap: 10px;
}
input {
  flex: 1;
  padding: 12px;
  border: 1px solid #dcdfe6;
  border-radius: 6px;
  outline: none;
  font-size: 16px;
}
button {
  padding: 0 24px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 16px;
  transition: opacity 0.2s;
}
button:hover {
  opacity: 0.8;
}
.send-btn {
  background: #409eff;
  color: white;
}
.stop-btn {
  background: #f56c6c;
  color: white;
}
</style>
