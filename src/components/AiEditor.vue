<template>
  <div class="ai-editor-layout">
    <!-- 左侧编辑器 -->
    <div class="editor-panel">
      <div class="hint">💡 请选择需要优化的文本</div>
      <div class="editor-content-fixed">
        <editor-content :editor="editor" />
      </div>
      <div class="button-group" style="margin-top: 18px;">
        <button @click="runAiCommand('keypoints')" :disabled="isDisabled">提取要点</button>
        <button @click="runAiCommand('rephrase')" :disabled="isDisabled">改写</button>
        <button @click="runAiCommand('summarize')" :disabled="isDisabled">总结</button>
        <button @click="runAiCommand('simplify')" :disabled="isDisabled">简化</button>
        <button @click="runAiCommand('fixSpelling')" :disabled="isDisabled">纠正拼写</button>
        <button @click="runAiCommand('continueWriting')" :disabled="isDisabled">续写</button>
        <button @click="runAiCommand('emojify')" :disabled="isDisabled">添加表情</button>
        <button @click="runAiCommand('deEmojify')" :disabled="isDisabled">移除表情</button>
        <button @click="runAiCommand('translateChinese')" :disabled="isDisabled">翻译为中文</button>
        <button @click="runAiCommand('translateEnglish')" :disabled="isDisabled">翻译为英语</button>
      </div>
      <div v-if="state.errorMessage" class="hint error">{{ state.errorMessage }}</div>
      <div v-if="editor && editor.isEmpty" class="editor-placeholder">
        请在下方编辑区输入或粘贴文本
      </div>
    </div>
    <!-- 右侧AI结果 -->
    <div class="ai-result-panel">
      <div class="ai-result-title">AI 生成内容</div>
      <!-- 历史对话区，固定高度，内容多时可滚动，内容为空时显示空白 -->
      <div class="ai-chat-history-scroll">
        <template v-if="chatHistory.length > 0">
          <div v-for="(item, idx) in chatHistory" :key="idx" class="chat-item">
            <!-- 用户气泡在上方，靠右 -->
            <div class="chat-row single">
              <div class="chat-user-side">
                <div class="chat-bubble user">
                  <div class="chat-user">你：</div>
                  <div>{{ item.user }}</div>
                </div>
              </div>
            </div>
            <!-- AI气泡在下方，靠左 -->
            <div class="chat-row single">
              <div class="chat-ai-side">
                <div class="chat-bubble ai">
                  <div class="chat-ai">AI：</div>
                  <div>{{ item.ai }}</div>
                </div>
                <div class="chat-actions left">
                  <button
                    @click="replaceSelectionFromHistory(idx)"
                    :disabled="!item.ai"
                  >替代</button>
                </div>
              </div>
            </div>
          </div>
        </template>
        <template v-else>
          <!-- 没有历史时显示空白（可加提示或留空） -->
          <div style="height:100%;"></div>
        </template>
        <div v-if="state.isLoading" class="hint purple-spinner" style="text-align:center;margin:8px 0;">
          AI 正在生成中……
        </div>
      </div>
      <!-- 选中内容气泡和自定义prompt输入区移到历史对话区下方 -->
      <div>
        <div v-if="selectedTextForPrompt" class="selected-bubble">
          <span>选中内容：</span>
          <div class="chat-bubble user">{{ selectedTextForPrompt }}</div>
        </div>
        <div class="ai-custom-prompt">
          <input
            v-model="customPrompt"
            type="text"
            placeholder="请输入你的问题或需求"
            @keyup.enter="sendCustomPrompt"
          />
          <button @click="sendCustomPrompt" :disabled="!customPrompt || state.isLoading">发送</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import StarterKit from '@tiptap/starter-kit'
import { Editor, EditorContent } from '@tiptap/vue-3'
import OpenAI from 'openai'
import { Plugin } from '@tiptap/pm/state'
import { Decoration, DecorationSet } from '@tiptap/pm/view'
import { defineComponent } from 'vue'

