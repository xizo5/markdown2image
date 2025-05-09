<script setup lang="ts">
import { ref, onMounted, computed, watch, nextTick } from 'vue'
import { marked } from 'marked'
import { toPng, toJpeg } from 'html-to-image'
import { saveAs } from 'file-saver'
import 'github-markdown-css/github-markdown-light.css'
import mermaid from 'mermaid'

// 在console中输出消息，便于调试
const log = (message: string, ...args: any[]) => {
  console.log(`[Markdown2Image] ${message}`, ...args);
};

log('App started');

// 初始化mermaid，使用最简单的配置
mermaid.initialize({
  startOnLoad: false,
  theme: 'default'
});

const markdownInput = ref(`# Markdown to Image 转换器

## 使用说明
1. 在左侧编辑 Markdown 内容
2. 右侧预览渲染效果
3. 点击下方按钮可以将内容导出为图片

- 支持标准 Markdown 语法
- 支持 Mermaid 图表语法
- 可以导出 PNG 或 JPG 格式

\`\`\`js
// 示例代码
function hello() {
  console.log('Hello, World!');
}
\`\`\`

> 引用文本示例

| 表格 | 示例 |
|------|------|
| 内容1 | 内容2 |
| 内容3 | 内容4 |

## Mermaid 图表示例

\`\`\`mermaid
graph TD
    A[开始] --> B{判断条件}
    B -->|是| C[处理1]
    B -->|否| D[处理2]
    C --> E[结束]
    D --> E
\`\`\`

\`\`\`mermaid
sequenceDiagram
    参与者A->>参与者B: 你好！
    参与者B->>参与者A: 你好！
    参与者A->>参与者B: 如何使用Mermaid?
    参与者B->>参与者A: 参考Mermaid文档
\`\`\`
`)

const previewHtml = ref('')
const previewContainer = ref<HTMLElement | null>(null)
const exportFormat = ref('png')
const isExporting = ref(false)
const isFullscreen = ref(false)

// 定义解析后内容的类型
interface ContentPart {
  type: 'markdown' | 'mermaid';
  content: string;
}

// 分别处理Markdown和Mermaid内容
const processMixedContent = (): ContentPart[] => {
  log('处理混合内容');
  const parts: ContentPart[] = [];
  
  // 分离Markdown和Mermaid内容
  const text = markdownInput.value;
  let currentIndex = 0;
  let mermaidStart = text.indexOf('```mermaid', currentIndex);
  
  while (mermaidStart !== -1) {
    // 添加mermaid前的普通markdown内容
    if (mermaidStart > currentIndex) {
      const markdownContent = text.substring(currentIndex, mermaidStart).trim();
      if (markdownContent) {
        parts.push({ type: 'markdown', content: markdownContent });
      }
    }
    
    // 查找mermaid代码块结束位置
    const contentStart = text.indexOf('\n', mermaidStart) + 1;
    const mermaidEnd = text.indexOf('```', contentStart);
    
    if (mermaidEnd === -1) {
      // 没有找到结束标记，跳出循环
      break;
    }
    
    // 提取mermaid内容
    const mermaidContent = text.substring(contentStart, mermaidEnd).trim();
    parts.push({ type: 'mermaid', content: mermaidContent });
    
    // 更新索引位置到下一个搜索起点
    currentIndex = mermaidEnd + 3;
    mermaidStart = text.indexOf('```mermaid', currentIndex);
  }
  
  // 添加最后剩余的markdown内容
  if (currentIndex < text.length) {
    const markdownContent = text.substring(currentIndex).trim();
    if (markdownContent) {
      parts.push({ type: 'markdown', content: markdownContent });
    }
  }
  
  log(`内容解析完成，共 ${parts.length} 部分`);
  return parts;
};

