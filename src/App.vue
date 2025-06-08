<script setup>
import { ref, onMounted } from 'vue'
import emojiList from './assets/emoji.json'
import Fuse from 'fuse.js'
import CommentComposer from './components/CommentComposer.vue'

const keyword = ref('')
const result = ref([])
const toast = ref({ text: '', visible: false, isError: false })
const mode = ref('emoji')

const categories = ref([])
const selectedCategory = ref('')

onMounted(() => {
  const set = new Set(emojiList.map(e => e.category).filter(Boolean))
  categories.value = [...set]
})

const fuse = new Fuse(emojiList, {
  keys: ['description', 'tags', 'aliases'],
  includeMatches: true,
  threshold: 0.2
})

function searchEmoji() {
  const key = keyword.value.trim()
  if (!key) {
    result.value = []
    selectedCategory.value = ''
    return
  }

  result.value = fuse.search(key)
  selectedCategory.value = ''
}

function filterByCategory(cat) {
  keyword.value = ''
  selectedCategory.value = cat
  result.value = emojiList
    .filter(e => e.category === cat)
    .map(e => ({ item: e, matches: [] }))
}

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

function showToast(text, isError = false) {
  toast.value = { text, visible: true, isError }
  setTimeout(() => toast.value.visible = false, 2000)
}

function copyFormat(format = 'comment') {
  const emojis = result.value.map(r => r.item.emoji).join(' ')
  const text = {
    comment: `// ${emojis} ${keyword.value}`,
    markdown: `> ${emojis} \`${keyword.value}\``,
    html: `<!-- ${emojis} ${keyword.value} -->`
  }[format]

  navigator.clipboard.writeText(text)
    .then(() => showToast(`✅ Copied as ${format}`))
    .catch(() => showToast('❌ Copy failed', true))
}

function copyEmoji(emoji) {
  navigator.clipboard.writeText(emoji)
    .then(() => showToast(`✅ Copied ${emoji}`))
    .catch(() => showToast(`❌ Copy failed`, true))
}

function buttonStyle(active) {
  return {
    margin: '0 4px',
    padding: '4px 12px',
    borderRadius: '6px',
    background: active ? '#007bff' : '#eee',
    color: active ? 'white' : '#333',
    border: '1px solid #ccc'
  }
}
</script>

<template>
  <div style="padding: 2rem; font-family: sans-serif; max-width: 700px; margin: auto;">
    <h1>🔖 EmojiTag</h1>

    <!-- 模式切换按钮 -->
    <div style="margin-bottom: 1rem;">
      <button @click="mode = 'emoji'" :style="buttonStyle(mode === 'emoji')">🧩 Emoji 搜索</button>
      <button @click="mode = 'comment'" :style="buttonStyle(mode === 'comment')">📝 注释推荐</button>
    </div>

    <!-- 搜索框，仅 emoji 模式显示 -->
    <div v-if="mode === 'emoji'">
      <input
        v-model="keyword"
        @keyup.enter="searchEmoji"
        placeholder="请输入关键词..."
        style="padding: 8px; font-size: 16px; width: 60%;"
      />
      <button @click="searchEmoji" style="margin-left: 8px;">Search</button>
    </div>

    <!-- 分类浏览，仅 emoji 模式显示 -->
    <div v-if="categories.length && mode === 'emoji'" style="margin-top: 1rem;">
      <button
        v-for="cat in categories"
        :key="cat"
        @click="filterByCategory(cat)"
        :style="buttonStyle(selectedCategory === cat)"
      >
        {{ cat }}
      </button>
    </div>

    <!-- emoji 模式结果 -->
    <div v-if="mode === 'emoji' && result.length" style="margin-top: 1.5rem;">
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
            title="点击复制 emoji"
          >{{ r.item.emoji }}</span>
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

    <!-- 注释推荐模式 -->
    <CommentComposer v-if="mode === 'comment'" />

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
