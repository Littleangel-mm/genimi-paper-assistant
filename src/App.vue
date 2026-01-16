<script setup>
import { ref, nextTick, onMounted } from 'vue'
import { GoogleGenerativeAI } from '@google/generative-ai'

const language = ref('zh')
const style = ref('academic')
const input = ref('')
const apiKey = ref(import.meta.env.VITE_GEMINI_API_KEY || '')
const modelName = ref(import.meta.env.VITE_GEMINI_MODEL || 'gemini-1.5-flash')
const loading = ref(false)
const error = ref('')
const messages = ref([
  { role: 'assistant', content: '请粘贴需要润色的论文段落，选择语言与风格后发送' }
])
const listEl = ref(null)
const conversations = ref([])
const activeId = ref('')
const isSidebarOpen = ref(true)
const openMenuId = ref('')
const previewAvatarSrc = ref('')

const newId = () => `${Date.now()}_${Math.random().toString(36).slice(2, 8)}`

const persist = () => {
  localStorage.setItem('polish_conversations', JSON.stringify(conversations.value))
  localStorage.setItem('polish_active', activeId.value)
}

const ensureInit = () => {
  const raw = localStorage.getItem('polish_conversations')
  const aid = localStorage.getItem('polish_active')
  if (raw) {
    conversations.value = JSON.parse(raw)
  }
  if (!conversations.value.length) {
    const id = newId()
    conversations.value.push({ id, title: '新的会话', messages: messages.value.slice(), createdAt: Date.now(), updatedAt: Date.now() })
    activeId.value = id
  } else {
    activeId.value = aid || conversations.value[0].id
    const cur = conversations.value.find(x => x.id === activeId.value)
    messages.value = cur ? cur.messages : messages.value
  }
  persist()
}

onMounted(ensureInit)

const setTitleFromFirstUser = conv => {
  const first = conv.messages.find(m => m.role === 'user')
  if (first) {
    const t = first.content.replace(/\s+/g, ' ').slice(0, 20)
    conv.title = t || conv.title || '未命名对话'
  }
}

const selectChat = id => {
  activeId.value = id
  const cur = conversations.value.find(x => x.id === id)
  messages.value = cur ? cur.messages : []
  isSidebarOpen.value = false
  openMenuId.value = ''
  persist()
}

const newChat = () => {
  const id = newId()
  const welcome = { role: 'assistant', content: '本网页致力于论文润色，结果为大模型提供，仅供参考' }
  conversations.value.unshift({ id, title: '新的会话', messages: [welcome], createdAt: Date.now(), updatedAt: Date.now() })
  selectChat(id)
}

const renameChat = id => {
  const cur = conversations.value.find(x => x.id === id)
  if (!cur) return
  const name = prompt('重命名对话', cur.title || '')
  if (name && name.trim()) {
    cur.title = name.trim()
    cur.updatedAt = Date.now()
    persist()
  }
}

const deleteChat = id => {
  const idx = conversations.value.findIndex(x => x.id === id)
  if (idx === -1) return
  conversations.value.splice(idx, 1)
  if (!conversations.value.length) {
    newChat()
    return
  }
  if (activeId.value === id) {
    selectChat(conversations.value[0].id)
  } else {
    persist()
  }
}

const scrollToBottom = async () => {
  await nextTick()
  if (listEl.value) listEl.value.scrollTop = listEl.value.scrollHeight
}

