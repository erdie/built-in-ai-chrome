<template>
  <div class="p-6 max-w-lg mx-auto">
    <h2 class="text-2xl font-bold mb-4">Support Chat with Auto Translate</h2>

    <!-- Input Form -->
    <form v-if="formVisible" @submit="translateText" class="space-y-4">
      <textarea
        v-model="inputText"
        @input="detectLanguage"
        placeholder="Type your message..."
        class="w-full p-2 border rounded"
      ></textarea>

      <p class="text-sm text-gray-600">
        Detected Language: <span class="font-medium">{{ detectedLanguage }}</span>
        <span v-if="confidence">({{ confidence }})</span>
      </p>

      <select v-model="targetLanguage" class="border p-2 rounded">
        <option value="en">English</option>
        <option value="es">Spanish</option>
        <option value="ja">Japanese</option>
      </select>

      <button
        type="submit"
        class="w-full bg-blue-500 text-white py-2 rounded hover:bg-blue-600"
      >
        Translate
      </button>
    </form>

    <!-- Output -->
    <output
      v-if="outputText"
      class="block mt-4 p-4 bg-gray-100 border rounded"
    >
      {{ outputText }}
    </output>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';

const inputText = ref('');
const outputText = ref('');
const detectedLanguage = ref('');
const confidence = ref('');
const targetLanguage = ref('en');
const formVisible = ref(false);

let detector = null;
let translator = null;

// Initialize the detector and translator
const initTranslation = async () => {
  if (!('translation' in self) || !('createDetector' in self.translation)) {
    console.error('Translation API not supported in your browser.');
    return;
  }

  detector = await self.translation.createDetector();

  // Make form visible after initialization
  formVisible.value = true;
};

const detectLanguage = async () => {
  if (!inputText.value.trim()) {
    detectedLanguage.value = 'Not sure what language this is';
    confidence.value = '';
    return;
  }

  const { detectedLanguage: lang, confidence: conf } = (
    await detector.detect(inputText.value.trim())
  )[0];

  confidence.value = `${(conf * 100).toFixed(1)}%`;
  detectedLanguage.value = languageTagToHumanReadable(lang, 'en');
};

const translateText = async (event) => {
  event.preventDefault();

  if (!detector || !inputText.value.trim()) return;

  try {
    const sourceLanguage = (await detector.detect(inputText.value.trim()))[0].detectedLanguage;

    if (!['en', 'es', 'ja'].includes(sourceLanguage)) {
      outputText.value = 'Currently, only English ↔ Spanish and English ↔ Japanese are supported.';
      return;
    }

    translator = await self.translation.createTranslator({
      sourceLanguage,
      targetLanguage: targetLanguage.value,
    });

    outputText.value = await translator.translate(inputText.value.trim());
  } catch (error) {
    outputText.value = 'An error occurred. Please try again.';
    console.error(error.name, error.message);
  }
};

const languageTagToHumanReadable = (languageTag, targetLanguage) => {
  const displayNames = new Intl.DisplayNames([targetLanguage], {
    type: 'language',
  });
  return displayNames.of(languageTag);
};

onMounted(initTranslation);
</script>