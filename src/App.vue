<template>
  <div id="app">
    <header>
      <h1>AI同传</h1>
    </header>
    <main>
      <div class="control-panel">
        <div class="token-input">
          <label for="token">API Token:</label>
          <input type="text" id="token" v-model="token" @input="saveToken" placeholder="输入您的API Token">
        </div>
        <LanguageSelector @change-language-pair="handleLanguagePairChange" />
        <RecordingControl 
          @toggle-recording="handleToggleRecording"
          @stop-recording="handleStopRecording"
          @subtitle-update="handleSubtitleUpdate"
          :currentLanguagePair="currentLanguagePair"
        />
      </div>
      <SubtitleDisplay ref="subtitleDisplay" />
    </main>
  </div>
</template>

<script>
import RecordingControl from './components/RecordingControl.vue'
import LanguageSelector from './components/LanguageSelector.vue'
import SubtitleDisplay from './components/SubtitleDisplay.vue'

export default {
  name: 'App',
  components: {
    RecordingControl,
    LanguageSelector,
    SubtitleDisplay
  },
  data() {
    return {
      currentLanguagePair: 'enus-zhcn',
      token: ''
    }
  },
  created() {
    // 从本地存储加载token
    this.token = localStorage.getItem('apiToken') || '';
  },
  methods: {
    handleToggleRecording(isRecording) {
      console.log('录音状态:', isRecording);
    },
    handleStopRecording() {
      console.log('录音已停止');
    },
    handleLanguagePairChange(languagePair) {
      this.currentLanguagePair = languagePair;
      console.log('当前语言对:', languagePair);
    },
    handleSubtitleUpdate(data) {
      this.$refs.subtitleDisplay.updateSubtitles(data);
    },
    receiveSubtitles(original, translated) {
      this.$refs.subtitleDisplay.updateSubtitles(original, translated);
    },
    saveToken() {
      localStorage.setItem('apiToken', this.token);
    }
  },
  provide() {
    return {
      getToken: () => this.token
    }
  }
}
</script>

<style>
body {
  font-family: Arial, sans-serif;
  background-color: #f0f0f0;
  margin: 0;
  padding: 0;
}

#app {
  max-width: 800px;
  margin: 0 auto;
  background-color: white;
  box-shadow: 0 0 10px rgba(0,0,0,0.1);
  border-radius: 5px;
  overflow: hidden;
}

header {
  background-color: #007bff;
  color: white;
  padding: 20px;
  text-align: center;
}

h1 {
  margin: 0;
}

main {
  padding: 20px;
}

.control-panel {
  display: flex;
  flex-direction: column;
  gap: 20px;
  margin-bottom: 20px;
}

.token-input {
  display: flex;
  flex-direction: column;
}

.token-input label {
  margin-bottom: 5px;
}

.token-input input {
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 3px;
  font-size: 16px;
}
</style>