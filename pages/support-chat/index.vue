<template>
  <div class="min-h-screen bg-gray-100 p-4 flex flex-col items-center">
    <h1 class="text-2xl font-bold mb-6">💬 Translator and Language Detector API Playground</h1>

    <div v-if="!isSupported" class="text-red-600 mb-4">
      Your browser doesn't support the Translator or Language Detector APIs. If you're in Chrome, join the
      <a href="https://developer.chrome.com/docs/ai/built-in#get_an_early_preview" class="underline text-blue-500">
        Early Preview Program
      </a>
      to enable it.
    </div>

    <form @submit.prevent="handleTranslate" v-show="formVisible" class="w-full max-w-md space-y-4">
      <div>
        <label for="input" class="block text-lg font-semibold">Input:</label>
        <textarea id="input" v-model="inputText" class="w-full p-2 border rounded"></textarea>
      </div>

      <p v-if="detectedLanguage" class="text-sm text-gray-600">
        I'm {{ detectionConfidence }}% sure that this is {{ detectedLanguage }}
      </p>

      <div v-if="isTranslatorAvailable">
        <label for="translate" class="block text-lg font-semibold">Translate to:</label>
        <select id="translate" v-model="targetLanguage" class="w-full p-2 border rounded">
          <option value="en">English</option>
          <option value="ja">Japanese</option>
          <option value="es">Spanish</option>
        </select>
        <button type="submit" class="w-full bg-blue-500 text-white p-2 rounded mt-4">Translate</button>
      </div>
    </form>

    <div v-if="outputText" class="mt-6 w-full max-w-md">
      <label for="output" class="block text-lg font-semibold">Translation:</label>
      <output id="output" class="w-full p-2 border rounded bg-white text-gray-700">
        {{ outputText }}
      </output>
    </div>

    <footer class="mt-12 text-center text-sm text-gray-600">
      Modified code from <a href="https://github.com/tomayac" class="underline"> @tomayac </a>. Source code on
      <a href="https://github.com/tomayac/translation-language-detection-api-playground" class="underline">GitHub</a>.
    </footer>
  </div>
</template>

<script setup>
import { ref, onMounted, watchEffect } from 'vue'

const inputText = ref('Hello, world!')
const detectedLanguage = ref('')
const detectionConfidence = ref(0)
const outputText = ref('')
const targetLanguage = ref('es')
const isSupported = ref(true)
const formVisible = ref(false)
const isTranslatorAvailable = ref(false)
const detector = ref(null)

// Check if translation API is supported
onMounted(async () => {
  if (!('translation' in self) || !('createDetector' in self.translation)) {
    isSupported.value = false
    return
  }

  try {
    detector.value = await self.translation.createDetector()
    formVisible.value = true
  } catch (err) {
    isSupported.value = false
  }
})

// Detect language on input change
watchEffect(async () => {
  if (!inputText.value.trim()) {
    detectedLanguage.value = 'not sure what language this is'
    detectionConfidence.value = 0
    return
  }
  
  const { detectedLanguage: lang, confidence } = (await detector.value.detect(inputText.value.trim()))[0]
  detectedLanguage.value = languageTagToHumanReadable(lang, 'en')
  detectionConfidence.value = (confidence * 100).toFixed(1)
})

// Convert language tag to human-readable format
const languageTagToHumanReadable = (languageTag, targetLanguage) => {
  const displayNames = new Intl.DisplayNames([targetLanguage], {
    type: 'language',
  })
  return displayNames.of(languageTag)
}

// Handle translation
const handleTranslate = async () => {
  try {
    const sourceLanguage = (await detector.value.detect(inputText.value.trim()))[0].detectedLanguage
    if (!['en', 'ja', 'es'].includes(sourceLanguage)) {
      outputText.value = 'Currently, only English ↔ Spanish and English ↔ Japanese are supported.'
      return
    }
    
    const translator = await self.translation.createTranslator({
      sourceLanguage,
      targetLanguage: targetLanguage.value,
    })
    
    outputText.value = await translator.translate(inputText.value.trim())
  } catch (err) {
    outputText.value = 'An error occurred. Please try again.'
    console.error(err)
  }
}
</script>

<style scoped>
/* Scoped styling if necessary */
</style>
