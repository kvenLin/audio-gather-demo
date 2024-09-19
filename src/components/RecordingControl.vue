<template>
  <div class="recording-control">
    <button @click="toggleRecording" :class="{ 'recording': isRecording }">
      {{ isRecording ? '暂停' : (isPaused ? '继续' : '开始录音') }}
    </button>
    <button @click="stopRecording" :disabled="!isRecording && !isPaused">结束</button>
  </div>
</template>

<script>
import pako from 'pako';

export default {
  inject: ['getToken'],
  props: {
    currentLanguagePair: {
      type: String,
      required: true
    }
  },
  data() {
    return {
      isRecording: false,
      isPaused: false,
      socket: null,
      apiBaseUrl: process.env.VUE_APP_API_BASE_URL || 'http://localhost:8080',
      wsBaseUrl: process.env.VUE_APP_WS_BASE_URL || 'ws://localhost:8080',
      mediaRecorder: null,
      audioContext: null,
      audioInput: null,
      processor: null,
      stream: null,
      sessionId: null,
      index: null,
      deviceId: 'aggregate-device',
      energyThreshold: 0.01,
      audioBuffer: [],
      sampleRate: 48000, // 采样率
      desiredDuration: 10, // 期望的音频时长（毫秒）
      channels: 2, // 双通道
      bufferSize: 512, // 修改：使用最接近 10ms 的 2 的幂值
      bufferThreshold: 4096, // 约 85ms 的音频数据 (48000 * 0.085)
      lastSendTime: 0,
      minSendInterval: 100, // 最小发送间隔（毫秒）
      silenceThreshold: 1000,// 静音阈值
      silenceDuration: 0,// 静音持续时间
      maxSilenceDuration: 60000, // 60秒，单位毫秒
      isFirstSend: true,
      lastAudioEnergyTime: 0,
      isAudioStarted: false, // 新增：用于标记是否已经开始接收到音频数据
      recordingStartTime: 0,
    }
  },
  computed: {
    desiredSamples() {
      return Math.floor(this.sampleRate * this.desiredDuration / 1000);
    }
  },
  methods: {
    async toggleRecording() {
      if (!this.isRecording && !this.isPaused) {
        await this.startRecording();
      } else if (this.isRecording) {
        this.pauseRecording();
      } else if (this.isPaused) {
        await this.resumeRecording();
      }
    },

    async startRecording() {
      try {
        if (!this.stream) {
          if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
            throw new Error('您的浏览器不支持音频录制。请使用最新版本的Chrome、Firefox或Safari。');
          }
          this.stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        }
        await this.initializeRecording();
      } catch (error) {
        console.error('开始录音时发生错误:', error);
        alert(error.message || '无法访问麦克风。请确保您已授予麦克风访问权限，并且使用的是安全连接（HTTPS或localhost）。');
      }
    },

    async resumeRecording() {
      try {
        if (!this.stream) {
          this.stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        }
        await this.initializeRecording();
      } catch (error) {
        console.error('继续录音时发生错误:', error);
        alert(error.message || '无法继续录音。请重试。');
      }
    },

    async initializeRecording() {
      const response = await this.sendStartRequest();
      if (response.isSuccess) {
        this.sessionId = response.data.sessionId;
        this.index = response.data.index;
        this.isRecording = true;
        this.isPaused = false;
        this.connectWebSocket();
        await this.startAudioCapture();
        this.$emit('toggle-recording', this.isRecording);
      } else {
        console.error('开始录音失败:', response.msg);
        throw new Error(response.msg || '开始录音失败');
      }
    },

    async sendStartRequest() {
      const url = `${this.apiBaseUrl}/api/audio/action`;
      const data = {
        action: 'start',
        recordType: 'mixed',
        languagePair: this.currentLanguagePair,
        beginTime: Date.now()
      };
      const headers = {
        'Content-Type': 'application/json',
        'Authorization': this.getToken()
      };

      const response = await fetch(url, {
        method: 'POST',
        headers: headers,
        body: JSON.stringify(data)
      });

      return await response.json();
    },

    connectWebSocket() {
      const wsUrl = `${this.wsBaseUrl}/recordSubtitle/${this.sessionId}/${this.index}`;
      this.socket = new WebSocket(wsUrl);

      this.socket.onopen = () => {
        console.log('WebSocket连接已建立');
        this.sendInitialPayload();
      };

      this.socket.onmessage = (event) => {
        const data = JSON.parse(event.data);
        if (data.code === 0) {
          // 处理字幕数据
          this.$emit('subtitle-update', data);
        } else {
          // 处理其他类型的消息
          console.log('收到消息:', data);
        }
      };

      this.socket.onerror = (error) => {
        console.error('WebSocket错误:', error);
      };

      this.socket.onclose = () => {
        console.log('WebSocket连接已关闭');
        this.stopAudioCapture();
      };
    },

    sendInitialPayload() {
      const payload = {
        index: this.index,
        open: {
          audioBit: 16,
          audioChannels: this.channels,
          audioRatio: this.sampleRate
        },
        sessionId: this.sessionId,
        ts: Date.now()
      };
      this.socket.send(JSON.stringify(payload));
    },

    async startAudioCapture() {
      if (!this.stream) return;

      this.audioContext = new (window.AudioContext || window.webkitAudioContext)({ sampleRate: this.sampleRate });
      this.audioInput = this.audioContext.createMediaStreamSource(this.stream);
      this.processor = this.audioContext.createScriptProcessor(this.bufferSize, this.channels, this.channels);

      this.audioInput.connect(this.processor);
      this.processor.connect(this.audioContext.destination);

      this.recordingStartTime = Date.now();

      this.processor.onaudioprocess = (e) => {
        if (this.isRecording && this.socket && this.socket.readyState === WebSocket.OPEN) {
          const leftChannel = e.inputBuffer.getChannelData(0);
          const rightChannel = e.inputBuffer.getChannelData(1);
          const interleaved = this.interleave(leftChannel, rightChannel);
          
          const audioEnergy = this.calculateAudioEnergy(interleaved);
          const currentTime = Date.now();

          if (!this.isAudioStarted) {
            this.isAudioStarted = true;
            this.lastAudioEnergyTime = currentTime;
          }

          if (audioEnergy < this.silenceThreshold) {
            this.silenceDuration += currentTime - this.lastAudioEnergyTime;
          } else {
            this.silenceDuration = 0;
            this.isFirstSend = false;
          }

          this.lastAudioEnergyTime = currentTime;

          if (this.silenceDuration >= this.maxSilenceDuration) {
            if (this.isFirstSend) {
              console.log('开始时持续60秒静音，停止录音且不发送数据');
              this.stopRecording();
            } else {
              console.log('中途静音时间超过60秒，停止录音');
              this.stopRecording();
            }
            return;
          }

          if (!this.isFirstSend) {
            this.audioBuffer = this.audioBuffer.concat(Array.from(interleaved));
            
            if (this.audioBuffer.length >= this.bufferThreshold && 
                currentTime - this.lastSendTime >= this.minSendInterval) {
              this.sendAudioData(this.audioBuffer);
              this.audioBuffer = [];
              this.lastSendTime = currentTime;
            }
          } else if (currentTime - this.recordingStartTime >= this.maxSilenceDuration) {
            console.log('开始时持续60秒静音，停止录音且不发送数据');
            this.stopRecording();
          }
        }
      };
    },

    calculateAudioEnergy(buffer) {
      const int16Buffer = this.floatTo16BitPCM(buffer);
      const bytes = new Uint8Array(int16Buffer);
      let sum = 0;
      const count = bytes.length / 2;
      for (let i = 0; i < bytes.length; i += 2) {
        let sample = ((bytes[i + 1] << 8) | (bytes[i] & 0xFF));
        if (sample > 32767) {
          sample -= 65536;
        }
        sum += sample * sample;
      }
      return Math.sqrt(sum / count);
    },

    interleave(leftChannel, rightChannel) {
      const length = leftChannel.length + rightChannel.length;
      const result = new Float32Array(length);

      let inputIndex = 0;
      for (let i = 0; i < length; ) {
        result[i++] = leftChannel[inputIndex];
        result[i++] = rightChannel[inputIndex];
        inputIndex++;
      }
      return result;
    },

    sendAudioData(audioData) {
      if (this.isFirstSend) {
        console.log('首次检测到声音，开始发送数据');
        this.isFirstSend = false;
      }
      
      const compressedData = pako.gzip(this.floatTo16BitPCM(audioData));
      const base64Data = btoa(String.fromCharCode.apply(null, new Uint8Array(compressedData)));
      
      const payload = {
        audioBase64: base64Data,
        deviceId: this.deviceId,
        index: this.index,
        sessionId: this.sessionId,
        source: 'mixed',
        state: {
          dataQueueSize: 0,
          timestamp: Date.now()
        },
        ts: Date.now()
      };
      
      this.socket.send(JSON.stringify(payload));
    },

    floatTo16BitPCM(input) {
      const output = new Int16Array(input.length);
      for (let i = 0; i < input.length; i++) {
        const s = Math.max(-1, Math.min(1, input[i]));
        output[i] = s < 0 ? s * 0x8000 : s * 0x7FFF;
      }
      return output.buffer;
    },

    pauseRecording() {
      this.isRecording = false;
      this.isPaused = true;
      this.$emit('toggle-recording', this.isRecording);
      this.disconnectAudio();
    },

    stopRecording() {
      this.isRecording = false;
      this.isPaused = false;
      this.isAudioStarted = false; // 重置音频开始标记
      this.silenceDuration = 0; // 重置静音持续时间
      this.isFirstSend = true; // 重置第一次发送标识
      this.recordingStartTime = 0; // 重置录音开始时间
      this.$emit('stop-recording');
      this.disconnectAll();
    },

    disconnectAudio() {
      if (this.processor) {
        this.processor.disconnect();
        this.processor.onaudioprocess = null;
      }
      if (this.audioInput) {
        this.audioInput.disconnect();
      }
      if (this.audioContext) {
        this.audioContext.close();
      }
      this.audioBuffer = [];
    },

    disconnectAll() {
      this.disconnectAudio();
      if (this.socket) {
        this.socket.close();
        this.socket = null;
      }
      if (this.stream) {
        this.stream.getTracks().forEach(track => track.stop());
        this.stream = null;
      }
    },

    stopAudioCapture() {
      if (this.processor) {
        this.processor.disconnect();
        this.processor.onaudioprocess = null;
      }
      if (this.audioInput) {
        this.audioInput.disconnect();
      }
      if (this.audioContext) {
        this.audioContext.close();
      }
      if (this.stream) {
        this.stream.getTracks().forEach(track => track.stop());
        this.stream = null;
      }
      this.audioBuffer = [];
    }
  }
}
</script>

<style scoped>
.recording-control {
  display: flex;
  gap: 10px;
}

button {
  flex: 1;
  padding: 10px 20px;
  font-size: 16px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
}

button:first-child {
  background-color: #28a745;
  color: white;
}

button:first-child.recording {
  background-color: #dc3545;
}

button:last-child {
  background-color: #6c757d;
  color: white;
}

button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>