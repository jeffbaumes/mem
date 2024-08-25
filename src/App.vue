<script setup lang="ts">
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

let words = ref<Word[]>([]);
let text = ref('');
let started = ref(false);

function findFirstLetter(str: string) {
  const match = str.match(/\p{L}/u);
  return match ? match[0] : null;
}

const handleKeyDown = (event: KeyboardEvent) => {
  if (!started.value) {
    return;
  }
  focusOnInput();
  let wordIndex = words.value.findIndex((word) => word.state === WordState.Hidden);
  if (wordIndex === -1) {
    wordIndex = words.value.length;
  }
  if (event.key === 'ArrowLeft' || event.key === 'Backspace') {
    if (wordIndex > 0) {
      words.value[wordIndex - 1].state = WordState.Hidden;
    }
    return;
  }
  if (event.key === 'ArrowRight' || event.key === 'Enter' || event.key === ' ') {
    if (wordIndex < words.value.length) {
      words.value[wordIndex].state = WordState.Incorrect;
    }
    return;
  }
  if (event.key === 'Escape') {
    words.value.forEach((word) => word.state = WordState.Hidden);
    return;
  }
  if (event.key === 'ArrowUp') {
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
  if (event.key === 'ArrowDown') {
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
    if (firstLetter == null || firstLetter.toLocaleLowerCase() === event.key.toLocaleLowerCase()) {
      word.state = WordState.Correct;
    } else {
      word.state = WordState.Incorrect;
    }
  }
};

const focusOnInput = () => {
  const inputElement = document.getElementById('keepKeyboardOpen') as HTMLInputElement;
  if (inputElement) {
    inputElement.focus();
  }
};

const updateURLWithText = () => {
  if (text.value) {
    const newUrl = `${window.location.pathname}?q=${encodeURIComponent(text.value)}`;
    window.history.pushState({ path: newUrl }, '', newUrl);
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

onMounted(() => {
  window.addEventListener('keydown', handleKeyDown);
  const queryParams = new URLSearchParams(window.location.search);
  const queryValue = queryParams.get('q');
  if (queryValue) {
    text.value = queryValue;
  }
});

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleKeyDown);
});

</script>

<template>
  <div class="container mx-auto p-4">
    <input id="keepKeyboardOpen" style="opacity: 0; position: absolute; bottom: 0;" />
    <div class="text-2xl font-bold mb-2">Mem</div>
    <div v-if="!started">
      <div class="mb-2">
        <button class="btn btn-sm mr-2" v-if="text.length > 0" @click="updateURLWithText">Set permalink</button>
        <button class="btn btn-sm mr-2" v-if="text.length > 0" @click="start()">Start</button>
      </div>
      <textarea v-model="text" class="textarea textarea-bordered w-full h-80" placeholder="Enter text to memorize"></textarea>
    </div>
    <div v-else>
      <div class="mb-2">
        <button class="btn btn-sm mr-2" @click="stop()">Stop</button>
        <button class="btn btn-sm mr-2" @click="words.forEach((word) => word.state = WordState.Hidden)">Reset</button>
      </div>
      <div class="text-sm mb-2">Type the first letter of each word. Arrow keys advance words or sentences. Esc to reset.</div>
      <hr>
      <div>
        <span v-for="word, ind in words" :key="ind">
          <span v-if="word.state !== WordState.Hidden" :class="{'text-error': word.state === WordState.Incorrect}">{{`${word.text} `}}</span>
          <br v-if="word.state !== WordState.Hidden && word.paragraphBreak">
          <br v-if="word.state !== WordState.Hidden && word.paragraphBreak">
        </span>
      </div>
    </div>
  </div>
</template>
