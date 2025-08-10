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
// 当前朗读行索引
const currentLineIndex = ref<number | null>(null);

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

// 旧的 recordTabAsWav（按固定时长）会在未等待用户点“共享”时就开始朗读 => 改成可 start/stop 的录制器
// 新：startWavRecorder 等待用户完成“共享”后再返回 recorder，朗读再开始
function startWavRecorder() {
  return new Promise<{
    stop: () => Blob
  }>(async (resolve, reject) => {
    try {
      const stream = await navigator.mediaDevices.getDisplayMedia({ video: true, audio: true });
      // 只要音频，视频轨道关掉
      stream.getVideoTracks().forEach(t => t.stop());

      const ctx = new AudioContext();
      const source = ctx.createMediaStreamSource(stream);
      const processor = ctx.createScriptProcessor(4096, 1, 1);
      const samples: Float32Array[] = [];
      source.connect(processor);
      processor.connect(ctx.destination);
      processor.onaudioprocess = e => {
        samples.push(new Float32Array(e.inputBuffer.getChannelData(0)));
      };

      const stop = () => {
        processor.disconnect();
        source.disconnect();
        stream.getAudioTracks().forEach(t => t.stop());

        // 合并 Float32 -> PCM16
        let length = samples.reduce((sum, a) => sum + a.length, 0);
        const pcm16 = new DataView(new ArrayBuffer(length * 2));
        let offset = 0;
        for (const f32 of samples) {
          for (let i = 0; i < f32.length; i++, offset += 2) {
            let s = Math.max(-1, Math.min(1, f32[i]));
            pcm16.setInt16(offset, s < 0 ? s * 0x8000 : s * 0x7FFF, true);
          }
        }
        // 封装 WAV
        const sampleRate = ctx.sampleRate;
        const buffer = new ArrayBuffer(44 + pcm16.byteLength);
        const view = new DataView(buffer);
        const w = (o: number, s: string) => { for (let i = 0; i < s.length; i++) view.setUint8(o + i, s.charCodeAt(i)); };
        w(0,'RIFF');
        view.setUint32(4, 36 + pcm16.byteLength, true);
        w(8,'WAVE');
        w(12,'fmt ');
        view.setUint32(16,16,true);
        view.setUint16(20,1,true);      // PCM
        view.setUint16(22,1,true);      // mono
        view.setUint32(24,sampleRate,true);
        view.setUint32(28,sampleRate*2,true);
        view.setUint16(32,2,true);
        view.setUint16(34,16,true);
        w(36,'data');
        view.setUint32(40, pcm16.byteLength, true);
        new Uint8Array(buffer,44).set(new Uint8Array(pcm16.buffer));
        return new Blob([buffer], { type: 'audio/wav' });
      };

      resolve({ stop });
    } catch (e) {
      reject(e);
    }
  });
}

// 将文本转换为语音文件的函数
const speak = async () => {
  if (synth.speaking) synth.cancel();
  if (!text.value || voices.value.length === 0) {
    errorMessage.value = '请输入文本并选择语音。';
    return;
  }

  // 1. 先请求共享（等待用户点击“共享”完成）——避免“还没共享就开始朗读”
  errorMessage.value = '请在弹窗中选择“此标签页”并勾选共享音频，然后点击共享。';
  let recorder: { stop: () => Blob };
  try {
    recorder = await startWavRecorder(); // 等待用户授权完成
  } catch (e) {
    errorMessage.value = '获取标签页音频失败或已取消共享。';
    return;
  }
  errorMessage.value = '正在朗读并录音，请稍候…';

  // 2. 逐行朗读
  const lines = text.value.split('\n').filter(l => l.trim() !== '');
  for (let i = 0; i < lines.length; i++) {
    const raw = lines[i];
    let utterText = raw;
    let voice: SpeechSynthesisVoice | undefined;

    if (raw.startsWith('Speaker 1:')) {
      voice = voices.value.find(v => v.voiceURI === speaker1VoiceURI.value);
      utterText = raw.replace(/^Speaker 1:\s*/, '');
    } else if (raw.startsWith('Speaker 2:')) {
      voice = voices.value.find(v => v.voiceURI === speaker2VoiceURI.value);
      utterText = raw.replace(/^Speaker 2:\s*/, '');
    } else {
      voice = voices.value.find(v => v.voiceURI === speaker1VoiceURI.value);
    }

    if (!voice) {
      errorMessage.value = '未找到合适的语音，请重新选择。';
      return;
    }

    const utter = new SpeechSynthesisUtterance(utterText);
    utter.voice = voice;
    await new Promise<void>((resolve, reject) => {
      utter.onend = () => resolve();
      utter.onerror = (e) => {
        console.error(e);
        errorMessage.value = '朗读出错';
        reject(e);
      };
      synth.speak(utter);
    });
  }

  // 3. 停止录制并下载
  const wavBlob = recorder.stop();
  const a = document.createElement('a');
  a.href = URL.createObjectURL(wavBlob);
  a.download = 'tts-audio.wav';
  a.click();
  errorMessage.value = 'WAV 文件已下载。';
}

