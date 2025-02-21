<template>
  <div id="app">
    <div class="container">
      <h1 class="title">AI诊疗调试</h1>
      
      <div class="config-section" style="display: none;">
        <input 
          v-model="apiKey" 
          type="password" 
          placeholder="请输入 API Key"
          class="api-key-input"
        />
      </div>

      <div class="medical-form">
        <div class="form-group">
          <label>基本信息</label>
          <div class="form-row">
            <div class="form-field">
              <label class="sub-label">年龄</label>
              <input v-model="patientInfo.age" class="form-input small" />
            </div>
            <div class="form-field">
              <label class="sub-label">性别</label>
              <select v-model="patientInfo.gender" class="form-input small">
                <option value="男">男</option>
                <option value="女">女</option>
              </select>
            </div>
            <div class="form-field">
              <label class="sub-label">体重(kg)</label>
              <input v-model="patientInfo.weight" class="form-input small" />
            </div>
          </div>
        </div>

        <div class="form-group">
          <label class="sub-label">主诉</label>
          <textarea v-model="patientInfo.mainComplaint" class="form-input" rows="2" placeholder="请详细描述患者的主要症状和不适"></textarea>
        </div>

        <div class="form-group">
          <label class="sub-label">现病史</label>
          <textarea v-model="patientInfo.presentIllness" rows="3" class="form-input"></textarea>
        </div>

        <div class="form-group">
          <label class="sub-label">既往史</label> 
          <textarea v-model="patientInfo.pastHistory" rows="2" class="form-input"></textarea>
        </div>

        <div class="form-group">
          <div class="form-row">
            <div class="form-field">
              <label class="sub-label">体温(℃)</label>
              <input v-model="patientInfo.temperature" class="form-input small" />
            </div>
            <div class="form-field">
              <label class="sub-label">血压(mmHg)</label>
              <input v-model="patientInfo.bloodPressure" class="form-input medium" />
            </div>
            <div class="form-field">
              <label class="sub-label">脉搏(次/分)</label>
              <input v-model="patientInfo.pulse" class="form-input small" />
            </div>
            <div class="form-field">
              <label class="sub-label">呼吸(次/分)</label>
              <input v-model="patientInfo.breathing" class="form-input small" />
            </div>
          </div>
        </div>

        <div class="form-group">
          <div class="form-row wrap">
            <div class="form-field">
              <label class="sub-label">舌质</label>
              <input v-model="patientInfo.tongue" class="form-input medium" />
            </div>
            <div class="form-field">
              <label class="sub-label">舌苔</label>
              <input v-model="patientInfo.coating" class="form-input medium" />
            </div>
            <div class="form-field">
              <label class="sub-label">舌形</label>
              <input v-model="patientInfo.tongueShape" class="form-input medium" />
            </div>
            <div class="form-field">
              <label class="sub-label">脉象</label>
              <input v-model="patientInfo.pulseType" class="form-input medium" />
            </div>
            <div class="form-field">
              <label class="sub-label">指纹</label>
              <input v-model="patientInfo.fingerprint" class="form-input medium" />
            </div>
          </div>
        </div>

        <button @click="generateContent" class="form-button">生成问诊内容</button>
      </div>

      <div class="chat-container">
        <div class="input-section">
          <textarea 
            v-model="userInput" 
            placeholder="请输入您要发送给模型的内容..."
            rows="5"
            class="message-input"
          ></textarea>
          <button @click="sendRequest" :disabled="loading" class="send-button">
            <span v-if="!loading">发送请求</span>
            <div v-else class="loading-spinner"></div>
          </button>
        </div>

        <div class="loading-message" v-if="loading">
          <div class="dots-loading">
            <span></span>
            <span></span>
            <span></span>
          </div>
          正在等待模型响应...
        </div>

        <div class="response-section" v-if="streamResponse">
          <h3>响应结果</h3>
          <div class="response-content">
            <pre>{{ streamResponse }}</pre>
            <div class="typing-indicator" v-if="loading">▋</div>
          </div>
        </div>

        <div class="error-message" v-if="error">
          <i class="error-icon">⚠️</i>
          {{ error }}
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import OpenAI from 'openai'
import { conversationContext } from './bing.js'

