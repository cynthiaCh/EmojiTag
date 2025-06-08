<script setup>
import { ref } from 'vue'
import Fuse from 'fuse.js'
import commentSamples from '../assets/comment-samples-large.json'

const keyword = ref('')
const suggestions = ref([])
const selected = ref('')
const format = ref('js-single')
const copied = ref(false)

// 初始化 Fuse 搜索引擎
const fuse = new Fuse(commentSamples, {
  keys: [
    { name: 'keyword', weight: 0.5 },
    { name: 'description', weight: 0.3 },
    { name: 'tags', weight: 0.2 }
  ],
  threshold: 0.3,
  includeMatches: true,
  minMatchCharLength: 1
})

function getSuggestions() {
  const word = keyword.value.trim()
  if (!word) return suggestions.value = []
  const results = fuse.search(word).slice(0, 5)
  suggestions.value = results.map(r => r.item)
}

function formatComment(emoji, comment) {
  const text = `${emoji} ${comment}`
  switch (format.value) {
    case 'js-single': return `// ${text}`
    case 'js-multi': return `/**\n * ${text}\n */`
    case 'python': return `# ${text}`
    case 'html': return `<!-- ${text} -->`
    default: return text
  }
}

function copy(comment) {
  navigator.clipboard.writeText(comment).then(() => {
    copied.value = true
    setTimeout(() => copied.value = false, 1500)
  })
}
</script>

<template>
  <div style="max-width: 600px; margin: 2rem auto; font-family: sans-serif;">
    <input
      v-model="keyword"
      @input="getSuggestions"
      placeholder="请输入关键词（如 删除 / 缓存 / 请求）"
      style="padding: 8px; font-size: 16px; width: 100%;"
    />

    <div v-if="suggestions.length" style="margin-top: 1rem;">
      <div v-for="s in suggestions" :key="s.id" style="margin-bottom: 1rem;">
        <div style="font-size: 1.5rem; cursor: pointer;" @click="selected = formatComment(s.emoji, s.comment)">
          {{ s.emoji }} {{ s.comment }}
        </div>
        <div style="font-size: 0.9rem; color: #666; margin-left: 2rem;">
          {{ s.description }}
        </div>
      </div>
    </div>

    <div v-if="selected" style="margin-top: 2rem;">
      <label style="font-weight: bold; display: block; margin-bottom: 0.5rem;">🧱 注释格式：</label>
      <select v-model="format" @change="selected = formatComment(selected.split(' ').shift(), selected.split(' ').slice(1).join(' '))">
        <option value="js-single">JavaScript 单行 // 注释</option>
        <option value="js-multi">JavaScript 多行 /** */</option>
        <option value="python">Python # 注释</option>
        <option value="html">HTML <!-- 注释 --></option>
      </select>

      <pre style="background: #f5f5f5; color: #222; padding: 1rem; border-radius: 6px; margin-top: 1rem;">{{ selected }}</pre>
      <button @click="copy(selected)" style="margin-top: 0.5rem;">📋 复制注释</button>
      <span v-if="copied" style="margin-left: 10px; color: green;">✅ 已复制</span>
    </div>
  </div>
</template>

<style scoped>
select {
  padding: 6px;
  font-size: 14px;
  margin-bottom: 1rem;
}
</style>
