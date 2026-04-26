<script setup>
import { ref, nextTick, onMounted, onBeforeUnmount, computed } from 'vue'
import { GoogleGenerativeAI } from '@google/generative-ai'
import { encode } from 'plantuml-encoder'

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
const previewAvatarSrc = ref('')
const currentPage = ref('chat')
const pumlInput = ref(`@startuml
Alice -> Bob: hello
Bob --> Alice: hi
@enduml`)
const actionDialogId = ref('')
const renameDialogOpen = ref(false)
const deleteDialogOpen = ref(false)
const renameDraft = ref('')
const popoverPosition = ref({ top: 0, left: 0 })

const diagramPatterns = [
  { name: 'ER图', category: 'ER', checks: [/\bentity\b/i, /\|\|--|o\{--|::/i] },
  { name: '类图', category: '结构图', checks: [/\bclass\b/i, /--\|>|<\|--|--\*/i] },
  { name: '对象图', category: '结构图', checks: [/\bobject\b/i, /:\s*[A-Za-z_]\w*/] },
  { name: '包图', category: '结构图', checks: [/\bpackage\b/i] },
  { name: '组件图', category: '结构图', checks: [/\bcomponent\b/i, /\[[^\]]+\]\s*--/i] },
  { name: '部署图', category: '结构图', checks: [/\bnode\b/i, /\bartifact\b/i] },
  { name: '复合结构图', category: '结构图', checks: [/\bframe\b/i, /\bport\b/i] },
  { name: '剖面图', category: '结构图', checks: [/\bstereotype\b/i, /<<\s*profile\s*>>/i] },
  { name: '用例图', category: '行为图', checks: [/\busecase\b/i, /:\w+\s*-->\s*\(\w+/i] },
  { name: '活动图', category: '行为图', checks: [/\bstart\b/i, /if\s*\(|-->.*:/i] },
  { name: '状态机图', category: '行为图', checks: [/\[\*\]\s*-->/, /\bstate\b/i] },
  { name: '时序图', category: '行为图', checks: [/->|-->/, /\bparticipant\b|\bactor\b/i] },
  { name: '序列图', category: '行为图', checks: [/\bautonumber\b/i, /\bactivate\b|\bdeactivate\b/i] },
  { name: '通信图', category: '行为图', checks: [/\bparticipant\b/i, /\b:\s*\d+\s*:/] },
  { name: '交互概览图', category: '行为图', checks: [/\bpartition\b/i, /\brepeat\b/i] },
  { name: '定时图', category: '行为图', checks: [/\brobust\b|\bconcise\b/i, /@[\dms]+/] }
]

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

const handleWindowClick = (event) => {
  const target = event.target
  if (!target) return
  if (target.closest('.action-popover') || target.closest('.dots')) return
  closeDialogs()
}

onMounted(() => {
  ensureInit()
  window.addEventListener('click', handleWindowClick)
})

onBeforeUnmount(() => {
  window.removeEventListener('click', handleWindowClick)
})

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
  persist()
}

const newChat = () => {
  const id = newId()
  const welcome = { role: 'assistant', content: '本网页致力于论文润色，结果为大模型提供，仅供参考' }
  conversations.value.unshift({ id, title: '新的会话', messages: [welcome], createdAt: Date.now(), updatedAt: Date.now() })
  selectChat(id)
}

const openConversationMenu = (id, event) => {
  actionDialogId.value = id
  const rect = event?.currentTarget?.getBoundingClientRect?.()
  if (rect) {
    const width = 220
    const gap = 6
    const margin = 8
    let left = rect.right - width
    let top = rect.bottom + gap
    left = Math.max(margin, Math.min(left, window.innerWidth - width - margin))
    if (top + 220 > window.innerHeight) {
      top = Math.max(margin, rect.top - 220 - gap)
    }
    popoverPosition.value = { top, left }
  }
}

const closeDialogs = () => {
  actionDialogId.value = ''
  renameDialogOpen.value = false
  deleteDialogOpen.value = false
}

const openRenameDialog = () => {
  const cur = conversations.value.find(x => x.id === actionDialogId.value)
  if (!cur) return
  renameDraft.value = cur.title || ''
  renameDialogOpen.value = true
}

const confirmRename = () => {
  const id = actionDialogId.value
  if (!id) return
  const cur = conversations.value.find(x => x.id === id)
  if (!cur) return
  const name = renameDraft.value.trim()
  if (!name) return
  cur.title = name
  cur.updatedAt = Date.now()
  closeDialogs()
  persist()
}

const openDeleteDialog = () => {
  deleteDialogOpen.value = true
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
  closeDialogs()
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

const confirmDelete = () => {
  if (!actionDialogId.value) return
  deleteChat(actionDialogId.value)
}

const detectDiagram = (puml) => {
  const text = (puml || '').trim()
  if (!text) return { name: '未识别', category: '未分类' }
  for (const pattern of diagramPatterns) {
    if (pattern.checks.some((rule) => rule.test(text))) {
      return { name: pattern.name, category: pattern.category }
    }
  }
  return { name: '通用UML图', category: '待确认' }
}

const detectedDiagram = computed(() => detectDiagram(pumlInput.value))

const umlImageUrl = computed(() => {
  const text = pumlInput.value.trim()
  if (!text) return ''
  try {
    return `https://www.plantuml.com/plantuml/svg/${encode(text)}`
  } catch (_) {
    return ''
  }
})
</script>

<template>
  <div class="theme-bg">
    <div class="layout">
      <aside class="sidebar" :class="{ open: isSidebarOpen }">
        <div class="assistant-card">
          <div class="assistant-avatar">✨</div>
          <div class="assistant-meta">
            <div class="assistant-title">论文专用小助手</div>
            <div class="assistant-subtitle">学术写作 · 润色优化 · 语言提升</div>
          </div>
        </div>
        <button class="new-btn" @click="newChat">+ 新建对话</button>
        <button class="uml-entry-btn" :class="{ active: currentPage === 'uml' }" @click="currentPage = 'uml'">UML类图生成</button>
        <div class="conversation-list">
          <div
            v-for="c in conversations"
            :key="c.id"
            class="conv-item"
            :class="{ active: c.id === activeId }"
            @click="selectChat(c.id); currentPage = 'chat'"
          >
            <span class="conv-dot"></span>
            <div class="conv-title">{{ c.title || '新的会话' }}</div>
            <button class="dots" @click.stop.prevent="openConversationMenu(c.id, $event)">⋯</button>
          </div>
        </div>
        <Teleport to="body">
          <div
            v-if="actionDialogId && !renameDialogOpen && !deleteDialogOpen"
            class="dialog-card action-popover action-dialog"
            :style="{ top: `${popoverPosition.top}px`, left: `${popoverPosition.left}px` }"
            @click.stop
          >
            <div class="dialog-title">对话操作</div>
            <button class="dialog-btn" @click="openRenameDialog">重命名对话</button>
            <button class="dialog-btn danger" @click="openDeleteDialog">删除对话</button>
            <button class="dialog-btn ghost" @click="closeDialogs">取消</button>
          </div>
          <div
            v-if="actionDialogId && renameDialogOpen"
            class="dialog-card action-popover"
            :style="{ top: `${popoverPosition.top}px`, left: `${popoverPosition.left}px` }"
            @click.stop
          >
            <div class="dialog-title">重命名对话</div>
            <input v-model="renameDraft" class="dialog-input" placeholder="请输入新的对话名称" />
            <div class="dialog-actions">
              <button class="dialog-btn ghost" @click="closeDialogs">取消</button>
              <button class="dialog-btn primary" :disabled="!renameDraft.trim()" @click="confirmRename">保存</button>
            </div>
          </div>
          <div
            v-if="actionDialogId && deleteDialogOpen"
            class="dialog-card action-popover"
            :style="{ top: `${popoverPosition.top}px`, left: `${popoverPosition.left}px` }"
            @click.stop
          >
            <div class="dialog-title">删除确认</div>
            <p class="dialog-desc">删除后不可恢复，确定删除该对话吗？</p>
            <div class="dialog-actions">
              <button class="dialog-btn ghost" @click="closeDialogs">取消</button>
              <button class="dialog-btn danger" @click="confirmDelete">确认删除</button>
            </div>
          </div>
        </Teleport>
        <div class="side-foot">学术写作更轻松</div>
      </aside>

      <div class="page">
        <template v-if="currentPage === 'chat'">
        <header class="header">
          <div class="welcome">
            <h1>欢迎回来！我是你的论文专用小助手 ✨</h1>
            <p>专注学术写作优化，助你提升论文质量与语言表达</p>
          </div>
          <div class="toolbar">
            <label class="pill">
              <span>会话语言</span>
              <select v-model="language">
                <option value="zh">中文</option>
                <option value="en">英文</option>
              </select>
            </label>
            <label class="pill">
              <span>学术水平</span>
              <select v-model="style">
                <option value="academic">学术严谨</option>
                <option value="concise">简洁通顺</option>
                <option value="formal">正式规范</option>
              </select>
            </label>
            <label class="pill">
              <span>模型版本</span>
              <select v-model="modelName">
                <option value="gemini-1.5-flash">1.5-flash</option>
                <option value="gemini-1.5-pro">1.5-pro</option>
                <option value="gemini-2.5-flash">2.5-flash</option>
              </select>
            </label>
            <button class="pill clear-chat" @click="clearChat">清空会话</button>
          </div>
        </header>

        <main class="chat" ref="listEl">
          <div class="hero-card" v-if="!messages.length || messages.length === 1">
            <div class="hero-content">
              <div class="hero-grid">
                <div class="panel">
                  <div class="panel-title">润色说明</div>
                  <div class="panel-body">本次润色主要优化措辞、修正语法、提升逻辑与清晰度。</div>
                </div>
                <div class="panel">
                  <div class="panel-title">润色后</div>
                  <div class="panel-body">请粘贴需要润色的论文段落，选择语言与风格后发送。</div>
                </div>
              </div>
            </div>
            <div class="actions">
              <button class="copy-btn">复制润色后</button>
            </div>
          </div>

          <div v-for="(m, i) in messages" :key="i" class="message" :class="m.role">
            <div v-if="m.role !== 'assistant'" class="bubble user-bubble" v-html="m.content.replace(/\n/g, '<br>')"></div>
            <div v-else class="simple-assistant">
              <div class="bubble ai-bubble" v-html="m.content.replace(/\n/g, '<br>')"></div>
              <div class="mini-actions">
                <button class="mini-copy-btn" :class="{ copied: m._copied }" @click="copyAfter(m)">{{ m._copied ? '已复制' : '复制' }}</button>
              </div>
            </div>
          </div>

          <div v-if="loading" class="message assistant">
            <div class="bubble ai-bubble typing">正在思考...</div>
          </div>
        </main>

        <footer class="composer">
          <div class="composer-tools">
            <button class="tool-btn">📎</button>
            <button class="tool-btn">T</button>
            <button class="tool-btn">✦</button>
          </div>
          <textarea
            v-model="input"
            class="input"
            placeholder="在这里输入或粘贴你需要润色的内容..."
            @keydown="onKey"
          />
          <button class="send" :disabled="loading || !input.trim()" @click="send">➤</button>
        </footer>

        <div v-if="error" class="error">{{ error }}</div>
        <div v-if="previewAvatarSrc" class="img-preview" @click="closePreview">
          <img :src="previewAvatarSrc" @click.stop />
        </div>
        </template>

        <template v-else>
          <header class="header uml-header">
            <div class="welcome">
              <h1>UML / ER 图生成器</h1>
              <p>左侧输入 PUML，右侧实时生成图并自动识别图类型</p>
            </div>
            <div class="uml-type-chip">{{ detectedDiagram.category }} · {{ detectedDiagram.name }}</div>
          </header>
          <main class="uml-main">
            <section class="uml-editor-card">
              <div class="uml-block-title">PUML 输入</div>
              <textarea
                v-model="pumlInput"
                class="uml-editor"
                placeholder="请输入 @startuml ... @enduml 的 PUML 语句"
              />
            </section>
            <section class="uml-preview-card">
              <div class="uml-block-title">图形预览</div>
              <div class="uml-preview">
                <img v-if="umlImageUrl" :src="umlImageUrl" alt="UML Preview" />
                <div v-else class="uml-empty">请输入有效的 PUML 语句</div>
              </div>
            </section>
          </main>
        </template>
      </div>
    </div>
  </div>
</template>

<style scoped>
.theme-bg {
  min-height: 100vh;
  min-height: 100svh;
  padding: 18px;
  background: url('/theme-bg.png') center/cover no-repeat fixed;
}

.layout {
  display: grid;
  grid-template-columns: 265px 1fr;
  gap: 14px;
  height: calc(100vh - 36px);
  height: calc(100svh - 36px);
}

.sidebar {
  border-radius: 20px;
  padding: 16px;
  color: #5f5682;
  background: rgba(255, 255, 255, 0.72);
  border: 1px solid rgba(255, 255, 255, 0.75);
  box-shadow: 0 10px 28px rgba(183, 144, 208, 0.15);
  backdrop-filter: blur(10px);
  display: flex;
  flex-direction: column;
  min-height: 0;
}

.assistant-card {
  display: flex;
  gap: 10px;
  align-items: center;
}

.assistant-avatar {
  width: 42px;
  height: 42px;
  border-radius: 50%;
  display: grid;
  place-items: center;
  background: linear-gradient(145deg, #f7bce7, #d7d8ff);
  box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.65);
}

.assistant-title {
  font-size: 15px;
  font-weight: 700;
  color: #5d4d8b;
}

.assistant-subtitle {
  margin-top: 2px;
  font-size: 11px;
  color: #968fb1;
}

.new-btn {
  margin: 18px 0 12px;
  width: 100%;
  border: 1px solid #ecd4e8;
  border-radius: 11px;
  background: rgba(255, 255, 255, 0.82);
  color: #7e6897;
  font-size: 14px;
  font-weight: 600;
  padding: 11px;
}

.uml-entry-btn {
  width: 100%;
  border: 1px solid #ecd4e8;
  border-radius: 11px;
  background: rgba(255, 255, 255, 0.82);
  color: #7e6897;
  font-size: 14px;
  font-weight: 600;
  padding: 10px;
  margin-bottom: 12px;
}

.uml-entry-btn.active {
  background: rgba(255, 219, 239, 0.76);
  color: #9c4c77;
}

.conversation-list {
  overflow: auto;
  min-height: 0;
}

.conv-item {
  position: relative;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 11px 10px;
  border-radius: 12px;
  margin-bottom: 6px;
  color: #6d6690;
  cursor: pointer;
}

.conv-item.active,
.conv-item:hover {
  background: rgba(245, 218, 236, 0.65);
}

.conv-dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background: #cb9ef1;
  flex: 0 0 auto;
}

.conv-title {
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-size: 14px;
}

.dots {
  border: none;
  background: transparent;
  color: #a499c9;
  padding: 0 4px;
  font-size: 18px;
}

.side-foot {
  margin-top: auto;
  border-radius: 14px;
  background: rgba(255, 255, 255, 0.78);
  border: 1px solid rgba(238, 207, 233, 0.8);
  padding: 11px 12px;
  color: #756d9b;
  font-size: 13px;
  text-align: center;
}

.page {
  min-width: 0;
  border-radius: 20px;
  background: rgba(255, 255, 255, 0.5);
  border: 1px solid rgba(255, 255, 255, 0.72);
  backdrop-filter: blur(7px);
  box-shadow: 0 12px 30px rgba(185, 156, 201, 0.14);
  display: grid;
  grid-template-rows: auto 1fr auto;
  overflow: hidden;
}

.header {
  padding: 14px 22px 10px;
}

.welcome h1 {
  margin: 0;
  font-size: 35px;
  line-height: 1.1;
  color: #4f427d;
}

.welcome p {
  margin: 10px 0 0;
  font-size: 20px;
  color: #877daa;
}

.toolbar {
  margin-top: 10px;
  display: flex;
  justify-content: flex-end;
  gap: 8px;
}

.pill {
  display: flex;
  align-items: center;
  gap: 8px;
  border-radius: 12px;
  border: 1px solid #ead7ee;
  background: rgba(255, 255, 255, 0.85);
  padding: 9px 12px;
  color: #6f648f;
  font-size: 14px;
}

.pill select {
  border: none;
  background: transparent;
  color: #5b4d84;
  font-size: 13px;
  outline: none;
}

.clear-chat {
  color: #b14f7f;
  border-color: #f0d7e6;
}

.chat {
  padding: 0 22px 14px;
  overflow: auto;
  position: relative;
}

.hero-card {
  border-radius: 20px;
  background: rgba(255, 255, 255, 0.7);
  border: 1px solid #edd8ea;
  padding: 16px 16px 12px;
  margin-bottom: 14px;
  min-height: 320px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  width: min(68%, 820px);
}

.hero-content {
  width: 100%;
}

.hero-grid,
.compare {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
}

.message {
  margin-bottom: 12px;
}

.bubble {
  border-radius: 16px;
  border: 1px solid #edd8ea;
  background: rgba(255, 255, 255, 0.8);
  padding: 12px;
}

.user-bubble {
  max-width: 72%;
  margin-left: auto;
  background: rgba(255, 240, 250, 0.86);
}

.ai-bubble {
  max-width: 72%;
  background: rgba(255, 255, 255, 0.9);
  color: #5b5380;
}

.simple-assistant {
  max-width: 72%;
}

.mini-actions {
  margin-top: 6px;
}

.mini-copy-btn {
  border: none;
  background: transparent;
  color: #a177ad;
  font-size: 12px;
  padding: 0;
  cursor: pointer;
}

.mini-copy-btn.copied {
  color: #5ea46b;
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

.typing {
  color: #857b9f;
}

.panel {
  border: 1px solid #ecd9eb;
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.85);
}

.panel-title {
  font-size: 14px;
  color: #8a6eac;
  padding: 8px 12px;
  border-bottom: 1px solid #f0dded;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.panel-body {
  padding: 12px;
  white-space: pre-wrap;
  word-break: break-word;
  color: #615988;
}

.explain-group {
  margin-bottom: 8px;
}
.explain-title {
  font-size: 12px;
  color: #8f86ad;
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
  justify-content: center;
  margin-top: 12px;
}

.copy-btn {
  padding: 8px 18px;
  border: none;
  border-radius: 999px;
  background: linear-gradient(90deg, #ff98ce, #ea79ba);
  color: #fff;
  font-size: 13px;
  cursor: pointer;
  box-shadow: 0 6px 16px rgba(229, 116, 169, 0.3);
}

.copy-btn.copied {
  background: linear-gradient(90deg, #87c683, #58a863);
}

.clear {
  padding: 8px 12px;
  border: 1px solid #edd8ea;
  border-radius: 8px;
  background: #fff;
  cursor: pointer;
}

.composer {
  margin: 0 22px 22px;
  border-radius: 18px;
  border: 1px solid #efdced;
  background: rgba(255, 255, 255, 0.86);
  padding: 12px;
  display: grid;
  grid-template-columns: auto 1fr auto;
  gap: 12px;
  align-items: end;
}

.composer-tools {
  display: flex;
  gap: 8px;
}

.tool-btn {
  width: 36px;
  height: 36px;
  border-radius: 11px;
  border: 1px solid #ecd6ea;
  background: #fff;
  color: #8c84ad;
  padding: 0;
  font-size: 16px;
  display: grid;
  place-items: center;
}

.input {
  min-height: 92px;
  max-height: 180px;
  padding: 10px 12px;
  border: none;
  font-size: 14px;
  line-height: 1.5;
  resize: vertical;
  background: transparent;
  color: #5c517f;
}

.input:focus {
  outline: none;
}

.send {
  width: 52px;
  height: 52px;
  border: none;
  border-radius: 16px;
  background: linear-gradient(145deg, #ff8fc8, #f27dbc);
  color: white;
  font-size: 20px;
  display: grid;
  place-items: center;
}

.send:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.uml-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.uml-type-chip {
  border-radius: 999px;
  padding: 8px 12px;
  background: rgba(255, 255, 255, 0.85);
  border: 1px solid #ecd8eb;
  color: #7f628f;
  font-size: 13px;
  white-space: nowrap;
}

.uml-main {
  padding: 0 22px 22px;
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  min-height: 0;
}

.uml-editor-card,
.uml-preview-card {
  border-radius: 16px;
  border: 1px solid #ecd8ea;
  background: rgba(255, 255, 255, 0.82);
  padding: 12px;
  display: flex;
  flex-direction: column;
  min-height: 0;
}

.uml-block-title {
  font-size: 14px;
  font-weight: 600;
  color: #6f5a8b;
  margin-bottom: 10px;
}

.uml-editor {
  width: 100%;
  flex: 1;
  resize: none;
  border: 1px solid #ecd8ea;
  border-radius: 10px;
  background: #fff;
  padding: 10px;
  font-family: Consolas, "Courier New", monospace;
  font-size: 13px;
  line-height: 1.5;
  color: #493e6e;
}

.uml-editor:focus {
  outline: none;
  border-color: #d8b8dd;
}

.uml-preview {
  flex: 1;
  border: 1px dashed #e4cfe4;
  border-radius: 10px;
  background: rgba(255, 255, 255, 0.9);
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: auto;
  padding: 10px;
}

.uml-preview img {
  max-width: 100%;
  height: auto;
  display: block;
}

.uml-empty {
  color: #9f91bc;
  font-size: 13px;
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

.dialog-card {
  width: min(92vw, 360px);
  border-radius: 14px;
  border: 1px solid #ead7e9;
  background: #fff;
  box-shadow: 0 18px 40px rgba(102, 67, 125, 0.2);
  padding: 14px;
}

.action-popover {
  position: fixed;
  z-index: 220;
  width: 220px;
}

.action-dialog {
  display: grid;
  gap: 8px;
}

.dialog-title {
  font-size: 16px;
  font-weight: 700;
  color: #645481;
  margin-bottom: 8px;
}

.dialog-desc {
  margin: 0 0 12px;
  color: #7b6f9d;
  font-size: 14px;
}

.dialog-input {
  width: 100%;
  height: 38px;
  border: 1px solid #e8d3e7;
  border-radius: 10px;
  padding: 0 10px;
  color: #5d507f;
  font-size: 14px;
  box-sizing: border-box;
}

.dialog-input:focus {
  outline: none;
  border-color: #d4b3d8;
}

.dialog-actions {
  margin-top: 12px;
  display: flex;
  justify-content: flex-end;
  gap: 8px;
}

.dialog-btn {
  border: 1px solid #e8d4e8;
  background: #fff;
  color: #6e608f;
  border-radius: 10px;
  padding: 8px 12px;
  font-size: 14px;
  width: 100%;
  text-align: center;
}

.dialog-btn.primary {
  background: rgba(255, 219, 239, 0.76);
  color: #8f4d78;
  border-color: #efcde2;
}

.dialog-btn.danger {
  background: #ffe9f1;
  border-color: #f3c4d5;
  color: #b65883;
}

.dialog-btn.ghost {
  background: #fff;
}

.welcome {
  display: flex;
  flex-direction: column;
  gap: 0;
}

@media (min-width: 1200px) {
  .header {
    display: grid;
    grid-template-columns: 1fr auto;
    align-items: start;
    gap: 12px;
  }

  .toolbar {
    margin-top: 2px;
    flex-wrap: nowrap;
  }
}

@media (max-width: 1080px) {
  .layout {
    grid-template-columns: 1fr;
  }

  .sidebar {
    max-height: 280px;
  }

  .hero-grid,
  .compare {
    grid-template-columns: 1fr;
  }

  .uml-main {
    grid-template-columns: 1fr;
  }

  .uml-header {
    flex-direction: column;
    align-items: flex-start;
    gap: 8px;
  }

  .user-bubble,
  .ai-bubble,
  .simple-assistant {
    max-width: 100%;
  }
}
</style>