export default {
  name: 'App',
  data() {
    return {
      userInput: '',
      error: null,
      loading: false,
      apiKey: 'sk-f4573b183ccf429783708cbf75d27a02',
      openai: null,
      streamResponse: null,
      messages: [...conversationContext],  // 初始化消息历史
      patientInfo: {
        age: "49",
        gender: "男",
        weight: "90",
        mainComplaint: "反复咳嗽咳痰5年，最近一周发热",
        presentIllness: "患者5年内反复出现咳嗽咳痰，痰中带血，曾被诊断为支气管扩张感染",
        pastHistory: "患者曾患高血压，糖尿病",
        temperature: "39",
        bloodPressure: "150/90",
        pulse: "77",
        breathing: "50",
        tongue: "淡红",
        coating: "厚",
        tongueShape: "毛舌",
        pulseType: "沉",
        fingerprint: "沉"
      }
    }
  },
  methods: {
    async sendRequest() {
      if (!this.apiKey) {
        this.error = '请先输入 API Key'
        return
      }

      if (!this.openai) {
        this.initOpenAI()
      }
      
      this.loading = true
      this.error = null
      this.streamResponse = ''
      
      // 添加用户输入到消息历史
      const finalString = '你是一名专业的坐诊医师，我会提供给你患者的信息，你要帮我生成:辅助检查结果一句话,治则一句话,辨病辨证依据一句话，一段治疗方案，请结合上下文帮我生成至少一对中医疾病和症候（也可以多对，可能性由高到低生成）。不用markdown格式。'
      this.messages.push({
        role: "user",
        content: finalString + this.userInput
      })
      
      try {
        const completion = await this.openai.chat.completions.create({
          model: 'qwen-max',
          messages: this.messages,  // 使用完整的消息历史
          stream: true,
          stream_options: {
            include_usage: true
          }
        })

        let assistantResponse = ''
        for await (const chunk of completion) {
          if (Array.isArray(chunk.choices) && chunk.choices.length > 0) {
            const content = chunk.choices[0].delta.content || ''
            assistantResponse += content
            this.streamResponse += content
          }
        }

        // 将助手的回复添加到消息历史
        this.messages.push({
          role: "assistant",
          content: assistantResponse
        })
        
      } catch (err) {
        console.error('API 调用错误:', err)
        this.error = '请求失败：' + (err.message || '未知错误')
      } finally {
        this.loading = false
      }
    },
    initOpenAI() {
      try {
        this.openai = new OpenAI({
          apiKey: this.apiKey,
          baseURL: "https://dashscope.aliyuncs.com/compatible-mode/v1",
          dangerouslyAllowBrowser: true  // 允许在浏览器端使用
        })
      } catch (err) {
        this.error = '初始化失败：' + err.message
      }
    },
    generateContent() {
      const info = this.patientInfo
      this.userInput = `患者今年${info.age}岁，${info.gender}，主诉：${info.mainComplaint}，现病史：${info.presentIllness}，既往史：${info.pastHistory}，体温${info.temperature}度，收缩压${info.bloodPressure}，脉搏${info.pulse}次，呼吸次数${info.breathing}次，体重${info.weight}kg，舌质${info.tongue}，舌苔${info.coating}，舌形${info.tongueShape}，脉象${info.pulseType}，指纹${info.fingerprint}`
    }
  }
}
</script>

<style>
#app {
  font-family: 'PingFang SC', 'Microsoft YaHei', sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  min-height: 100vh;
  background: #f5f7fa;
  padding: 20px 0;
}

.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  background: white;
  border-radius: 12px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
}

.title {
  text-align: center;
  color: #409EFF;
  font-size: 24px;
  margin-bottom: 30px;
}

.chat-container {
  background: #fff;
  border-radius: 8px;
}

.input-section {
  margin: 20px 0;
  position: relative;
}

