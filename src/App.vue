<script setup lang="ts">
import { ref, onMounted } from 'vue';

// 定义响应式变量（状态）
// 保存输入的文本
const text = ref('こんにちは、これはテストです。'); 
// 保存可用语音列表
const voices = ref<SpeechSynthesisVoice[]>([]); 
// 保存用户选择的语音URI
const selectedVoiceURI = ref<string | null>(null); 
// 保存错误信息
const errorMessage = ref('');

// 获取 Web Speech API 的 SpeechSynthesis 实例
const synth = window.speechSynthesis;

// 获取可用语音列表的函数
const populateVoiceList = () => {
  const availableVoices = synth.getVoices();
  if (availableVoices.length > 0) {
    // 只筛选 Microsoft 和 Google 的语音
    voices.value = availableVoices.filter(voice => 
      voice.name.includes('Microsoft') || voice.name.includes('Google')
    );
    // 默认选择列表中的第一个语音
    if (voices.value.length > 0) {
      selectedVoiceURI.value = voices.value[0].voiceURI;
    }
  } else {
    errorMessage.value = '您的浏览器不支持语音合成，或语音列表加载失败。';
  }
};

// 组件挂载（显示准备完成）时执行
onMounted(() => {
  // getVoices() 异步加载语音列表，因此需要等待 voiceschanged 事件
  if (synth.getVoices().length === 0) {
    synth.onvoiceschanged = populateVoiceList;
  } else {
    populateVoiceList();
  }
});

// 将文本转换为语音并播放的函数
const speak = async () => {
  if (synth.speaking) {
    synth.cancel();
  }

  if (text.value !== '' && voices.value.length > 0) {
    errorMessage.value = '';
    // 按行分割
    const lines = text.value.split('\n').filter(line => line.trim() !== '');

    for (const line of lines) {
      let utterText = line;
      let voice: SpeechSynthesisVoice | undefined;

      if (line.startsWith('Speaker 1:')) {
        // 汉语语音
        voice = voices.value.find(v => v.lang.startsWith('zh'));
        utterText = line.replace(/^Speaker 1:\s*/, ''); // 去掉前缀
      } else if (line.startsWith('Speaker 2:')) {
        // 英语语音
        voice = voices.value.find(v => v.lang.startsWith('en'));
        utterText = line.replace(/^Speaker 2:\s*/, ''); // 去掉前缀
      } else {
        // 默认用当前选择
        voice = voices.value.find(v => v.voiceURI === selectedVoiceURI.value);
      }

      if (!voice) {
        errorMessage.value = '未找到合适的语音。';
        return;
      }

      let utterance = new SpeechSynthesisUtterance(utterText);
      utterance.voice = voice;

      // 错误处理
      utterance.onerror = (event) => {
        console.error('SpeechSynthesisUtterance.onerror', event);
        errorMessage.value = `语音生成时发生错误: ${event.error}`;
      };

      // 等待当前朗读结束再朗读下一句
      await new Promise<void>((resolve) => {
        utterance.onend = () => resolve();
        synth.speak(utterance);
      });
    }
  } else {
    errorMessage.value = '请输入文本并选择语音。';
  }
};
</script>

<template>
  <div class="app-container">
    <header>
      <h1>文本转语音应用</h1>
      <p>TypeScript & Vue.js</p>
    </header>

    <main>
      <div class="control-group">
        <label for="voice-select">请选择语音:</label>
        <select id="voice-select" v-model="selectedVoiceURI" class="select-box">
          <option v-for="voice in voices" :key="voice.voiceURI" :value="voice.voiceURI">
            {{ voice.name }} ({{ voice.lang }})
          </option>
        </select>
        <p v-if="voices.length === 0" class="info-text">
          没有可用的语音。请使用 Edge 或 Chrome 浏览器尝试。
        </p>
      </div>

      <div class="control-group">
        <label for="text-input">请输入文本:</label>
        <textarea id="text-input" v-model="text" rows="6" class="text-area"></textarea>
      </div>

      <button @click="speak" class="action-button">
        转换为语音
      </button>

      <div v-if="errorMessage" class="error-message">
        <p>{{ errorMessage }}</p>
      </div>
    </main>
  </div>
</template>

<style>
/* 应用整体样式 */
body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  background-color: #f0f2f5;
  color: #333;
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  margin: 0;
}

.app-container {
  background-color: #ffffff;
  padding: 2rem 3rem;
  border-radius: 12px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.1);
  width: 100%;
  max-width: 600px;
}

header {
  text-align: center;
  margin-bottom: 2rem;
  border-bottom: 1px solid #e0e0e0;
  padding-bottom: 1rem;
}

header h1 {
  color: #1a73e8;
  margin: 0;
}

header p {
  margin: 0.25rem 0 0;
  color: #666;
}

.control-group {
  margin-bottom: 1.5rem;
}

.control-group label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: 600;
  color: #555;
}

.select-box,
.text-area {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #ccc;
  border-radius: 8px;
  font-size: 1rem;
  transition: border-color 0.2s, box-shadow 0.2s;
}

.select-box:focus,
.text-area:focus {
  outline: none;
  border-color: #1a73e8;
  box-shadow: 0 0 0 3px rgba(26, 115, 232, 0.2);
}

.action-button {
  width: 100%;
  padding: 0.8rem 1rem;
  font-size: 1.1rem;
  font-weight: 600;
  color: #fff;
  background-color: #1a73e8;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background-color 0.3s, transform 0.1s;
}

.action-button:hover {
  background-color: #155ab6;
}

.action-button:active {
  transform: scale(0.98);
}

.error-message {
  margin-top: 1rem;
  padding: 1rem;
  background-color: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
  border-radius: 8px;
}

.info-text {
  font-size: 0.9rem;
  color: #888;
  margin-top: 0.5rem;
}
</style>