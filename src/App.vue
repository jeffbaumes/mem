<script setup lang="ts">
import pako from 'pako';
import { onBeforeUnmount, onMounted, ref } from 'vue';

enum WordState {
  Hidden,
  Revealed,
  Correct,
  Incorrect,
};

type Word = {
  text: string;
  state: WordState;
  paragraphBreak?: boolean;
  sentenceBreak?: boolean;
};

const words = ref<Word[]>([]);
const text = ref('');
const title = ref('');
const started = ref(false);
const keys = ref<string[]>(['qwertyuiop', 'asdfghjkl', 'zxcvbnm', '←→↑↓']);
const mainContentRef = ref<HTMLElement | null>(null);

const findFirstLetter = (str: string) => {
  const match = str.match(/\p{L}/u);
  return match ? match[0] : null;
};

const scrollToBottom = () => {
  const mainContent = mainContentRef.value;
  if (mainContent) {
    mainContent.scrollTop = mainContent.scrollHeight;
  }
};

const handleKey = (key: string) => {
  const modifierKeys = ['Shift', 'Control', 'Alt', 'Meta'];
  if (modifierKeys.includes(key)) {
    return;
  }
  setTimeout(scrollToBottom, 0);
  let wordIndex = words.value.findIndex((word) => word.state === WordState.Hidden);
  if (wordIndex === -1) {
    wordIndex = words.value.length;
  }
  if (key === 'ArrowLeft' || key === 'Backspace' || key === '←') {
    if (wordIndex > 0) {
      words.value[wordIndex - 1].state = WordState.Hidden;
    }
    return;
  }
  if (key === 'ArrowRight' || key === 'Enter' || key === ' ' || key === '→') {
    if (wordIndex < words.value.length) {
      words.value[wordIndex].state = WordState.Incorrect;
    }
    return;
  }
  if (key === 'Escape') {
    words.value.forEach((word) => word.state = WordState.Hidden);
    return;
  }
  if (key === 'ArrowUp' || key === '↑') {
    // Move at least one word back
    if (wordIndex > 0) {
      words.value[wordIndex - 1].state = WordState.Hidden;
      wordIndex -= 1;
    }
    // Find the end of the last sentence
    while (wordIndex > 0 && !words.value[wordIndex - 1].sentenceBreak) {
      words.value[wordIndex - 1].state = WordState.Hidden;
      wordIndex -= 1;
    }
    return;
  }
  if (key === 'ArrowDown' || key === '↓') {
    // Find the end of the next sentence
    while (wordIndex < words.value.length && !words.value[wordIndex].sentenceBreak) {
      words.value[wordIndex].state = WordState.Incorrect;
      wordIndex += 1;
    }
    if (wordIndex < words.value.length) {
      words.value[wordIndex].state = WordState.Incorrect;
    }
    return;
  }
  if (wordIndex < words.value.length) {
    const word = words.value[wordIndex];
    const firstLetter = findFirstLetter(word.text);
    if (firstLetter == null || firstLetter.toLocaleLowerCase() === key.toLocaleLowerCase()) {
      word.state = WordState.Correct;
    } else {
      word.state = WordState.Incorrect;
    }
  }
};

const handleKeyDown = (event: KeyboardEvent) => {
  if (!started.value) {
    return;
  }
  handleKey(event.key);
  event.preventDefault();
};

const updateURLWithText = () => {
  const gzippedText = pako.gzip(text.value);
  const base64Text = btoa(String.fromCharCode(...new Uint8Array(gzippedText)));
  const newUrl = `${window.location.pathname}?q=${encodeURIComponent(base64Text)}&title=${encodeURIComponent(title.value)}`;
  if (window.location.href !== window.location.origin + newUrl) {
    window.history.pushState({ path: newUrl }, '', newUrl);
  }
};

