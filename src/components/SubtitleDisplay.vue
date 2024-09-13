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
        this.currentOriginal = data.original.end;
        this.currentTranslated = data.translation.end;
        this.subtitles.push({
          original: this.currentOriginal,
          translated: this.currentTranslated
        });
        this.currentOriginal = '';
        this.currentTranslated = '';
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
/* ... 保持现有的样式 ... */
.subtitle-content {
  padding: 20px;
  height: 300px; /* 设置一个固定高度 */
  overflow-y: auto; /* 添加垂直滚动条 */
}
</style>