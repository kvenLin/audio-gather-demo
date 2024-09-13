<template>
  <div class="language-selector">
    <label for="language-pair">选择语言对：</label>
    <select id="language-pair" v-model="selectedLanguagePair" @change="changeLanguagePair">
      <option v-for="pair in languagePairs" :key="pair.value" :value="pair.value">
        {{ pair.label }}
      </option>
    </select>
  </div>
</template>

<script>
export default {
  data() {
    return {
      selectedLanguagePair: 'enus-zhcn',
      languagePairs: [
        { value: 'enus-zhcn', label: '英语 - 中文' },
        { value: 'zhcn-enus', label: '中文 - 英语' },
        { value: 'zhcn-jajp', label: '中文 - 日语' },
        { value: 'enus-jajp', label: '英语 - 日语' },
        { value: 'jajp-zhcn', label: '日语 - 中文' },
        { value: 'jajp-enus', label: '日语 - 英语' },
      ]
    }
  },
  methods: {
    changeLanguagePair() {
      this.$emit('change-language-pair', this.selectedLanguagePair);
    }
  }
}
</script>

<style scoped>
.language-selector {
  display: flex;
  flex-direction: column;
}

label {
  margin-bottom: 5px;
}

select {
  padding: 10px;
  font-size: 16px;
  border: 1px solid #ccc;
  border-radius: 3px;
  background-color: white;
  cursor: pointer;
}
</style>