const updateTextWithURL = () => {
  const queryParams = new URLSearchParams(window.location.search);
  const queryValue = queryParams.get('q');
  if (queryValue !== null) {
    try {
      const decodedData = atob(queryValue);
      const charData = decodedData.split('').map(c => c.charCodeAt(0));
      const binData = new Uint8Array(charData);
      const inflatedData = pako.ungzip(binData, { to: 'string' });
      text.value = inflatedData;
    } catch (e) {
      console.error('Error inflating data:', e);
    }
  }
  const titleValue = queryParams.get('title');
  if (titleValue !== null) {
    title.value = titleValue;
  }
};

const start = () => {
  if (text.value.length === 0) {
    return;
  }

  words.value = [];
  text.value.split(/\n\n+/).forEach((paragraph) => {
    paragraph.trim().split(/\s+/).forEach((word) => {
      if (word.length === 0) {
        return;
      }
      const sentenceBreak = word.endsWith('.') || word.endsWith('!') || word.endsWith('?');
      words.value.push({ text: word, state: WordState.Hidden, sentenceBreak });
    });
    if (words.value.length > 0) {
      words.value[words.value.length - 1].paragraphBreak = true;
    }
  });

  started.value = true;
};

const stop = () => {
  words.value = [];
  started.value = false;
};

const countState = (state: WordState) => {
  return words.value.reduce((prev, word) => word.state === state ? prev + 1 : prev, 0);
};

onMounted(() => {
  window.addEventListener('keydown', handleKeyDown);
  window.addEventListener('popstate', updateTextWithURL);
  updateTextWithURL();
  if (text.value.length > 0) {
    start();
  }
});

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleKeyDown);
  window.removeEventListener('popstate', updateTextWithURL);
});

</script>

<template>
  <div class="container mx-auto p-2 flex flex-col h-screen max-w-prose">
    <header>
      <div class="text-2xl font-bold mb-2">{{ title }}</div>
      <div v-if="!started">
        <button class="btn btn-sm mr-2 mb-2" v-if="text.length > 0" @click="start()">Start memorizing</button>
        <button class="btn btn-sm mr-2 mb-2" v-if="text.length > 0" @click="updateURLWithText()">Save in URL to bookmark or share</button>
      </div>
      <div v-else>
        <div>
          <button class="btn btn-sm mr-2 mb-2" @click="stop()">Back</button>
          <button class="btn btn-sm mr-2 mb-2" @click="words.forEach((word) => word.state = WordState.Hidden)">Start over</button>
        </div>
        <div class="font-bold text-sm mb-2">
          {{ countState(WordState.Correct) }} of {{ countState(WordState.Correct) + countState(WordState.Incorrect) }} correct, {{ words.length }} total
          <span v-if="countState(WordState.Correct) === words.length" class="ml-1">🎉</span>
        </div>
      </div>
    </header>
    <main class="flex-1 overflow-y-auto" ref="mainContentRef">
      <div v-if="!started">
        <input v-model="title" class="input input-bordered w-full mb-2" placeholder="Title (optional)">
        <textarea v-model="text" class="textarea textarea-bordered w-full h-80 text-base" placeholder="Enter text to memorize"></textarea>
      </div>
      <div v-else>
        <span v-if="words.reduce((prev, word) => word.state !== WordState.Hidden ? prev + 1 : prev, 0) > 0" v-for="word, ind in words" :key="ind">
          <span v-if="word.state !== WordState.Hidden" :class="{'text-error': word.state === WordState.Incorrect}">{{`${word.text} `}}</span>
          <br v-if="word.state !== WordState.Hidden && word.paragraphBreak">
          <br v-if="word.state !== WordState.Hidden && word.paragraphBreak">
        </span>
        <span v-else>
          <div class="italic">Instructions: Type the first letter of each word. Arrow keys advance words or sentences.</div>
        </span>
      </div>
    </main>
    <footer class="touch-manipulation">
      <div v-for="line in keys" class="flex justify-center w-full mb-1">
        <div style="width: 10%" v-for="key in line">
          <button class="bg-gray-100 w-11/12 rounded-btn flex align-center justify-center py-2 font-bold" @click="handleKey(key)">{{ key }}</button>
        </div>
      </div>
    </footer>
  </div>
</template>