export default defineComponent({
  components: {
    EditorContent,
  },

  data() {
    return {
      state: {
        isLoading: false,
        errorMessage: null,
        response: null,
      },
      editor: null,
      openai: null,
      apiKey: 'sk-8557191220f74fc4bc2e919eb1e8147b', // 请替换为你的API密钥
      highlightRange: null,
      customPrompt: '',
      chatHistory: [],
    }
  },

  computed: {
    isDisabled() {
      if (!this.editor) return true
      // 检查编辑器是否有内容
      return this.editor.isEmpty
    },
    selectedTextForPrompt() {
      if (!this.editor) return ''
      return this.editor.state.doc.textBetween(
        this.highlightRange?.from || 0, 
        this.highlightRange?.to || this.editor.state.doc.content.size
      )
    }
  },

  methods: {
    initOpenAI() {
      if (!this.apiKey) return
      this.openai = new OpenAI({
        apiKey: this.apiKey,
        baseURL: 'https://api.deepseek.com/v1',
        dangerouslyAllowBrowser: true,
      })
    },
    
    // 新增：自动选择全文的方法
    selectAllText() {
      if (!this.editor) return
      const { doc } = this.editor.state
      this.editor.chain().focus().selectAll().run()
      this.highlightRange = { from: 0, to: doc.content.size }
    },
    
    async runAiCommand(command) {
      if (!this.editor || !this.openai) return
      
      // 如果没有选中内容，自动选择全文
      const { from, to } = this.editor.state.selection
      if (from === to) {
        this.selectAllText()
      }
      
      const selectedText = this.selectedTextForPrompt
      this.highlightRange = { from: 0, to: this.editor.state.doc.content.size }
      
      const commandMap = {
        keypoints: '提取要点', rephrase: '改写', summarize: '总结', simplify: '简化',
        fixSpelling: '纠正拼写', continueWriting: '续写', emojify: '添加表情',
        deEmojify: '移除表情', translateChinese: '翻译为中文', translateEnglish: '翻译为英语'
      }
      const userQuestion = `请帮我${commandMap[command]}${from !== to ? '（针对选中内容）' : '（针对全文）'}`
      let prompt = ''
      switch (command) {
        case 'keypoints': prompt = `提取以下文本的关键点：\n\n${selectedText}`; break
        case 'rephrase': prompt = `用不同的表达方式重写以下文本：\n\n${selectedText}`; break
        case 'summarize': prompt = `总结以下文本的主要内容：\n\n${selectedText}`; break
        case 'simplify': prompt = `简化以下文本，使其更容易理解：\n\n${selectedText}`; break
        case 'fixSpelling': prompt = `修正以下文本中的拼写和语法错误：\n\n${selectedText}`; break
        case 'continueWriting': prompt = `继续以下文本的内容：\n\n${selectedText}`; break
        case 'emojify': prompt = `在以下文本中添加适当的表情符号：\n\n${selectedText}`; break
        case 'deEmojify': prompt = `从以下文本中移除所有表情符号：\n\n${selectedText}`; break
        case 'translateChinese': prompt = `将以下文本翻译成中文：\n\n${selectedText}`; break
        case 'translateEnglish': prompt = `将以下文本翻译成英语：\n\n${selectedText}`; break
      }
      try {
        this.state.isLoading = true
        this.state.errorMessage = null
        const response = await fetch('https://api.deepseek.com/v1/chat/completions', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${this.apiKey}`,
          },
          body: JSON.stringify({
            model: 'deepseek-chat',
            messages: [
              { role: 'system', content: '你是一个智能写作助手，帮助用户处理文本。请只返回普通文本，不要使用markdown格式，如果用户不要求翻译，原文使用哪种语言，返回文本使用哪种语言。' },
              { role: 'user', content: prompt }
            ],
            temperature: 0.7,
            max_tokens: 1000,
          }),
        })
        const data = await response.json()
        if (!data.choices || !Array.isArray(data.choices) || !data.choices[0]) {
          this.state.errorMessage = data.error?.message || 'AI接口返回异常，请检查API Key和配额'
          this.state.isLoading = false
          return
        }
        const aiResponse = data.choices[0].message.content
        this.state.response = aiResponse
        this.chatHistory.push({ user: userQuestion, ai: aiResponse })
      } catch (error) {
        this.state.errorMessage = `AI处理失败: ${error.message}`
      } finally {
        this.state.isLoading = false
      }
    },
    
    async sendCustomPrompt() {
      if (!this.customPrompt) return
      this.state.isLoading = true
      this.state.errorMessage = null
      
      // 如果没有选中内容，自动选择全文
      const { from, to } = this.editor.state.selection
      if (from === to) {
        this.selectAllText()
      }
      
      const selectedText = this.selectedTextForPrompt
      let prompt = selectedText
        ? `针对以下文本片段，${this.customPrompt}\n\n${selectedText}`
        : this.customPrompt
      this.highlightRange = { from: 0, to: this.editor.state.doc.content.size }
      
      try {
        const response = await fetch('https://api.deepseek.com/v1/chat/completions', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${this.apiKey}`,
          },
          body: JSON.stringify({
            model: 'deepseek-chat',
            messages: [
              { role: 'system', content: '你是一个智能写作助手，帮助用户处理文本。请只返回普通文本，不要使用markdown格式。' },
              { role: 'user', content: prompt }
            ],
            temperature: 0.7,
            max_tokens: 1000,
          }),
        })
        const data = await response.json()
        if (!data.choices || !Array.isArray(data.choices) || !data.choices[0]) {
          this.state.errorMessage = data.error?.message || 'AI接口返回异常，请检查API Key和配额'
          this.state.isLoading = false
          return
        }
        const aiResponse = data.choices[0].message.content
        this.state.response = aiResponse
        this.chatHistory.push({
          user: this.customPrompt + (selectedText ? `（针对选中内容）` : ''),
          ai: aiResponse,
        })
        this.customPrompt = ''
      } catch (error) {
        this.state.errorMessage = `AI处理失败: ${error.message}`
      } finally {
        this.state.isLoading = false
      }
    },
    
    replaceSelectionFromHistory(idx) {
      const historyItem = this.chatHistory[idx]
      if (!this.editor || !this.highlightRange || !historyItem.ai) return
      const { from, to } = this.highlightRange
      historyItem.originalText = this.editor.state.doc.textBetween(from, to)
      this.editor.chain().focus().deleteRange({ from, to }).insertContent(historyItem.ai).run()
      this.state.response = ''
      this.highlightRange = null
      historyItem.replaced = true
    },
    
    undoReplaceFromHistory(idx) {
      const historyItem = this.chatHistory[idx]
      if (!this.editor || !this.highlightRange || !historyItem.originalText) return
      const { from, to } = this.highlightRange
      this.editor.chain().focus().deleteRange({ from, to }).insertContent(historyItem.originalText).run()
      historyItem.replaced = false
    },
    
    discardHistory(idx) {
      this.chatHistory.splice(idx, 1)
    }
  },

  watch: {
    chatHistory() {
      this.$nextTick(() => {
        const chatScroll = this.$el.querySelector('.ai-chat-history-scroll')
        if (chatScroll) chatScroll.scrollTop = chatScroll.scrollHeight
      })
    },
    'editor.state.selection'() {
      const { from, to } = this.editor.state.selection
      this.highlightRange = from !== to ? { from, to } : null
    }
  },

  mounted() {
    this.initOpenAI()
    this.editor = new Editor({
      extensions: [
        StarterKit,
        new Plugin({
          props: {
            decorations: () => null
          }
        })
      ],
      content: `
        <p>Rocking like a mobile?</p>
        <p>Did you hear about the mobile phone that joined a rock band? Yeah, it was a real smartTONE!
        It rocked the stage with its gigabytes of rhythm and had everyone calling for an encore, but
        it couldn't resist the temptation to drop a few bars and left the audience in absolute silence.
        Turns out, it wasn't quite cut out for the music industry.</p>
        <p>They say it's now pursuing a career in ringtone composition. Who knows, maybe one day it'll top
        the charts with its catchy melodies!</p>
      `,
    })
    this.editor.registerPlugin(new Plugin({
      props: {
        decorations: (state) => {
          if (!this.highlightRange) return null
          const { from, to } = this.highlightRange
          return DecorationSet.create(state.doc, [
            Decoration.inline(from, to, { class: 'ai-highlight' })
          ])
        }
      }
    }))
    this.$nextTick(() => {
      const chatScroll = this.$el.querySelector('.ai-chat-history-scroll')
      if (chatScroll) chatScroll.scrollTop = chatScroll.scrollHeight
    })
  },

  beforeUnmount() {
    this.editor.destroy()
  },
})
</script>