.message-input {
  width: 95%;
  padding: 15px;
  border: 2px solid #e8e8e8;
  border-radius: 8px;
  resize: vertical;
  font-size: 16px;
  transition: border-color 0.3s;
  line-height: 1.6;
}

.message-input:focus {
  outline: none;
  border-color: #409EFF;
}

.send-button {
  background-color: #409EFF;
  color: white;
  padding: 12px 24px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 16px;
  transition: background-color 0.3s;
  margin-top: 10px;
  min-width: 120px;
  min-height: 44px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.send-button:hover {
  background-color: #66b1ff;
}

.send-button:disabled {
  background-color: #a0cfff;
  cursor: not-allowed;
}

.response-section {
  margin-top: 30px;
  padding: 20px;
  background-color: #f8f9fa;
  border-radius: 8px;
  border-left: 4px solid #409EFF;
}

.response-section h3 {
  color: #409EFF;
  margin-top: 0;
  font-weight: 500;
}

.response-content {
  position: relative;
  padding: 10px;
}

.response-content pre {
  white-space: pre-wrap;
  word-wrap: break-word;
  margin: 0;
  font-family: 'PingFang SC', 'Microsoft YaHei', sans-serif;
  line-height: 1.6;
  font-size: 15px;
}

.error-message {
  margin-top: 20px;
  padding: 12px 20px;
  background-color: #fef0f0;
  color: #f56c6c;
  border-radius: 6px;
  display: flex;
  align-items: center;
}

.error-icon {
  margin-right: 8px;
  font-style: normal;
}

.loading-message {
  text-align: center;
  color: #909399;
  margin: 20px 0;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
}

.dots-loading {
  display: flex;
  gap: 4px;
}

.dots-loading span {
  width: 6px;
  height: 6px;
  background-color: #409EFF;
  border-radius: 50%;
  display: inline-block;
  animation: dots 1.4s infinite ease-in-out both;
}

.dots-loading span:nth-child(1) { animation-delay: -0.32s; }
.dots-loading span:nth-child(2) { animation-delay: -0.16s; }

@keyframes dots {
  0%, 80%, 100% { transform: scale(0); }
  40% { transform: scale(1); }
}

.api-key-input {
  width: 95%;
  padding: 12px;
  border: 2px solid #e8e8e8;
  border-radius: 8px;
  font-size: 14px;
  transition: border-color 0.3s;
}

.api-key-input:focus {
  outline: none;
  border-color: #409EFF;
}

.typing-indicator {
  display: inline-block;
  animation: blink 1s step-end infinite;
  margin-left: 2px;
  font-weight: bold;
  color: #409EFF;
}

@keyframes blink {
  0%, 100% { opacity: 1; }
  50% { opacity: 0; }
}

.medical-form {
  background: #fff;
  padding: 20px;
  border-radius: 8px;
  margin-bottom: 20px;
}

.form-group {
  margin-bottom: 15px;
}

.form-group label {
  display: block;
  margin-bottom: 8px;
  color: #606266;
  font-weight: 500;
}

.form-row {
  display: flex;
  gap: 10px;
  margin-bottom: 10px;
}

.form-row.wrap {
  flex-wrap: wrap;
}

.form-input {
  padding: 8px 12px;
  border: 1px solid #dcdfe6;
  border-radius: 4px;
  font-size: 14px;
  transition: all 0.3s;
}

.form-input:focus {
  outline: none;
  border-color: #409EFF;
}

.form-input.small {
  width: 80px;
}

.form-input.medium {
  width: 120px;
}

textarea.form-input {
  width: 100%;
  resize: vertical;
}

.form-button {
  background-color: #67c23a;
  color: white;
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.form-button:hover {
  background-color: #85ce61;
}

.form-field {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.sub-label {
  font-size: 12px;
  color: #606266;
  margin-bottom: 2px;
}

.form-field .form-input {
  margin-top: 0;
}

.form-row {
  align-items: flex-end;
}

.form-row.wrap {
  align-items: flex-start;
}
</style>