const send = async () => {
  error.value = ''
  const text = input.value.trim()
  if (!text) {
    error.value = '请输入需要润色的文本'
    return
  }
  messages.value.push({ role: 'user', content: text })
  input.value = ''
  loading.value = true
  await scrollToBottom()
  try {
    if (!apiKey.value) {
      throw new Error('请先填写API密钥')
    }
    const genAI = new GoogleGenerativeAI(apiKey.value)
    let instruction = ''
    if (language.value === 'zh') {
      if (style.value === 'concise') instruction = '请将下面的论文段落进行中文润色，要求简洁、通顺、清晰，保持专业术语与含义不变。'
      else if (style.value === 'formal') instruction = '请将下面的论文段落进行中文润色，要求正式、规范、严谨，保持专业术术语与含义不变。'
      else instruction = '请将下面的论文段落进行中文润色，要求学术严谨、清晰、流畅，保持专业术语与含义不变。'
      instruction += ' 同时生成一个50-100字的中文总结，并按“措辞、语法、逻辑、术语”四类分别列出1-4条要点。输出格式严格为：<POLISHED>润色后文本</POLISHED>\\n<SUMMARY>总结</SUMMARY>\\n<EXPLAINS>{\"措辞\":[\"...\"],\"语法\":[\"...\"],\"逻辑\":[\"...\"],\"术语\":[\"...\"]}</EXPLAINS>'
    } else {
      if (style.value === 'concise') instruction = 'Polish the following academic paragraph in English to be concise, clear, and fluent. Preserve technical terms and meaning.'
      else if (style.value === 'formal') instruction = 'Polish the following academic paragraph in English to be formal and rigorous. Preserve technical terms and meaning.'
      else instruction = 'Polish the following academic paragraph in English to be academically rigorous, clear, and fluent. Preserve technical terms and meaning.'
      instruction += ' Also provide a 50-100 word English summary and 1-4 bullet points under each of “Diction, Grammar, Logic, Terminology”. Output strictly as: <POLISHED>polished text</POLISHED>\\n<SUMMARY>summary</SUMMARY>\\n<EXPLAINS>{\"Diction\":[\"...\"],\"Grammar\":[\"...\"],\"Logic\":[\"...\"],\"Terminology\":[\"...\"]}</EXPLAINS>'
    }
    const prompt = `${instruction}\n\n${text}`
    const model = genAI.getGenerativeModel({ model: modelName.value })
    const resp = await model.generateContent(prompt)
    const out = resp.response.text()
    const pMatch = out.match(/<POLISHED>([\s\S]*?)<\/POLISHED>/i)
    const sMatch = out.match(/<SUMMARY>([\s\S]*?)<\/SUMMARY>/i)
    const polished = pMatch ? pMatch[1].trim() : (out || '')
    const summary = sMatch ? sMatch[1].trim() : (language.value === 'zh' ? '本次润色主要优化措辞、修正语法、提升逻辑与清晰度。' : 'Polishing improved diction, fixed grammar, and enhanced clarity and coherence.')
    const eMatch = out.match(/<EXPLAINS>([\s\S]*?)<\/EXPLAINS>/i)
    let explains = null
    try {
      explains = eMatch ? JSON.parse(eMatch[1]) : null
    } catch (_) {
      explains = null
    }
    messages.value.push({ role: 'assistant', content: polished, summary, explains })
    const cur = conversations.value.find(x => x.id === activeId.value)
    if (cur) {
      cur.updatedAt = Date.now()
      setTitleFromFirstUser(cur)
      persist()
    }
  } catch (e) {
    error.value = e.message || '发生错误'
  } finally {
    loading.value = false
    await scrollToBottom()
  }
}

const onKey = e => {
  if (e.key === 'Enter' && !e.shiftKey) {
    e.preventDefault()
    if (!loading.value) send()
  }
}

const clearChat = () => {
  messages.value = [{ role: 'assistant', content: '本网页致力于论文润色，结果为大模型提供，仅供参考' }]
}

const copyAfter = async m => {
  const text = (m && m.content) ? m.content : ''
  try {
    if (navigator.clipboard && navigator.clipboard.writeText) {
      await navigator.clipboard.writeText(text)
    } else {
      const ta = document.createElement('textarea')
      ta.value = text
      document.body.appendChild(ta)
      ta.select()
      document.execCommand('copy')
      document.body.removeChild(ta)
    }
    m._copied = true
    setTimeout(() => { m._copied = false }, 1200)
  } catch (_) {
    m._copied = true
    setTimeout(() => { m._copied = false }, 1200)
  }
}

const normalizeExplains = (explains) => {
  const out = {}
  const pick = (key, label) => {
    const v = explains && Array.isArray(explains[key]) ? explains[key].filter(Boolean) : []
    if (v.length) out[label] = v
  }
  pick('措辞', '措辞')
  pick('语法', '语法')
  pick('逻辑', '逻辑')
  pick('术语', '术语')
  pick('Diction', '措辞')
  pick('Grammar', '语法')
  pick('Logic', '逻辑')
  pick('Terminology', '术语')
  return out
}

