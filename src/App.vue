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
          <textarea v-model="patientInfo.mainComplaint" class="form-input" rows="2" placeholder="请详细描述患者的主要症状和不适" style="margin-right: 10px;"></textarea>
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
          <div class="message-input" style="min-height: 100px">{{ userInput }}</div>
          <button 
            @click="sendRequest" 
            :disabled="loading || !userInput.trim()" 
            class="send-button"
          >
            <span v-if="!loading">发送请求</span>
            <div v-else class="loading-spinner"></div>
          </button>
        </div>

        <div class="loading-message" v-if="loading">
          <div class="progress-bar">
            <div class="progress-bar-inner"></div>
          </div>
          <span class="loading-text">AI 正在分析患者信息...</span>
        </div>

        <!-- 流式输出显示区域，始终显示原始字符串 -->
        <div v-if="streamResponse" class="stream-output">
          <pre>{{ streamResponse }}</pre>
        </div>

        <!-- 最终渲染结果 -->
        <div class="response-section" v-if="!loading && parsedResult">
          <h3>诊断结果</h3>
          <div>
            <div class="diagnosis-item">
              <h4>辅助检查建议</h4>
              <p>{{ parsedResult.auxiliaryExamination }}</p>
            </div>
            
            <div class="diagnosis-item">
              <h4>治则</h4>
              <p>{{ parsedResult.treatmentPrinciple }}</p>
            </div>
            
            <div class="diagnosis-item">
              <h4>辨病辨证依据</h4>
              <p>{{ parsedResult.diagnosisBasis }}</p>
            </div>
            
            <div class="diagnosis-item">
              <h4>治疗方案</h4>
              <p>{{ parsedResult.treatmentPlan }}</p>
            </div>
            
            <div class="diagnosis-item">
              <h4>中医诊断</h4>
              <div class="diagnosis-list">
                <div v-for="(item, index) in parsedResult.tcmDiagnosis" 
                     :key="index"
                     class="diagnosis-pair">
                  <div class="diagnosis-text">
                    {{ item.disease }}：{{ item.syndrome }}
                  </div>
                </div>
              </div>
            </div>
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
export default {
  name: 'App',
  data() {
    return {
      userInput: '',
      error: null,
      loading: false,
      apiKey: process.env.VUE_APP_API_KEY,
      appId: process.env.VUE_APP_APP_ID,
      pipelineIds: process.env.VUE_APP_PIPELINE_IDS,
      streamResponse: null,
      parsedResult: null,
      patientInfo: {
        age: "49",
        gender: "男",
        weight: "90",
        mainComplaint: "反复头晕、头痛伴心悸5年，加重1周",
        presentIllness: `患者5年前无明显诱因出现头晕、头痛，伴有心悸，就诊于当地医院，测血压为160/100 mmHg，诊断为"高血压"。此后间断服用降压药物（具体药物不详），血压控制不佳。1周前因工作压力大，头晕、头痛症状加重，伴有视物模糊，无恶心、呕吐，无肢体活动障碍。自测血压为170/110 mmHg，遂来就诊。`,
        pastHistory: "患者曾有高血压、糖尿病、高脂血症病史，无药物过敏史",
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
  mounted() {
    console.log(process.env);
  },
  methods: {
    parseDiagnosis(diagnosisStr) {
      try {
        // 移除末尾的逗号和省略号
        const cleanStr = diagnosisStr.replace(/[,...]$/, '')
        // 按逗号分割不同的诊断对
        const pairs = cleanStr.split(',')
        
        // 直接返回所有诊断对，不进行分组
        return pairs.map(pair => {
          // 分割疾病和证候
          const [disease, syndrome] = pair.split('|')
          // 分别解析疾病和证候的名称和编码
          const [diseaseName, diseaseCode] = disease.split('-')
          const [syndromeName, syndromeCode] = syndrome.split('-')
          
          return {
            disease: `${diseaseName}（${diseaseCode}）`,
            syndrome: `${syndromeName}（${syndromeCode}）`
          }
        })
      } catch (e) {
        console.error('解析诊断数据失败:', e)
      }
      return []
    },

    parseResponse(text) {
      const parts = text.split('#')
      if (parts.length >= 5) {
        return {
          auxiliaryExamination: parts[0],
          treatmentPrinciple: parts[1],
          diagnosisBasis: parts[2],
          treatmentPlan: parts[3],
          tcmDiagnosis: this.parseDiagnosis(parts[4])
        }
      }
      return null
    },

    async sendRequest() {
      if (!this.apiKey) {
        this.error = '请先输入 API Key'
        return
      }

      this.loading = true
      this.error = null
      this.streamResponse = ''

      try {
        const url = `https://dashscope.aliyuncs.com/api/v1/apps/${this.appId}/completion`
        const strBefore = `你是一名专业的坐诊医师，请基于大模型和应用里的知识库数据分析以下患者信息。请严格按照我提供的格式输出：
一句辅助检查建议#一句治则#一句辨病辨证依据#一句治疗方案#疾病名称-疾病编码|证候名称-证候编码,疾病名称-疾病编码|证候名称-证候编码

请分析以下患者信息：`

        const strAfter = `
请记住：
1. 必须按照示例格式输出，用#分隔每个部分
2. 每个字段一句话概括
3. 最后一部分是疾病和证候对，可以返回多对，用逗号分隔
4. 每对中疾病和证候用|分隔，名称和编码用-分隔
5. 疾病和症候内容必须基于知识库数据`

        const strper = strBefore + this.userInput + strAfter
        const requestData = {
          input: {
            prompt: strper
          },
          parameters: {
            rag_options: {
              pipeline_ids: this.pipelineIds
            },
            incremental_output: true // 启用增量输出
          },
          debug: {}
        }

        // 使用 fetch 发送请求并处理流式响应
        const response = await fetch(url, {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${this.apiKey}`,
            'Content-Type': 'application/json',
            'X-DashScope-SSE': 'enable' // 启用 SSE
          },
          body: JSON.stringify(requestData)
        })

        if (!response.ok) {
          const errorData = await response.json()
          throw new Error(errorData.message || '请求失败')
        }

        // 创建 SSE 读取器
        const reader = response.body.getReader()
        const decoder = new TextDecoder()
        let fullResponse = ''
        let tempResponse = ''

        // 读取流式响应
        try {
          let reading = true
          while (reading) {
            const { done, value } = await reader.read()
            if (done) {
              reading = false
              continue
            }

            // 解码二进制数据
            const chunk = decoder.decode(value)
            const lines = chunk.split('\n')

            // 处理每一行数据
            for (const line of lines) {
              if (line.trim() === '') continue
              if (line.startsWith('data:')) {
                try {
                  const data = JSON.parse(line.slice(5))
                  if (data.output && data.output.text) {
                    tempResponse += data.output.text
                    // 实时显示流式输出
                    this.streamResponse = tempResponse
                  }
                } catch (e) {
                  console.warn('解析流式数据失败:', e)
                }
              }
            }
          }
        } finally {
          reader.releaseLock()
        }

        // 所有数据接收完毕，进行最终处理
        fullResponse = tempResponse
        const parsedData = this.parseResponse(fullResponse)

        // 保持流式输出显示原始字符串
        this.streamResponse = fullResponse

        // 添加一个新的属性用于存储解析后的数据
        if (parsedData) {
          this.parsedResult = parsedData
        } else {
          this.parsedResult = null
        }

      } catch (err) {
        console.error('API 调用错误:', err)
        this.error = '请求失败：' + (err.message || '未知错误')
      } finally {
        this.loading = false
      }
    },
    generateContent() {
      const info = this.patientInfo
      const prompt = `患者今年${info.age}岁，${info.gender}，主诉：${info.mainComplaint}，
现病史：${info.presentIllness}，既往史：${info.pastHistory}，
体温${info.temperature}度，收缩压${info.bloodPressure}，
脉搏${info.pulse}次，呼吸次数${info.breathing}次，体重${info.weight}kg，
舌质${info.tongue}，舌苔${info.coating}，舌形${info.tongueShape}，
脉象${info.pulseType}，指纹${info.fingerprint}`
      this.userInput = prompt
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
  /* 使用渐变背景 */
  background: linear-gradient(135deg, #f5f7fa 0%, #e4e7eb 100%);
  padding: 20px 0;
}

.container {
  max-width: 1000px; /* 增加宽度 */
  margin: 0 auto;
  padding: 30px;
  background: rgba(255, 255, 255, 0.95);
  border-radius: 16px;
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.1);
  backdrop-filter: blur(10px); /* 毛玻璃效果 */
}

.title {
  text-align: center;
  color: #2b5876;
  font-size: 28px;
  margin-bottom: 40px;
  position: relative;
  /* 添加科技感装饰 */
  &:after {
    content: '';
    position: absolute;
    bottom: -10px;
    left: 50%;
    transform: translateX(-50%);
    width: 60px;
    height: 3px;
    background: linear-gradient(90deg, #2b5876, #4e4376);
    border-radius: 3px;
  }
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
  background: linear-gradient(45deg, #2b5876, #4e4376);
  color: white;
  padding: 12px 24px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 16px;
  transition: all 0.3s;
  margin-top: 10px;
  min-width: 120px;
  min-height: 44px;
  display: flex;
  align-items: center;
  justify-content: center;
  
  &:hover:not(:disabled) {
    transform: translateY(-2px);
    box-shadow: 0 6px 20px rgba(43, 88, 118, 0.3);
  }
  
  &:disabled {
    background: linear-gradient(45deg, #a8b4c0, #b3b0ba);
    cursor: not-allowed;
    opacity: 0.7;
    transform: none;
    box-shadow: none;
  }
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
  margin: 30px 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 15px;
}

.loading-text {
  color: #2b5876;
  font-size: 15px;
  font-weight: 500;
}

.progress-bar {
  width: 300px;
  height: 4px;
  background: rgba(43, 88, 118, 0.1);
  border-radius: 2px;
  overflow: hidden;
  position: relative;
}

.progress-bar-inner {
  position: absolute;
  left: 0;
  top: 0;
  height: 100%;
  width: 30%;
  background: linear-gradient(90deg, #2b5876, #4e4376);
  border-radius: 2px;
  animation: progress 1.5s ease-in-out infinite;
}

@keyframes progress {
  0% {
    left: -30%;
  }
  100% {
    left: 100%;
  }
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
  background: linear-gradient(to right, #ffffff, #f8f9fa);
  padding: 25px;
  border-radius: 12px;
  margin-bottom: 30px;
  border: 1px solid rgba(65, 184, 255, 0.1);
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

/* 添加新的宽度类 */
.form-input.full {
  width: 100%;
  max-width: 800px;
}

textarea.form-input {
  width: 98%;
  resize: vertical;
}

.form-button {
  background: linear-gradient(45deg, #2b5876, #4e4376);
  color: white;
  padding: 12px 30px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s;
  font-weight: 500;
  box-shadow: 0 4px 15px rgba(43, 88, 118, 0.2);
  
  &:hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 20px rgba(43, 88, 118, 0.3);
  }
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

.diagnosis-container {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.diagnosis-item {
  background: white;
  padding: 20px;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.05);
  margin: 8px 0;
  border: 1px solid rgba(65, 184, 255, 0.1);
  transition: all 0.3s ease;
  
  &:hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 25px rgba(0, 0, 0, 0.08);
  }
}

.diagnosis-item h4 {
  color: #2b5876;
  margin: 0 0 15px 0;
  font-size: 18px;
  font-weight: 600;
  display: flex;
  align-items: center;
  
  &:before {
    content: '';
    display: inline-block;
    width: 4px;
    height: 18px;
    background: linear-gradient(to bottom, #2b5876, #4e4376);
    margin-right: 10px;
    border-radius: 2px;
  }
}

.diagnosis-item p {
  margin: 0;
  color: #2c3e50;
  line-height: 1.6;
}

.diagnosis-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.diagnosis-pair {
  padding: 15px;
  background: linear-gradient(to right, #f8f9fa, #ffffff);
  border-radius: 8px;
  border-left: 4px solid #2b5876;
  transition: all 0.3s ease;
  
  &:hover {
    background: linear-gradient(to right, #f1f4f7, #ffffff);
    border-left-color: #4e4376;
  }
}

.diagnosis-text {
  color: #2c3e50;
  font-size: 15px;
  line-height: 1.6;
}

.syndrome-item {
  display: inline-block;
}

.syndrome-separator {
  margin: 0 4px;
  color: #909399;
}

/* 添加动画效果 */
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

.diagnosis-item {
  animation: fadeIn 0.5s ease forwards;
}

.stream-output {
  margin: 20px 0;
  padding: 20px;
  background: rgba(255, 255, 255, 0.95);
  border-radius: 8px;
  border-left: 4px solid #2b5876;
  font-family: 'PingFang SC', 'Microsoft YaHei', sans-serif;
  line-height: 1.6;
  overflow-x: hidden;  /* 防止水平溢出 */
  animation: fadeIn 0.3s ease;
}

.stream-output pre {
  margin: 0;
  font-family: inherit;
  font-size: 15px;
  white-space: pre-wrap;  /* 允许文本换行 */
  word-break: break-all;  /* 在任意字符间换行 */
  overflow-wrap: break-word;  /* 长单词换行 */
  max-width: 100%;  /* 限制最大宽度 */
}
</style>