// 更新预览并渲染Mermaid图表
const updatePreview = async () => {
  log('开始更新预览');
  
  // 清空预览容器
  if (previewContainer.value) {
    previewContainer.value.innerHTML = '';
    
    // 处理混合内容
    const parts = processMixedContent();
    
    for (const part of parts) {
      if (part.type === 'markdown') {
        // 渲染普通Markdown
        const div = document.createElement('div');
        div.className = 'markdown-part';
        div.innerHTML = marked.parse(part.content, { breaks: true }) as string;
        previewContainer.value.appendChild(div);
      } else if (part.type === 'mermaid') {
        // 为Mermaid创建专用容器
        const container = document.createElement('div');
        container.className = 'mermaid-container';
        container.style.background = '#f6f8fa';
        container.style.padding = '16px';
        container.style.borderRadius = '6px';
        container.style.marginTop = '16px';
        container.style.marginBottom = '16px';
        
        const mermaidDiv = document.createElement('div');
        mermaidDiv.className = 'mermaid';
        mermaidDiv.textContent = part.content;
        container.appendChild(mermaidDiv);
        
        previewContainer.value.appendChild(container);
      }
    }
    
    // 渲染所有Mermaid图表
    try {
      log('开始渲染Mermaid图表');
      await mermaid.run();
      log('Mermaid图表渲染完成');
    } catch (error) {
      log('Mermaid图表渲染失败:', error);
    }
  }
};

// 导出为图片
const exportToImage = async () => {
  if (!previewContainer.value) return
  
  log('开始导出图片');
  isExporting.value = true
  try {
    // 直接使用当前预览容器，确保样式一致
    const currentPreview = previewContainer.value
    
    // 记录原始样式和滚动状态
    const originalHeight = currentPreview.style.height
    const originalMinHeight = currentPreview.style.minHeight
    const originalOverflow = currentPreview.style.overflowY
    const originalScrollTop = currentPreview.scrollTop
    
    // 临时修改样式以显示完整内容
    currentPreview.style.height = 'auto'
    currentPreview.style.minHeight = 'auto'
    currentPreview.style.overflowY = 'visible'
    currentPreview.scrollTop = 0
    
    // 给足够时间让渲染完成
    await new Promise(resolve => setTimeout(resolve, 1000))
    
    // 配置导出选项
    const options = {
      quality: 1,
      backgroundColor: 'white',
      pixelRatio: 2,
      style: {
        'margin': '1rem'
      }
    }
    
    // 根据选择的格式导出图片
    let imageBlob: Blob
    if (exportFormat.value === 'png') {
      const pngData = await toPng(currentPreview, options)
      imageBlob = await (await fetch(pngData)).blob()
      saveAs(imageBlob, 'markdown-export.png')
    } else {
      const jpegData = await toJpeg(currentPreview, options)
      imageBlob = await (await fetch(jpegData)).blob()
      saveAs(imageBlob, 'markdown-export.jpg')
    }
    
    // 恢复原始样式
    currentPreview.style.height = originalHeight
    currentPreview.style.minHeight = originalMinHeight
    currentPreview.style.overflowY = originalOverflow
    currentPreview.scrollTop = originalScrollTop
    
    log('图片导出成功');
  } catch (error) {
    log('导出图片失败', error)
    alert('导出图片失败，请重试：' + String(error))
  } finally {
    isExporting.value = false
  }
}

// 手动刷新Mermaid图表
const refreshMermaid = async () => {
  try {
    log('手动刷新Mermaid图表');
    await mermaid.run();
    log('手动刷新完成');
  } catch (error) {
    log('手动刷新失败:', error);
  }
};

// 切换全屏模式
const toggleFullscreen = () => {
  isFullscreen.value = !isFullscreen.value
  
  // 如果浏览器支持元素全屏API，则使用它
  if (isFullscreen.value) {
    if (document.documentElement.requestFullscreen) {
      document.documentElement.requestFullscreen()
    }
  } else {
    if (document.exitFullscreen) {
      document.exitFullscreen()
    }
  }
}

// 计算全屏按钮文本
const fullscreenButtonText = computed(() => {
  return isFullscreen.value ? '退出全屏' : '全屏模式'
})

// 监听Markdown输入变化，更新预览
watch(markdownInput, () => {
  updatePreview()
})

// 初始化和监听事件
onMounted(() => {
  log('组件挂载完成');
  // 初始渲染
  updatePreview()
  
  // 监听全屏状态变化
  document.addEventListener('fullscreenchange', () => {
    isFullscreen.value = !!document.fullscreenElement
  })
})
</script>