// 新增朗读功能（不录音，仅朗读）
const readAloud = async () => {
  if (synth.speaking) synth.cancel();

  if (text.value !== '' && voices.value.length > 0) {
    errorMessage.value = '';
    const lines = text.value.split('\n').filter(line => line.trim() !== '');

    for (let i = 0; i < lines.length; i++) {
      const line = lines[i];
      let utterText = line;
      let voice: SpeechSynthesisVoice | undefined;

      if (line.startsWith('Speaker 1:')) {
        voice = voices.value.find(v => v.voiceURI === speaker1VoiceURI.value);
        utterText = line.replace(/^Speaker 1:\s*/, '');
      } else if (line.startsWith('Speaker 2:')) {
        voice = voices.value.find(v => v.voiceURI === speaker2VoiceURI.value);
        utterText = line.replace(/^Speaker 2:\s*/, '');
      } else {
        voice = voices.value.find(v => v.voiceURI === speaker1VoiceURI.value);
      }

      if (!voice) {
        errorMessage.value = '未找到合适的语音，请从下拉列表中选择一个。';
        currentLineIndex.value = null;
        return;
      }

      // 设置当前高亮行
      currentLineIndex.value = i;

      let utterance = new SpeechSynthesisUtterance(utterText);
      utterance.voice = voice;

      utterance.onerror = (event) => {
        console.error('SpeechSynthesisUtterance.onerror', event);
        errorMessage.value = `语音生成时发生错误: ${event.error}`;
      };

      await new Promise<void>((resolve) => {
        utterance.onend = () => resolve();
        synth.speak(utterance);
      });
    }
    // 朗读结束后取消高亮
    currentLineIndex.value = null;
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

      <!-- 新增的朗读行高亮显示区域 -->
      <div class="read-lines-preview">
        <div
          v-for="(line, idx) in text.split('\n').filter(l => l.trim() !== '')"
          :key="idx"
          :class="['read-line', { active: idx === currentLineIndex }]"
        >
          {{ line }}
        </div>
      </div>
    </main>
  </div>
</template>

<style>
/* 调整：减少左侧余白 */
body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  background-color: #f0f2f5;
  color: #333;
  min-height: 100vh;
  margin: 0;
  display: flex;
  justify-content: flex-start; /* 原 center -> 左对齐，去掉左侧空白 */
  align-items: flex-start;
}

.app-container {
  background-color: #ffffff;
  padding: 2rem 2rem 2rem 1.5rem; /* 左内边距略减小 (原 2rem 3rem) */
  border-radius: 12px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.1);
  width: 1200px;              /* 避免 135vw 造成左右留白/滚动，可按需再调 */
  max-width: 95vw;
  min-width: 600px;
  margin: 0 0 0 0.5rem;        /* 靠左，仅保留很小间距 */
  display: flex;
  flex-direction: column;
  align-items: center;
  box-sizing: border-box;
}

/* 如果仍觉得左边多，可再把 margin-left 改成 0，或把 padding-left 再减小 */
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
  min-height: 36rem;        /* 原来18rem，变为两倍36rem */
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

/* 新增的朗读行高亮显示样式 */
.read-lines-preview {
  margin: 1.5rem 0 0 0;
  background: #f7fafd;
  border-radius: 8px;
  padding: 1rem;
  min-height: 3rem;
  font-size: 1.1rem;
}
.read-line {
  padding: 0.2rem 0.5rem;
  border-radius: 4px;
  transition: background 0.2s;
  font-size: 1.3em; /* 普通行字体也变大 */
}
.read-line.active {
  background: #ffe082;
  color: #e53935;      /* 高亮行字体为红色 */
  font-weight: bold;
  font-size: 1.5em; /* 高亮行字体更大 */
}
</style>