const showAvatar = src => {
  previewAvatarSrc.value = src || ''
}
const closePreview = () => {
  previewAvatarSrc.value = ''
}
</script>

<template>
  <div class="layout">
    <!-- 改动过的侧边栏 -->
    <aside class="sidebar" :class="{ open: isSidebarOpen }">
      <div class="sidebar-header">
        <div class="brand">聊天管理</div>
        <button class="new-btn" @click="newChat">
          <span class="plus">+</span> 新建对话
        </button>
      </div>

      <div class="conversation-list">
        <div
          v-for="c in conversations"
          :key="c.id"
          class="conv-item"
          :class="{ active: c.id === activeId }"
          @click="selectChat(c.id)"
        >
          <div class="conv-title">{{ c.title || '新的会话' }}</div>
          
          <button 
            class="dots"
            @click.stop.prevent="openMenuId = openMenuId === c.id ? '' : c.id"
          >⋯</button>

          <div 
            v-if="openMenuId === c.id" 
            class="context-menu"
            @click.stop
          >
            <button class="menu-item" @click="renameChat(c.id)">重命名</button>
            <button class="menu-item danger" @click="deleteChat(c.id)">删除</button>
          </div>
        </div>
      </div>
    </aside>

    <div class="page">
      <header class="header">
        <div class="title">水论文专用小助手</div>
        <div class="toolbar">
          <button class="clear" @click="isSidebarOpen = !isSidebarOpen">会话</button>
          <select v-model="language">
            <option value="zh">中文</option>
            <option value="en">英文</option>
          </select>
          <select v-model="style">
            <option value="academic">学术严谨</option>
            <option value="concise">简洁通顺</option>
            <option value="formal">正式规范</option>
          </select>
          <select v-model="modelName">
            <option value="gemini-1.5-flash">1.5-flash</option>
            <option value="gemini-1.5-pro">1.5-pro</option>
            <option value="gemini-2.5-flash">2.5-flash</option>
          </select>
          <button class="clear" @click="clearChat">清空会话</button>
        </div>
      </header>

      <!-- 改动过的对话消息区域（更接近现代聊天界面风格） -->
      <main class="chat" ref="listEl">
        <div v-for="(m, i) in messages" :key="i" class="message" :class="m.role">
          <div class="avatar">
            <img v-if="m.role === 'assistant'" src="http://xtstuc.dyfl.top/xtsimage/intellligent.jpg" class="avatar-img" alt="AI" @click="showAvatar($event.target.src)" />
            <img v-else src="http://xtstuc.dyfl.top/xtsimage/6a6f6a3beb424eaed75effde133c9b85.png" class="avatar-img" alt="我" @click="showAvatar($event.target.src)" />
          </div>
          <div v-if="m.role !== 'assistant'" class="bubble" v-html="m.content.replace(/\n/g, '<br>')"></div>
          <div v-else class="bubble">
            <div class="compare">
              <div class="panel">
                <div class="panel-title">润色说明</div>
                <div class="panel-body">
                  <div v-if="m.explains" class="explain">
                    <div class="explain-group" v-for="(items, name) in normalizeExplains(m.explains)" :key="name">
                      <div class="explain-title">{{ name }}</div>
                      <ul class="explain-list">
                        <li v-for="(it, idx) in items" :key="idx">{{ it }}</li>
                      </ul>
                    </div>
                  </div>
                  <div v-else v-html="(m.summary || (language==='zh' ? '本次润色主要优化措辞、修正语法、提升逻辑与清晰度。' : 'Polishing improved diction, fixed grammar, and enhanced clarity and coherence.')).replace(/\n/g, '<br>')"></div>
                </div>
              </div>
              <div class="panel">
                <div class="panel-title">润色后</div>
                <div class="panel-body" v-html="m.content.replace(/\n/g, '<br>')"></div>
              </div>
            </div>
            <div class="actions">
              <button class="copy-btn" :class="{ copied: m._copied }" @click="copyAfter(m)">
                {{ m._copied ? '已复制' : '复制润色后' }}
              </button>
            </div>
          </div>
        </div>

        <div v-if="loading" class="message assistant">
          <div class="avatar">
            <img src="http://xtstuc.dyfl.top/xtsimage/intellligent.jpg" class="avatar-img" alt="AI" @click="showAvatar($event.target.src)" />
          </div>
          <div class="bubble typing">正在思考...</div>
        </div>
      </main>

      <footer class="composer">
        <textarea
          v-model="input"
          class="input"
          placeholder="Shift+Enter 换行，Enter 发送"
          @keydown="onKey"
        />
        <button class="send" :disabled="loading || !input.trim()" @click="send">
          {{ loading ? '处理中' : '发送' }}
        </button>
      </footer>

      <div v-if="error" class="error">{{ error }}</div>

      <div v-if="previewAvatarSrc" class="img-preview" @click="closePreview">
        <img :src="previewAvatarSrc" @click.stop />
      </div>
    </div>
  </div>