<template>
  <div class="container" :class="{ 'fullscreen': isFullscreen }">
    <h1 class="app-title">Markdown 转图片工具</h1>
    
    <div class="editor-container">
      <div class="editor-panel">
        <h2>编辑 Markdown</h2>
        <textarea 
          v-model="markdownInput" 
          class="markdown-editor"
          placeholder="在此输入Markdown内容..." 
        ></textarea>
      </div>
      
      <div class="preview-panel">
        <h2>
          预览
          <button 
            class="refresh-button" 
            title="刷新图表" 
            @click="refreshMermaid"
          >
            ↻
          </button>
        </h2>
        <div 
          ref="previewContainer" 
          class="markdown-preview markdown-body" 
        ></div>
      </div>
    </div>
    
    <div class="export-controls">
      <button 
        @click="toggleFullscreen" 
        class="fullscreen-button"
      >
        {{ fullscreenButtonText }}
      </button>
      
      <div class="format-selector">
        <label>
          <input type="radio" v-model="exportFormat" value="png"> PNG
        </label>
        <label>
          <input type="radio" v-model="exportFormat" value="jpg"> JPG
        </label>
      </div>
      
      <button 
        @click="exportToImage" 
        class="export-button"
        :disabled="isExporting"
      >
        {{ isExporting ? '导出中...' : `导出为 ${exportFormat.toUpperCase()}` }}
      </button>
    </div>
    
    <footer class="footer">
      <p>基于 Vue 3 + TypeScript + Vite 开发</p>
    </footer>
  </div>
</template>

<style>
html, body {
  margin: 0;
  padding: 0;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  background-color: #f5f5f5;
  color: #333;
  /* height: 100%; */
}

body, #app {
  /* min-height: 100vh; */
  width: 100%;
}

.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem 1rem;
  min-height: 100vh;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
}

.container.fullscreen {
  max-width: 100%;
  padding: 1rem;
  background-color: #f5f5f5;
}

.app-title {
  text-align: center;
  margin-bottom: 2rem;
  color: #333;
}

.editor-container {
  display: flex;
  gap: 30px;
  flex-direction: row;
}

@media (max-width: 768px) {
  .editor-container {
    flex-direction: column;
  }
}

.editor-panel, .preview-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  position: relative;
}

.editor-panel h2, .preview-panel h2 {
  margin-top: 0;
  padding-bottom: 0.5rem;
  border-bottom: 1px solid #eee;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.refresh-button {
  font-size: 18px;
  background: none;
  border: none;
  cursor: pointer;
  color: #2196F3;
  padding: 0 8px;
}

.refresh-button:hover {
  color: #0D47A1;
}

.markdown-editor {
  /* width: 100%; */
  flex: 1;
  padding: 1rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-family: 'Courier New', Courier, monospace;
  resize: vertical;
  font-size: 14px;
  line-height: 1.5;
}

.container.fullscreen .markdown-editor {
}

.markdown-preview {
  width: 100%;
  flex: 1;
  padding: 1rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  background-color: white;
  overflow-y: auto;
  box-sizing: border-box;
}

.container.fullscreen .markdown-preview {
}

.export-controls {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 1rem;
  margin: 2rem 0;
  flex-wrap: wrap;
}

.format-selector {
  display: flex;
  gap: 1rem;
}

.export-button, .fullscreen-button {
  padding: 0.6rem 1.2rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  font-weight: bold;
  transition: background-color 0.3s;
}

.export-button {
  background-color: #4CAF50;
  color: white;
}

.export-button:hover {
  background-color: #388E3C;
}

.export-button:disabled {
  background-color: #9E9E9E;
  cursor: not-allowed;
}

.fullscreen-button {
  background-color: #2196F3;
  color: white;
}

.fullscreen-button:hover {
  background-color: #1976D2;
}

/* Markdown样式 */
.markdown-body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif;
  font-size: 16px;
  line-height: 1.6;
  word-wrap: break-word;
  padding: 1rem;
}

/* Mermaid图表样式 */
.mermaid-container {
  display: flex;
  justify-content: center;
  margin: 1rem 0;
}

.mermaid {
  font-family: 'trebuchet ms', verdana, arial, sans-serif;
  font-size: 16px;
  margin: 0 auto;
}

.footer {
  text-align: center;
  color: #666;
  margin-top: 3rem;
  font-size: 0.9rem;
}
</style>
