<script setup>
import { ref, onMounted } from 'vue'
import emojiList from './assets/emoji.json'
import Fuse from 'fuse.js'

const keyword = ref('')
const result = ref([])
const toast = ref({ text: '', visible: false, isError: false })

// 分类浏览
const categories = ref([])
const selectedCategory = ref('')

// 初始化分类
onMounted(() => {
  const set = new Set(emojiList.map(e => e.category).filter(Boolean))
  categories.value = [...set]
})

const fuse = new Fuse(emojiList, {
  keys: ['description', 'tags', 'aliases'],
  includeMatches: true,
  threshold: 0.1
})

function searchEmoji() {
  const key = keyword.value.trim()
  if (!key) {
    result.value = []
    selectedCategory.value = ''
    return
  }
  selectedCategory.value = ''
  result.value = fuse.search(key)
}

// 分类筛选
function filterByCategory(cat) {
  keyword.value = ''
  selectedCategory.value = cat
  result.value = emojiList
    .filter(e => e.category === cat)
    .map(e => ({ item: e, matches: [] }))
}

// 高亮匹配
function highlight(text, matches) {
  if (!matches?.length) return text
  let ranges = matches.flatMap(m => m.indices)
  ranges.sort((a, b) => a[0] - b[0])
  let result = ''
  let lastIndex = 0
  for (const [start, end] of ranges) {
    result += text.slice(lastIndex, start)
    result += `<mark>${text.slice(start, end + 1)}</mark>`
    lastIndex = end + 1
  }
  result += text.slice(lastIndex)
  return result
}

// Toast 提示
function showToast(text, isError = false) {
  toast.value = { text, visible: true, isError }
  setTimeout(() => toast.value.visible = false, 2000)
}

// 复制注释/markdown/html
function copyFormat(format = 'comment') {
  const emojis = result.value.map(r => r.item.emoji).join(' ')
  const content = {
    comment: `// ${emojis} ${keyword.value}`,
    markdown: `> ${emojis} \`${keyword.value}\``,
    html: `<!-- ${emojis} ${keyword.value} -->`
  }[format]
  navigator.clipboard.writeText(content)
    .then(() => showToast(`✅ Copied as ${format}`))
    .catch(() => showToast('❌ Copy failed', true))
}

// 单个 emoji 复制
function copyEmoji(emoji) {
  navigator.clipboard.writeText(emoji)
    .then(() => showToast(`✅ Copied ${emoji}`))
    .catch(() => showToast(`❌ Copy failed`, true))
}
</script>

<template>
  <div style="padding: 2rem; font-family: sans-serif; max-width: 700px; margin: auto;">
    <h1>🔖 EmojiTag</h1>

    <!-- 搜索框 -->
    <div>
      <input
        v-model="keyword"
        @keyup.enter="searchEmoji"
        placeholder="Enter a keyword..."
        style="padding: 8px; font-size: 16px; width: 60%;"
      />
      <button @click="searchEmoji" style="margin-left: 8px;">Search</button>
    </div>

    <!-- 分类浏览 -->
    <div v-if="categories.length" style="margin-top: 1rem;">
      <p>📂 Browse by category:</p>
      <button
        v-for="cat in categories"
        :key="cat"
        @click="filterByCategory(cat)"
        :style="{
          margin: '4px',
          padding: '4px 8px',
          background: selectedCategory === cat ? '#007bff' : '#f4f4f4',
          color: selectedCategory === cat ? 'white' : '#333',
          borderRadius: '6px',
          border: '1px solid #ccc',
          cursor: 'pointer'
        }"
      >
        {{ cat }}
      </button>
    </div>

    <!-- 搜索或分类结果 -->
    <div v-if="result.length" style="margin-top: 1.5rem;">
      <p style="font-size: 1.2rem; font-weight: bold;">🎯 Results:</p>
      <ul style="list-style: none; padding: 0;">
        <li
          v-for="r in result"
          :key="r.item.emoji + r.item.description"
          style="margin: 0.5rem 0;"
        >
          <span
            style="cursor: pointer; font-size: 1.8rem;"
            @click="copyEmoji(r.item.emoji)"
            title="Click to copy"
          >
            {{ r.item.emoji }}
          </span>
          &nbsp;
          <span
            v-html="highlight(
              r.item.description || '',
              r.matches?.filter(m => m.key === 'description')
            )"
          />
        </li>
      </ul>

      <!-- 复制按钮组 -->
      <div style="margin-top: 1rem;">
        <button @click="copyFormat('comment')">📋 Copy as Comment</button>
        <button @click="copyFormat('markdown')" style="margin-left: 8px;">📝 Markdown</button>
        <button @click="copyFormat('html')" style="margin-left: 8px;">🌐 HTML</button>
      </div>
    </div>

    <!-- Toast 提示 -->
    <div v-if="toast.visible" :class="['toast', toast.isError ? 'error' : '']">
      {{ toast.text }}
    </div>
  </div>
</template>

<style scoped>
mark {
  background-color: yellow;
  padding: 0 2px;
  border-radius: 3px;
}
.toast {
  position: fixed;
  bottom: 30px;
  left: 50%;
  transform: translateX(-50%);
  background: #444;
  color: #fff;
  padding: 10px 16px;
  border-radius: 8px;
  font-size: 14px;
  z-index: 1000;
  transition: opacity 0.3s;
}
.toast.error {
  background: darkred;
}
</style>
