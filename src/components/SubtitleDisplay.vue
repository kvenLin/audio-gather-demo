<template>
  <div class="subtitle-display">
    <div class="subtitle-options">
      <button @click="displayMode = 'original'" :class="{ active: displayMode === 'original' }">原文</button>
      <button @click="displayMode = 'translated'" :class="{ active: displayMode === 'translated' }">译文</button>
      <button @click="displayMode = 'bilingual'" :class="{ active: displayMode === 'bilingual' }">双语</button>
    </div>
    <div class="subtitle-content" ref="subtitleContent">
      <div v-for="(subtitle, index) in subtitles" :key="index">
        <p v-if="displayMode === 'original' || displayMode === 'bilingual'">{{ subtitle.original }}</p>
        <p v-if="displayMode === 'translated' || displayMode === 'bilingual'">{{ subtitle.translated }}</p>
      </div>
      <div class="current-subtitle">
        <p v-if="displayMode === 'original' || displayMode === 'bilingual'">{{ currentOriginal }}</p>
        <p v-if="displayMode === 'translated' || displayMode === 'bilingual'">{{ currentTranslated }}</p>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      displayMode: 'bilingual',
      subtitles: [],
      currentOriginal: '',
      currentTranslated: ''
    }
  },
  methods: {
    updateSubtitles(data) {
      if (data.original.ap) {
        this.currentOriginal = data.original.ap;
        this.currentTranslated = data.translation.ap;
      } else if (data.original.end) {
        this.subtitles.push({
          original: this.currentOriginal,
          translated: this.currentTranslated
        });
        this.currentOriginal = data.original.end;
        this.currentTranslated = data.translation.end;
      }
      this.$nextTick(() => {
        this.scrollToBottom();
      });
    },
    scrollToBottom() {
      const container = this.$refs.subtitleContent;
      container.scrollTop = container.scrollHeight;
    }
  }
}
</script>

<style scoped>
.subtitle-display {
  background-color: #f9f9f9;
  border-radius: 5px;
  overflow: hidden;
}

.subtitle-options {
  display: flex;
  background-color: #e9ecef;
}

.subtitle-options button {
  flex: 1;
  padding: 10px;
  border: none;
  background-color: transparent;
  cursor: pointer;
  transition: background-color 0.3s;
}

.subtitle-options button.active {
  background-color: #007bff;
  color: white;
}

.subtitle-content {
  padding: 20px;
  height: 300px;
  overflow-y: auto;
}

.subtitle-content p {
  margin: 0 0 10px;
}

.current-subtitle {
  border-top: 1px solid #ddd;
  padding-top: 10px;
  margin-top: 10px;
}
</style>