</template>

<style scoped>
.layout {
  display: grid;
  grid-template-columns: 260px 1fr;
  height: 100vh;
  height: 100svh;
}

.sidebar {
  background: #0f172a;
  color: #e5e7eb;
  display: flex;
  flex-direction: column;
  border-right: 1px solid #1e293b;
  transition: transform 0.3s ease;
  position: fixed;
  left: 0;
  top: 0;
  bottom: 0;
  width: 260px;
  z-index: 40;
  transform: translateX(-100%);
}

.sidebar.open {
  transform: translateX(0);
}

@media (min-width: 768px) {
  .sidebar {
    position: relative;
    transform: translateX(0);
  }
}

.sidebar-header {
  padding: 16px;
  border-bottom: 1px solid #1e293b;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.brand {
  font-weight: 600;
  font-size: 1.1rem;
}

.new-btn {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 6px 12px;
  background: #1e293b;
  border: 1px solid #334155;
  border-radius: 8px;
  color: #e5e7eb;
  font-size: 0.9rem;
}

.new-btn:hover {
  background: #253549;
}

.plus {
  font-size: 1.2rem;
  line-height: 1;
}

.conversation-list {
  flex: 1;
  overflow-y: auto;
  padding: 8px;
}

.conv-item {
  position: relative;
  padding: 10px 12px;
  margin-bottom: 4px;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.15s;
}

.conv-item:hover {
  background: #1e293b;
}

.conv-item.active {
  background: #334155;
}

.conv-title {
  font-size: 0.94rem;
  line-height: 1.4;
  padding-right: 32px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.dots {
  position: absolute;
  right: 8px;
  top: 50%;
  transform: translateY(-50%);
  width: 28px;
  height: 28px;
  border-radius: 6px;
  background: transparent;
  border: none;
  color: #94a3b8;
  font-size: 1.3rem;
  cursor: pointer;
  opacity: 0;
  transition: all 0.15s;
}

.conv-item:hover .dots,
.conv-item.active .dots {
  opacity: 0.7;
}

.dots:hover {
  opacity: 1 !important;
  background: #253549;
}

.context-menu {
  position: absolute;
  top: 100%;
  right: 4px;
  background: #1e293b;
  border: 1px solid #334155;
  border-radius: 8px;
  padding: 6px 0;
  min-width: 110px;
  box-shadow: 0 8px 20px rgba(0,0,0,0.4);
  z-index: 50;
}

.menu-item {
  display: block;
  width: 100%;
  padding: 8px 14px;
  text-align: left;
  background: none;
  border: none;
  color: #e5e7eb;
  font-size: 0.9rem;
  cursor: pointer;
}

.menu-item:hover {
  background: #253549;
}

.menu-item.danger {
  color: #fca5a5;
}

/* ── 改动后的消息气泡样式 ── */
.chat {
  padding: 16px 20px;
  overflow-y: auto;
  background: #f8fafc;
}

.message {
  display: flex;
  margin-bottom: 20px;
  gap: 12px;
  max-width: 85%;
}

.message.assistant {
  margin-right: auto;
}

.message.user {
  margin-left: auto;
  flex-direction: row-reverse;
}

.avatar {
  width: 38px;
  height: 38px;
  border-radius: 50%;
  background: #d1d5db;
  color: #111827;
  font-size: 14px;
  font-weight: 600;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  overflow: hidden;
}

.message.user .avatar {
  background: #3b82f6;
  color: white;
}

.avatar-img {
  width: 100%;
  height: 100%;
  border-radius: 50%;
  object-fit: cover;
  display: block;
}
.img-preview {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.6);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
}
.img-preview img {
  max-width: 88vw;
  max-height: 88vh;
  border-radius: 12px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.35);
}