<style lang="scss">
$main-green: #195c3e;
$main-green-light: #eaf5ee;
$main-green-dark: #143c2a;
$main-green-mid: #388e3c;

.ai-editor-layout {
  display: flex;
  gap: 32px;
  background: $main-green-light;
  min-height: 100vh;
  padding: 32px 0;
}
.editor-panel {
  flex: 1 1 0;
  min-width: 0;
  background: #fff;
  border-radius: 16px;
  box-shadow: 0 4px 24px rgba(25,92,62,0.08);
  padding: 32px 28px 24px 28px;
  margin-left: 32px;
  display: flex;
  flex-direction: column;
}
.button-group {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  margin-bottom: 18px;
}
.button-group button {
  background: linear-gradient(90deg, $main-green 60%, $main-green-mid 100%);
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 8px 18px;
  font-size: 15px;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.2s;
}
.button-group button:disabled {
  background: #e5eaf3;
  color: #aaa;
  cursor: not-allowed;
}
.hint {
  margin-bottom: 12px;
  font-size: 15px;
  color: $main-green;
}
.hint.error {
  color: #d93025;
  background: #fff0f0;
  border-radius: 6px;
  padding: 6px 12px;
}
.hint.purple-spinner {
  color: $main-green;
  font-weight: bold;
  font-size: 15px;
}
.editor-placeholder {
  color: #bbb;
  padding: 18px;
  text-align: center;
  position: absolute;
  width: 100%;
  pointer-events: none;
  z-index: 1;
  font-size: 16px;
  background: $main-green-light;
  border-radius: 8px;
}
.ai-result-panel {
  width: 400px;
  min-width: 320px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  background: #fff;
  border-radius: 16px;
  box-shadow: 0 4px 24px rgba(25,92,62,0.08);
  padding: 32px 24px 24px 24px;
  margin-right: 32px;
  position: relative;
}
.ai-result-title {
  font-weight: bold;
  font-size: 18px;
  margin-bottom: 8px;
  color: $main-green;
  letter-spacing: 1px;
}
.ai-highlight {
  background: #c8e6c9 !important;
}
.ai-chat-history-scroll {
  max-height: 260px;
  min-height: 260px;
  overflow-y: auto;
  padding-right: 4px;
  margin-bottom: 12px;
  scrollbar-width: thin;
  scrollbar-color: $main-green $main-green-light;
  background: #fff;
  border-radius: 12px;
}
.ai-chat-history-scroll::-webkit-scrollbar {
  width: 8px;
}
.ai-chat-history-scroll::-webkit-scrollbar-thumb {
  background: $main-green;
  border-radius: 8px;
}
.ai-chat-history-scroll::-webkit-scrollbar-track {
  background: $main-green-light;
  border-radius: 8px;
}
.chat-item {
  margin-bottom: 10px;
}
.chat-row {
  display: flex;
  justify-content: flex-start;
  align-items: flex-end;
  gap: 16px;
  margin-bottom: 0;
}
.chat-user-side {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  max-width: 60%;
  margin-left: auto;
}
.chat-ai-side {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  max-width: 60%;
}
.chat-bubble {
  position: relative;
  padding: 10px 16px;
  border-radius: 18px;
  margin-bottom: 4px;
  max-width: 100%;
  word-break: break-all;
  font-size: 15px;
  box-shadow: 0 2px 8px rgba(25,92,62,0.04);
}
.chat-bubble.ai {
  background: #f7f7f7;
  color: $main-green-dark;
  border-bottom-left-radius: 4px;
  border-top-left-radius: 0;
  margin-left: 8px;
  align-self: flex-start;
  /* 微信左侧气泡三角 */
  &::before {
    content: "";
    position: absolute;
    left: -8px;
    top: 16px;
    border-width: 8px 10px 8px 0;
    border-style: solid;
    border-color: transparent #f7f7f7 transparent transparent;
  }
}
.chat-bubble.user {
  background: linear-gradient(90deg, $main-green 60%, $main-green-dark 100%);
  color: #fff;
  border-bottom-right-radius: 4px;
  border-top-right-radius: 0;
  margin-right: 8px;
  align-self: flex-end;
  /* 微信右侧气泡三角 */
  &::before {
    content: "";
    position: absolute;
    right: -8px;
    top: 16px;
    border-width: 8px 0 8px 10px;
    border-style: solid;
    border-color: transparent transparent transparent $main-green;
  }
}
.chat-actions {
  display: flex;
  gap: 8px;
  margin-top: 2px;
}
.chat-actions.left {
  justify-content: flex-start;
}
.chat-actions button {
  background: $main-green-light;
  color: $main-green-dark;
  border: none;
  border-radius: 6px;
  padding: 4px 12px;
  font-size: 14px;
  cursor: pointer;
  box-shadow: 0 1px 4px rgba(25,92,62,0.06);
  transition: background 0.2s;
}
.chat-actions button:disabled {
  background: #e5eaf3;
  color: #aaa;
  cursor: not-allowed;
}
.selected-bubble {
  margin-bottom: 10px;
  font-size: 14px;
  color: $main-green;
  display: flex;
  align-items: center;
}
.selected-bubble .chat-bubble.user {
  display: inline-block;
  margin-left: 8px;
  background: linear-gradient(90deg, $main-green-light 60%, #d0e6db 100%);
  padding: 7px 14px;
  border-radius: 12px;
  max-width: 80%;
  word-break: break-all;
  font-size: 15px;
  box-shadow: 0 2px 8px rgba(25,92,62,0.04);
  color: $main-green-dark;
  max-height: 4.5em; // 约3行高度
  overflow-y: auto;
}
.ai-bottom-fixed {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 18px;
  background: #fff;
  z-index: 2;
}
.ai-custom-prompt {
  display: flex;
  gap: 10px;
  margin-bottom: 8px;
}
.ai-custom-prompt input {
  flex: 1;
  padding: 8px 12px;
  border-radius: 8px;
  border: 1px solid $main-green-mid;
  font-size: 15px;
  background: $main-green-light;
  box-shadow: 0 2px 8px rgba(25,92,62,0.04);
  color: $main-green-dark;
}
.ai-custom-prompt button {
  padding: 8px 20px;
  border-radius: 8px;
  background: linear-gradient(90deg, $main-green 60%, $main-green-mid 100%);
  color: #fff;
  border: none;
  cursor: pointer;
  font-size: 15px;
  font-weight: 500;
  box-shadow: 0 2px 8px rgba(25,92,62,0.08);
  transition: background 0.2s;
}
.ai-custom-prompt button:disabled {
  background: #e5eaf3;
  color: #aaa;
  cursor: not-allowed;
}
.editor-content-fixed {
  height: 320px;
  overflow-y: auto;
  border-radius: 12px;
  border: 1px solid $main-green-mid;
  background: $main-green-light;
  margin-bottom: 8px;
  box-shadow: 0 2px 8px rgba(25,92,62,0.04);
}
</style>  