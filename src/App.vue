<script setup lang="ts">
import { ref, onMounted, watch } from 'vue';

// 定义响应式变量（状态）
// 保存输入的文本
const text = ref(''); 
// 保存可用语音列表
const voices = ref<SpeechSynthesisVoice[]>([]);
// 保存 Speaker 1 和 Speaker 2 的语音选择
const speaker1Lang = ref('en');
const speaker2Lang = ref('zh');
const speaker1VoiceURI = ref<string | null>(null);
const speaker2VoiceURI = ref<string | null>(null);
// 保存错误信息
const errorMessage = ref('');
// 录音相关状态
const isRecording = ref(false);
let mediaRecorder: MediaRecorder | null = null;
let recordedChunks: Blob[] = [];


// 获取 Web Speech API 的 SpeechSynthesis 实例
const synth = window.speechSynthesis;

// 获取可用语音列表的函数
const populateVoiceList = () => {
  // 防止在 speak() 触发 onvoiceschanged 时重复执行
  if (voices.value.length > 0) return;

  const availableVoices = synth.getVoices();
  // 检测浏览器类型
  const ua = navigator.userAgent.toLowerCase();
  const isEdge = ua.includes('edg');
  const isChrome = !isEdge && ua.includes('chrome');

  if (availableVoices.length > 0) {
    if (isChrome) {
      // 仅显示 Google 语音包，且只包含中文（不含粤语）、日语、英语
      voices.value = availableVoices.filter(voice =>
        voice.name.toLowerCase().includes('google') &&
        (
          (voice.lang.startsWith('zh') && voice.lang !== 'zh-HK') ||
          voice.lang.startsWith('ja') ||
          voice.lang.startsWith('en')
        )
      );
      // Speaker 1 默认 English Emma
      speaker1VoiceURI.value =
        voices.value.find(v => v.name.toLowerCase().includes('emma'))?.voiceURI ||
        voices.value.find(v => v.lang.startsWith('en'))?.voiceURI ||
        voices.value[0]?.voiceURI;
      // Speaker 2 默认台湾
      speaker2VoiceURI.value =
        voices.value.find(v => v.lang === 'zh-TW')?.voiceURI ||
        voices.value.find(v => v.lang.startsWith('zh'))?.voiceURI ||
        voices.value[0]?.voiceURI;
    } else if (isEdge) {
      // 仅显示 Microsoft 语音包，且只包含中文（不含粤语）、日语、指定英语
      let msVoices = availableVoices.filter(voice =>
        voice.name.toLowerCase().includes('microsoft') &&
        (
          (voice.lang.startsWith('zh') && voice.lang !== 'zh-HK') ||
          voice.lang.startsWith('ja') ||
          (
            voice.lang === 'en-US' ||
            voice.lang === 'en-GB' ||
            voice.lang === 'en-AU' ||
            voice.lang === 'en-CA' ||
            voice.lang === 'en-NZ'
          )
        )
      );
      // 排序：汉语 -> 英语(美国>澳大利亚>加拿大>新西兰>英国) -> 日语
      msVoices.sort((a, b) => {
        const langOrder = (lang: string) => {
          if (lang.startsWith('zh')) return 0;
          // 英语细分
          if (lang === 'en-US') return 1;
          if (lang === 'en-AU') return 2;
          if (lang === 'en-CA') return 3;
          if (lang === 'en-NZ') return 4;
          if (lang === 'en-GB') return 5;
          if (lang.startsWith('en')) return 6;
          if (lang.startsWith('ja')) return 7;
          return 8;
        };
        return langOrder(a.lang) - langOrder(b.lang);
      });
      voices.value = msVoices;
      // Speaker 1 默认 English Emma
      speaker1VoiceURI.value =
        voices.value.find(v => v.name.toLowerCase().includes('emma'))?.voiceURI ||
        voices.value.find(v => v.lang.startsWith('en'))?.voiceURI ||
        voices.value[0]?.voiceURI;
      // Speaker 2 默认 Chinese Yaoyao
      speaker2VoiceURI.value =
        voices.value.find(v => v.name.toLowerCase().includes('yaoyao'))?.voiceURI ||
        voices.value.find(v => v.lang.startsWith('zh'))?.voiceURI ||
        voices.value[0]?.voiceURI;
    } else {
      // 其它浏览器，仅显示中文（不含粤语）、日语、英语
      voices.value = availableVoices.filter(voice =>
        (voice.lang.startsWith('zh') && voice.lang !== 'zh-HK') ||
        voice.lang.startsWith('ja') ||
        voice.lang.startsWith('en')
      );
      // Speaker 1 默认 English Emma
      speaker1VoiceURI.value =
        voices.value.find(v => v.name.toLowerCase().includes('emma'))?.voiceURI ||
        voices.value.find(v => v.lang.startsWith('en'))?.voiceURI ||
        voices.value[0]?.voiceURI;
      // Speaker 2 默认台湾
      speaker2VoiceURI.value =
        voices.value.find(v => v.lang === 'zh-TW')?.voiceURI ||
        voices.value.find(v => v.lang.startsWith('zh'))?.voiceURI ||
        voices.value[0]?.voiceURI;
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

  speechSynthesis.getVoices().forEach(v => console.log(v.name, v.lang));
});

// 监视 voices 的变化
watch(voices, (newVoices) => {
  if (newVoices.length > 0) {
    if (!newVoices.find(v => v.voiceURI === speaker1VoiceURI.value)) {
      speaker1VoiceURI.value = newVoices[0].voiceURI;
    }
    if (!newVoices.find(v => v.voiceURI === speaker2VoiceURI.value)) {
      speaker2VoiceURI.value = newVoices.length > 1 ? newVoices[1].voiceURI : newVoices[0].voiceURI;
    }
  }
});

// 监视 Speaker 1 语言选择的变化
watch(speaker1Lang, (newLang) => {
  const filtered = filteredVoicesByLang(newLang);
  if (filtered.length > 0) {
    if (newLang === 'en') {
      // 如果是英语，优先选择 'Emma'
      const emmaVoice = filtered.find(v => v.name.toLowerCase().includes('emma'));
      speaker1VoiceURI.value = emmaVoice ? emmaVoice.voiceURI : filtered[0].voiceURI;
    } else {
      // 其它语言选择第一个
      speaker1VoiceURI.value = filtered[0].voiceURI;
    }
  }
});

// 监视 Speaker 2 语言选择的变化
watch(speaker2Lang, (newLang) => {
  const filtered = filteredVoicesByLang(newLang);
  if (filtered.length > 0) {
    if (newLang === 'en') {
      // 如果是英语，优先选择 'Emma'
      const emmaVoice = filtered.find(v => v.name.toLowerCase().includes('emma'));
      speaker2VoiceURI.value = emmaVoice ? emmaVoice.voiceURI : filtered[0].voiceURI;
    } else {
      // 其它语言选择第一个
      speaker2VoiceURI.value = filtered[0].voiceURI;
    }
  }
});

// 根据语言筛选语音列表
const filteredVoicesByLang = (lang: string) => {
  return voices.value.filter(v => v.lang.startsWith(lang));
};

// 将文本转换为语音文件的函数  TODO
const speak = async () => {
  if (synth.speaking) synth.cancel();
  if (!text.value || voices.value.length === 0) {
    errorMessage.value = '请输入文本并选择语音。';
    return;
  }
//  TODO
}
// 新增朗读功能（不录音，仅朗读）
const readAloud = async () => {
  if (synth.speaking) synth.cancel();

  if (text.value !== '' && voices.value.length > 0) {
    errorMessage.value = '';
    // 按行分割
    const lines = text.value.split('\n').filter(line => line.trim() !== '');

    for (const line of lines) {
      let utterText = line;
      let voice: SpeechSynthesisVoice | undefined;

      if (line.startsWith('Speaker 1:')) {
        // 使用下拉框选择的 Speaker 1 语音
        voice = voices.value.find(v => v.voiceURI === speaker1VoiceURI.value);
        utterText = line.replace(/^Speaker 1:\s*/, ''); // 去掉前缀，只朗读内容
      } else if (line.startsWith('Speaker 2:')) {
        // 使用下拉框选择的 Speaker 2 语音
        voice = voices.value.find(v => v.voiceURI === speaker2VoiceURI.value);
        utterText = line.replace(/^Speaker 2:\s*/, ''); // 去掉前缀，只朗读内容
      } else {
        // 其它情况：使用当前下拉框选择的语音
        voice = voices.value.find(v => v.voiceURI === speaker1VoiceURI.value);
      }

      if (!voice) {
        // 如果经过所有回退仍然找不到，才报错
        if (line.startsWith('Speaker 1:')) {
          errorMessage.value = '未为 "Speaker 1" 找到合适的语音。';
        } else if (line.startsWith('Speaker 2:')) {
          errorMessage.value = '未为 "Speaker 2" 找到合适的语音。';
        } else {
          errorMessage.value = '未找到合适的默认语音 (用于不带标记的文本)。';
        }
        return;
      }

      // 创建语音合成对象
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
    </header>

    <main>
      <!-- Speaker 1 语音选择 -->
      <div class="control-group">
        <label>Speaker 1 语音:</label>
        <div class="voice-select-row">
          <select v-model="speaker1Lang" class="select-box short-select">
            <option value="zh">汉语</option>
            <option value="en">英语</option>
            <option value="ja">日语</option>
          </select>
          <select v-model="speaker1VoiceURI" class="select-box">
            <option
              v-for="voice in filteredVoicesByLang(speaker1Lang)"
              :key="voice.voiceURI"
              :value="voice.voiceURI"
            >
              {{ voice.name }} ({{ voice.lang }})
            </option>
          </select>
        </div>
      </div>

      <!-- Speaker 2 语音选择 -->
      <div class="control-group">
        <label>Speaker 2 语音:</label>
        <div class="voice-select-row">
          <select v-model="speaker2Lang" class="select-box short-select">
            <option value="zh">汉语</option>
            <option value="en">英语</option>
            <option value="ja">日语</option>
          </select>
          <select v-model="speaker2VoiceURI" class="select-box">
            <option
              v-for="voice in filteredVoicesByLang(speaker2Lang)"
              :key="voice.voiceURI"
              :value="voice.voiceURI"
            >
              {{ voice.name }} ({{ voice.lang }})
            </option>
          </select>
        </div>
      </div>

      <div class="control-group">
        <label for="text-input">请输入文本:</label>
        <textarea id="text-input" v-model="text" rows="6" class="text-area"></textarea>
      </div>

      <div style="display: flex; gap: 1rem; width: 100%;">
        <button @click="readAloud" class="action-button">
          朗读
        </button>
        <button @click="speak" class="action-button">
          生成语音并下载
        </button>
      </div>

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
  width: 90vw;              /* 在中小屏幕上宽度为页面的90% */
  max-width: 1200px;        /* 限制最大宽度，以保证在大屏幕上不会过宽 */
  min-width: 400px;
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

.control-group:has(.voice-select-row) {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.control-group:has(.voice-select-row) > label {
  margin-bottom: 0;
  flex-shrink: 0;
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

.text-area {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #ccc;
  border-radius: 8px;
  font-size: 1rem;
  transition: border-color 0.2s, box-shadow 0.2s;
  min-height: 18rem;        /* 原来6行，扩大3倍为18行 */
  resize: vertical;
}

.voice-select-row {
  display: flex;
  gap: 1rem;
  align-items: center;
  flex-grow: 1;
}
.short-select {
  width: 7rem;
  min-width: 6rem;
}
</style>