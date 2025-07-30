<template>
  <div id="app" class="min-h-screen bg-gray-50 font-sans">
    <header class="bg-white shadow-md">
      <div class="container mx-auto px-4 py-6">
        <h1 class="text-2xl font-bold text-gray-800">AI 文本编辑器</h1>
        <p class="text-gray-600 mt-2">使用 AI 增强你的写作体验</p>
      </div>
    </header>
    
    <main class="container mx-auto px-4 py-8">
      <!-- 使用 AI 编辑器组件 -->
      <AiEditor />
    </main>
    
    <footer class="bg-gray-800 text-white py-8 mt-12">
      <div class="container mx-auto px-4">
        <p class="text-center">© 2025 AI 文本编辑器 | 由 Tiptap 和 OpenAI 提供支持</p>
      </div>
    </footer>
  </div>
</template>

<script>
// 导入 AI 编辑器组件
import AiEditor from './components/AiEditor.vue';

export default {
  name: 'App',
  components: {
    AiEditor, // 注册组件
  },
};
</script>

<style>
/* 全局样式 */
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
</style>