.bubble {
  padding: 12px 16px;
  border-radius: 18px;
  line-height: 1.55;
  background: white;
  border: 1px solid #e2e8f0;
  box-shadow: 0 1px 3px rgba(0,0,0,0.06);
  white-space: pre-wrap;
  word-break: break-word;
}

.message.user .bubble {
  background: #dbeafe;
  border-color: #bfdbfe;
  border-radius: 18px 18px 4px 18px;
}

.message.assistant .bubble {
  border-radius: 18px 18px 18px 4px;
}

.typing {
  background: #f1f5f9;
  color: #64748b;
}

.compare {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
}

.panel {
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  background: #fff;
}

.panel-title {
  font-size: 13px;
  color: #64748b;
  padding: 8px 12px;
  border-bottom: 1px solid #e2e8f0;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.panel-body {
  padding: 12px;
  white-space: pre-wrap;
  word-break: break-word;
}

.explain-group {
  margin-bottom: 8px;
}
.explain-title {
  font-size: 12px;
  color: #64748b;
  margin-bottom: 4px;
}
.explain-list {
  margin: 0;
  padding-left: 18px;
}
.explain-list li {
  margin-bottom: 4px;
}

.actions {
  display: flex;
  justify-content: flex-end;
  margin-top: 10px;
}

.copy-btn {
  padding: 6px 12px;
  border: none;
  border-radius: 10px;
  background: #3b82f6;
  color: #fff;
  font-size: 14px;
  cursor: pointer;
  transition: background 0.15s;
}
.copy-btn:hover {
  background: #2563eb;
}
.copy-btn.copied {
  background: #16a34a;
}
.icon-btn {
  width: 32px;
  height: 32px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  color: #64748b;
  transition: background 0.15s, color 0.15s;
}
.icon-btn:hover {
  background: #f1f5f9;
  color: #3b82f6;
}
.icon-btn svg {
  width: 18px;
  height: 18px;
  fill: currentColor;
}
.copy-icon.copied {
  color: #16a34a;
}
.title-copy {
  margin-left: 8px;
}

@media (max-width: 768px) {
  .compare {
    grid-template-columns: 1fr;
  }
}
/* 以下部分保持原样不变 */
.page {
  display: grid;
  grid-template-rows: auto 1fr auto;
  height: 100vh;
  height: 100svh;
  background: #f5f7fb;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background: #ffffff;
  border-bottom: 1px solid #eaecef;
}

.title {
  font-size: 16px;
  font-weight: 600;
}

.toolbar {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

select {
  padding: 8px;
  border: 1px solid #e1e4ea;
  border-radius: 8px;
  background: #fff;
}

.clear {
  padding: 8px 12px;
  border: 1px solid #e1e4ea;
  border-radius: 8px;
  background: #fff;
  cursor: pointer;
}

.composer {
  display: flex;
  align-items: flex-end;
  gap: 12px;
  padding: 16px 20px;
  background: #ffffff;
  border-top: 1px solid #eaecef;
  --input-min: 52px;
  --input-py: 14px;
  --input-px: 16px;
  --radius: 16px;
}

.input {
  flex: 1;
  min-height: var(--input-min);
  max-height: 160px;
  padding: var(--input-py) var(--input-px);
  border: 1px solid #d1d5db;
  border-radius: var(--radius);
  font-size: 15px;
  line-height: 1.5;
  resize: vertical;
  transition: border-color 0.2s;
  background: #fff;
}

.input:focus {
  border-color: #3b82f6;
  outline: none;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.15);
}

.send {
  flex-shrink: 0;
  padding: 0 28px;
  height: calc(var(--input-min) + var(--input-py) * 2);
  border: none;
  border-radius: var(--radius);
  background: #3b82f6;
  color: white;
  font-size: 15px;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.2s;
}

.send:hover:not(:disabled) {
  background: #2563eb;
}

.send:disabled {
  background: #93c5fd;
  cursor: not-allowed;
}

.error {
  position: fixed;
  left: 50%;
  transform: translateX(-50%);
  bottom: 80px;
  background: #fee2e2;
  color: #b91c1c;
  border: 1px solid #fecaca;
  border-radius: 8px;
  padding: 8px 12px;
}
</style>
