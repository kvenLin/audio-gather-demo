<template>
  <div class="batch-login">
    <div class="login-section">
      <h2>批量登录</h2>
      <div class="statistics">
        <p>成功登录人数: {{ successLoginCount }}/{{ totalAccounts }}</p>
        <p>正在采集音频人数: {{ activeRecordingCount }}</p>
      </div>
      <div class="input-area">
        <el-input
          type="textarea"
          v-model="accountsInput"
          :rows="5"
          placeholder="请输入账号和密码，每行一个，格式：账号,密码"
        />
        <language-selector @change-language-pair="handleLanguageChange" />
        <el-button type="primary" @click="handleBatchLogin" :loading="isLoading">
          批量登录
        </el-button>
        <el-button type="success" @click="handleStartAll" :loading="isStarting">
          一键开启同传
        </el-button>
        <el-button type="danger" @click="closeAllConnections">
          关闭所有连接
        </el-button>
      </div>

      <div class="accounts-list">
        <el-table :data="loginResults" style="width: 100%" :header-cell-style="{background:'#f5f7fa',color:'#606266'}" :cell-style="{padding:'5px'}">
          <el-table-column prop="account" label="账号" width="180" />
          <el-table-column prop="status" label="登录状态">
            <template #default="scope">
              <el-tag :type="scope.row.status === '成功' ? 'success' : 'danger'">
                {{ scope.row.status }}
              </el-tag>
            </template>
          </el-table-column>
          <el-table-column prop="connectionStatus" label="连接状态">
            <template #default="scope">
              <el-tag :type="getConnectionStatusType(scope.row.account)">
                {{ getConnectionStatus(scope.row.account) }}
              </el-tag>
            </template>
          </el-table-column>
          <el-table-column prop="message" label="信息" />
        </el-table>
      </div>
    </div>
  </div>
</template>

<script>
import LanguageSelector from './LanguageSelector.vue'
import pako from 'pako'

export default {
  name: 'BatchLogin',
  components: {
    LanguageSelector
  },
  data() {
    return {
      accountsInput: '',
      loginResults: [],
      isLoading: false,
      isStarting: false,
      tokens: new Map(),
      wsConnections: new Map(),
      audioContexts: new Map(),
      audioInputs: new Map(),
      processors: new Map(),
      streams: new Map(),
      apiBaseUrl: process.env.VUE_APP_API_BASE_URL || 'http://localhost:8080',
      wsBaseUrl: process.env.VUE_APP_WS_BASE_URL || 'ws://localhost:8080',
      currentLanguagePair: 'enus-zhcn',
      sampleRate: 48000,
      channels: 2,
      bufferSize: 512,
      bufferThreshold: 4096,
      deviceId: 'aggregate-device',
      audioBuffers: new Map(),
      lastSendTimes: new Map(),
      minSendInterval: 20,
      silenceThreshold: 300,
      maxSilenceDuration: 60000,
      isFirstSends: new Map(),
      lastAudioEnergyTimes: new Map(),
      recordingStartTimes: new Map(),
      silenceDurations: new Map(),
      connectionStatuses: new Map(),
      activeConnections: [], // 新增：用于追踪活跃的连接
      successLoginCount: 0,
      totalAccounts: 0,
    }
  },
  computed: {
    activeRecordingCount() {
      return this.activeConnections.length;
    }
  },
  watch: {
    loginResults: {
      handler(newResults) {
        this.successLoginCount = newResults.filter(result => result.status === '成功').length
        this.totalAccounts = newResults.length
      },
      deep: true
    }
  },
  methods: {
    parseAccounts() {
      return this.accountsInput
        .split('\n')
        .filter(line => line.trim())
        .map(line => {
          const [account, password] = line.split(',').map(item => item.trim())
          return { account, password }
        })
    },

    async login(account, password) {
      try {
        const response = await fetch(process.env.VUE_APP_LOGIN_SERVICE_URL, {
          method: 'POST',
          headers: {
            'Useraccount': account,
            'Password': password,
            'Source': 'AI_SIMULTANEOUS_INTERPRETATION',
            'Content-Type': 'application/x-www-form-urlencoded',
            'Iskeepmore': 'true'
          }
        })
        
        const data = await response.json()
        if (data.success && data.data.Authentication) {
          this.tokens.set(account, data.data.Authentication)
          return { success: true, message: data.msg, token: data.data.Authentication }
        }
        return { success: false, message: data.msg || '登录失败：未获取到token' }
      } catch (error) {
        return { success: false, message: `登录失败：${error.message}` }
      }
    },

    async handleBatchLogin() {
      this.isLoading = true
      this.loginResults = []
      this.successLoginCount = 0 // 重置计数
      this.totalAccounts = 0
      const accounts = this.parseAccounts()

      for (const { account, password } of accounts) {
        const loginResult = await this.login(account, password)
        this.loginResults.push({
          account,
          status: loginResult.success ? '成功' : '失败',
          message: loginResult.message
        })
        this.connectionStatuses.set(account, '未连接') // 初始化连接状态
      }
      this.isLoading = false
    },

    async handleStartAll() {
      if (this.isStarting) return
      this.isStarting = true
      
      try {
        console.log('开始获取音频流...')
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true })
        console.log('成功获取音频流:', stream.getAudioTracks()[0].label)
        
        // 清空活跃连接列表
        this.activeConnections = []
        
        for (const result of this.loginResults.filter(result => result.status === '成功')) {
          try {
            console.log(`[${result.account}] 准备建立连接...`)
            // 添加延迟，避免连接过于频繁
            await new Promise(resolve => setTimeout(resolve, 1000))
            
            // 1. 调用开启同传的接口获取WebSocket连接信息
            console.log(`[${result.account}] 调用开启同传接口...`)
            const response = await fetch(`${this.apiBaseUrl}/api/audio/action`, {
              method: 'POST',
              headers: {
                'Authorization': this.tokens.get(result.account),
                'Content-Type': 'application/json'
              },
              body: JSON.stringify({
                action: 'start',
                recordType: 'mixed',
                languagePair: this.currentLanguagePair,
                beginTime: Date.now()
              })
            })

            const data = await response.json()
            console.log(`[${result.account}] 开启同传接口返回:`, data)
            
            if (data.isSuccess) {
              const sessionId = data.data.sessionId
              const index = data.data.index
              
              // 2. 建立WebSocket连接
              const wsUrl = `${this.wsBaseUrl}/recordSubtitle/${sessionId}/${index}`
              console.log(`[${result.account}] 开始建立WebSocket连接: ${wsUrl}`)
              const ws = new WebSocket(wsUrl)
              
              // 3. 等待WebSocket连接建立
              await new Promise((resolve, reject) => {
                const timeout = setTimeout(() => {
                  reject(new Error('WebSocket连接超时'))
                }, 5000)

                ws.onopen = () => {
                  clearTimeout(timeout)
                  resolve()
                  
                  console.log(`[${result.account}] WebSocket连接成功建立`)
                  this.wsConnections.set(result.account, ws)
                  this.connectionStatuses.set(result.account, '已连接')
                  if (!this.activeConnections.includes(result.account)) {
                    this.activeConnections.push(result.account) // 添加到活跃连接数组
                  }
                  this.$forceUpdate()
                  
                  // 发送初始化数据
                  const payload = {
                    index: index,
                    open: {
                      audioBit: 16,
                      audioChannels: this.channels,
                      audioRatio: this.sampleRate
                    },
                    sessionId: sessionId,
                    ts: Date.now()
                  }
                  console.log(`[${result.account}] 发送初始化数据:`, payload)
                  ws.send(JSON.stringify(payload))
                  
                  // 初始化音频处理
                  console.log(`[${result.account}] 开始初始化音频处理...`)
                  this.initAudioProcessing(result.account, stream, sessionId, index)
                }

                ws.onerror = (error) => {
                  clearTimeout(timeout)
                  console.error(`[${result.account}] WebSocket连接错误:`, error)
                  reject(error)
                }
              })

              ws.onmessage = (event) => {
                try {
                  const data = JSON.parse(event.data)
                  console.log(`[${result.account}] 收到WebSocket消息:`, data)
                  if (data.code === 0) {
                    console.log(`[${result.account}] 收到字幕:`, data.data)
                  } else {
                    console.warn(`[${result.account}] 收到非字幕消息:`, data)
                  }
                } catch (error) {
                  console.error(`[${result.account}] 解析WebSocket消息失败:`, error, event.data)
                }
              }

              ws.onerror = (error) => {
                console.error(`[${result.account}] WebSocket错误:`, error)
                this.loginResults = this.loginResults.map(result => {
                  if (result.account === result.account) {
                    result.status = '失败'
                    result.message = 'WebSocket连接错误'
                  }
                  return result
                })
              }

              ws.onclose = (event) => {
                console.log(`[${result.account}] WebSocket连接关闭:`, {
                  code: event.code,
                  reason: event.reason,
                  wasClean: event.wasClean
                })
                this.stopAudioProcessing(result.account)
                const index = this.activeConnections.indexOf(result.account)
                if (index > -1) {
                  this.activeConnections.splice(index, 1) // 从活跃连接数组中移除
                }
                this.connectionStatuses.set(result.account, '已断开')
                this.$forceUpdate()
              }

            } else {
              throw new Error(`获取WebSocket连接信息失败: ${data.msg || '未知错误'}`)
            }
          } catch (error) {
            console.error(`[${result.account}] 开启同传失败:`, error)
            this.loginResults = this.loginResults.map(result => {
              if (result.account === result.account) {
                result.status = '失败'
                result.message = error.message
              }
              return result
            })
          }
        }
      } catch (error) {
        console.error('获取音频流失败:', error)
        alert('无法访问麦克风。请确保您已授予麦克风访问权限。')
      }
      
      this.isStarting = false
    },

    initAudioProcessing(account, stream, sessionId, index) {
      console.log(`[${account}] 初始化音频处理开始...`)
      this.streams.set(account, stream)
      
      const audioContext = new (window.AudioContext || window.webkitAudioContext)({ sampleRate: this.sampleRate })
      const audioInput = audioContext.createMediaStreamSource(stream)
      const processor = audioContext.createScriptProcessor(this.bufferSize, this.channels, this.channels)
      
      this.audioContexts.set(account, audioContext)
      this.audioInputs.set(account, audioInput)
      this.processors.set(account, processor)
      
      audioInput.connect(processor)
      processor.connect(audioContext.destination)
      
      this.audioBuffers.set(account, [])
      this.lastSendTimes.set(account, 0)
      this.isFirstSends.set(account, true)
      this.lastAudioEnergyTimes.set(account, Date.now())
      this.recordingStartTimes.set(account, Date.now())
      this.silenceDurations.set(account, 0)
      
      console.log(`[${account}] 音频处理初始化完成，开始处理音频数据...`)
      
      processor.onaudioprocess = (e) => {
        const ws = this.wsConnections.get(account)
        if (ws && ws.readyState === WebSocket.OPEN) {
          const leftChannel = e.inputBuffer.getChannelData(0)
          const rightChannel = e.inputBuffer.getChannelData(1)
          const interleaved = this.interleave(leftChannel, rightChannel)
          
          const audioEnergy = this.calculateAudioEnergy(interleaved)
          const currentTime = Date.now()
          
          let silenceDuration = this.silenceDurations.get(account)
          const isFirstSend = this.isFirstSends.get(account)
          const lastAudioEnergyTime = this.lastAudioEnergyTimes.get(account)
          
          if (audioEnergy < this.silenceThreshold) {
            silenceDuration += currentTime - lastAudioEnergyTime
            this.silenceDurations.set(account, silenceDuration)
            if (silenceDuration % 5000 === 0) { // 每5秒输出一次静音状态
              console.log(`[${account}] 当前处于静音状态，持续时间: ${silenceDuration/1000}秒`)
            }
          } else {
            if (silenceDuration > 0) {
              console.log(`[${account}] 检测到声音，结束静音状态`)
            }
            this.silenceDurations.set(account, 0)
            this.isFirstSends.set(account, false)
          }
          
          this.lastAudioEnergyTimes.set(account, currentTime)
          
          if (silenceDuration >= this.maxSilenceDuration) {
            if (isFirstSend) {
              console.log(`[${account}] 开始时持续60秒静音，停止录音且不发送数据`)
              this.stopAudioProcessing(account)
            } else {
              console.log(`[${account}] 中途静音时间超过60秒，停止录音`)
              this.stopAudioProcessing(account)
            }
            return
          }
          
          if (!isFirstSend) {
            let audioBuffer = this.audioBuffers.get(account)
            audioBuffer = audioBuffer.concat(Array.from(interleaved))
            this.audioBuffers.set(account, audioBuffer)
            
            const lastSendTime = this.lastSendTimes.get(account)
            if (audioBuffer.length >= this.bufferThreshold && 
                currentTime - lastSendTime >= this.minSendInterval) {
              this.sendAudioData(account, audioBuffer, sessionId, index)
              this.audioBuffers.set(account, [])
              this.lastSendTimes.set(account, currentTime)
            }
          } else if (currentTime - this.recordingStartTimes.get(account) >= this.maxSilenceDuration) {
            console.log(`[${account}] 开始时持续60秒静音，停止录音且不发送数据`)
            this.stopAudioProcessing(account)
          }
        } else {
          console.warn(`[${account}] WebSocket未连接或已关闭，无法发送音频数据`)
        }
      }
    },

    calculateAudioEnergy(buffer) {
      const int16Buffer = this.floatTo16BitPCM(buffer)
      const bytes = new Uint8Array(int16Buffer)
      let sum = 0
      const count = bytes.length / 2
      for (let i = 0; i < bytes.length; i += 2) {
        let sample = ((bytes[i + 1] << 8) | (bytes[i] & 0xFF))
        if (sample > 32767) {
          sample -= 65536
        }
        sum += sample * sample
      }
      return Math.sqrt(sum / count)
    },

    interleave(leftChannel, rightChannel) {
      const length = leftChannel.length + rightChannel.length
      const result = new Float32Array(length)

      let inputIndex = 0
      for (let i = 0; i < length;) {
        result[i++] = leftChannel[inputIndex]
        result[i++] = rightChannel[inputIndex]
        inputIndex++
      }
      return result
    },

    sendAudioData(account, audioData, sessionId, index) {
      const ws = this.wsConnections.get(account)
      if (!ws) {
        console.warn(`[${account}] WebSocket连接不存在，无法发送音频数据`)
        return
      }

      if (this.isFirstSends.get(account)) {
        console.log(`[${account}] 首次检测到声音，开始发送数据`)
        this.isFirstSends.set(account, false)
      }
      
      try {
        const compressedData = pako.gzip(this.floatTo16BitPCM(audioData))
        const base64Data = btoa(String.fromCharCode.apply(null, new Uint8Array(compressedData)))
        
        const payload = {
          audioBase64: base64Data,
          deviceId: this.deviceId,
          index: index,
          sessionId: sessionId,
          source: 'mixed',
          state: {
            dataQueueSize: 0,
            timestamp: Date.now()
          },
          ts: Date.now()
        }
        
        ws.send(JSON.stringify(payload))
        console.log(`[${account}] 成功发送音频数据，大小: ${compressedData.length} 字节`)
      } catch (error) {
        console.error(`[${account}] 发送音频数据失败:`, error)
      }
    },

    floatTo16BitPCM(input) {
      const output = new Int16Array(input.length)
      for (let i = 0; i < input.length; i++) {
        const s = Math.max(-1, Math.min(1, input[i]))
        output[i] = s < 0 ? s * 0x8000 : s * 0x7FFF
      }
      return output.buffer
    },

    stopAudioProcessing(account) {
      try {
        console.log(`[${account}] 开始停止音频处理...`)
        
        // 停止音频处理器
        const processor = this.processors.get(account)
        if (processor) {
          processor.disconnect()
          this.processors.delete(account)
          console.log(`[${account}] 已断开处理器连接`)
        }

        // 停止音频输入
        const audioInput = this.audioInputs.get(account)
        if (audioInput) {
          audioInput.disconnect()
          this.audioInputs.delete(account)
          console.log(`[${account}] 已断开音频输入连接`)
        }

        // 关闭音频上下文
        const audioContext = this.audioContexts.get(account)
        if (audioContext && audioContext.state !== 'closed') {
          audioContext.close()
          this.audioContexts.delete(account)
          console.log(`[${account}] 已关闭音频上下文`)
        }

        // 关闭WebSocket连接
        const ws = this.wsConnections.get(account)
        if (ws && ws.readyState !== WebSocket.CLOSED) {
          ws.close()
          this.wsConnections.delete(account)
          console.log(`[${account}] 已关闭WebSocket连接`)
        }

        // 停止音频流
        const stream = this.streams.get(account)
        if (stream) {
          stream.getTracks().forEach(track => {
            track.stop()
            console.log(`[${account}] 已停止音频轨道:`, track.label)
          })
          this.streams.delete(account)
        }

        // 清理其他资源
        this.audioBuffers.delete(account)
        this.lastSendTimes.delete(account)
        this.isFirstSends.delete(account)
        this.lastAudioEnergyTimes.delete(account)
        this.recordingStartTimes.delete(account)
        this.silenceDurations.delete(account)

        // 更新状态
        const index = this.activeConnections.indexOf(account)
        if (index > -1) {
          this.activeConnections.splice(index, 1)
        }
        this.connectionStatuses.set(account, '已断开')
        this.$forceUpdate()

        console.log(`[${account}] 音频处理停止完成`)
      } catch (error) {
        console.error(`[${account}] 停止音频处理时出错:`, error)
      }
    },

    getConnectionStatus(account) {
      const status = this.connectionStatuses.get(account);
      return status || '未连接';
    },

    getConnectionStatusType(account) {
      const status = this.getConnectionStatus(account);
      switch(status) {
        case '已连接':
          return 'success';
        case '已断开':
          return 'danger';
        default:
          return 'info';
      }
    },

    handleLanguageChange(languagePair) {
      this.currentLanguagePair = languagePair;
    },

    closeAllConnections() {
      console.log('开始关闭所有连接...')
      // 获取所有需要关闭的账号
      const accounts = Array.from(this.wsConnections.keys())
      
      // 逐个关闭连接
      accounts.forEach(account => {
        console.log(`正在关闭账号 ${account} 的连接`)
        this.stopAudioProcessing(account)
      })

      // 清空活跃连接数组
      this.activeConnections = []
      this.$forceUpdate()
      console.log('所有连接已关闭')
    }
  }
}
</script>

<style scoped>
.batch-login {
  padding: 20px;
}

.login-section {
  max-width: 800px;
  margin: 0 auto;
}

.statistics {
  background-color: #f5f7fa;
  padding: 15px;
  border-radius: 4px;
  margin-bottom: 20px;
}

.statistics p {
  margin: 5px 0;
  font-size: 14px;
  color: #606266;
}

.input-area {
  margin-bottom: 20px;
}

.input-area .el-textarea {
  margin-bottom: 15px;
}

.input-area .el-button {
  margin-right: 10px;
}

.accounts-list {
  margin-top: 20px;
}
</style>
