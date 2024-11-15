<template>
  <div class="min-h-screen bg-gray-100 flex flex-col items-center justify-center p-4">
    <div class="w-full max-w-2xl bg-white shadow-md rounded-lg p-6">
      <h1 class="text-2xl font-bold mb-4">Interactive AI Chat</h1>
      
      <!-- Conversation Display -->
      <div class="mb-4 border rounded-md p-4 bg-gray-50 max-h-64 overflow-y-auto">
        <p class="text-sm text-gray-600 mb-2">Conversation:</p>
        <div v-html="formattedConversation" class="prose prose-sm"></div>
      </div>

      <!-- User Input -->
      <form @submit.prevent="sendMessage" class="flex items-center mb-4">
        <input
          v-model="userMessage"
          type="text"
          placeholder="Type your message..."
          class="flex-1 border rounded-md p-2 focus:outline-none focus:ring focus:ring-blue-300"
        />
        <button
          type="submit"
          class="ml-2 bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600"
          :disabled="loading || !userMessage.trim()"
        >
          Send
        </button>
      </form>

      <!-- Clear Button -->
      <button
        @click="clearConversation"
        class="w-full bg-red-500 text-white px-4 py-2 rounded hover:bg-red-600"
        :disabled="loading"
      >
        Clear Conversation
      </button>

      <div v-if="loading" class="text-blue-500 mt-2">AI is responding...</div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, computed } from 'vue';
import { marked } from 'marked';

let session = null; // Session object
const loading = ref(false);
const userMessage = ref('');
const conversation = reactive([]); // Conversation log

// Computed property to render markdown for the conversation
const formattedConversation = computed(() =>
  conversation
    .map((entry) =>
      entry.from === 'user'
        ? `<p class="text-blue-700"><strong>You:</strong> ${entry.text}</p>`
        : `<p class="text-gray-800"><strong>AI:</strong> ${marked(entry.text)}</p>`
    )
    .join('')
);

const sendMessage = async () => {
  if (!userMessage.value.trim()) return;

  // Add user message to the conversation
  conversation.push({ from: 'user', text: userMessage.value });

  // Clear input
  const promptText = userMessage.value;
  userMessage.value = '';
  loading.value = true;

  try {
    // Initialize session if it doesn't exist
    if (!session) {
      const { available } = await ai.languageModel.capabilities();
      if (available === 'no') {
        throw new Error('Language model is not available.');
      }
      session = await ai.languageModel.create();
    }

    // Send user message to the AI and stream its response
    const stream = session.promptStreaming(promptText);

    let result = '';
    let previousChunk = '';
    for await (const chunk of stream) {
      const newChunk = chunk.startsWith(previousChunk)
        ? chunk.slice(previousChunk.length)
        : chunk;
      result += newChunk;
      previousChunk = chunk;

      // Append AI's response chunk by chunk
      const aiEntry = conversation.find((entry) => entry.from === 'ai' && entry.streaming);
      if (aiEntry) {
        aiEntry.text = result; // Update streaming AI response
      } else {
        conversation.push({ from: 'ai', text: result, streaming: true });
      }
    }

    // Mark AI response as complete
    const aiEntry = conversation.find((entry) => entry.from === 'ai' && entry.streaming);
    if (aiEntry) {
      aiEntry.streaming = false;
    }
  } catch (error) {
    console.error('Error during AI streaming:', error);
    conversation.push({ from: 'ai', text: 'An error occurred. Please try again.' });
  } finally {
    loading.value = false;
  }
};

const clearConversation = async () => {
  if (session) {
    await session.destroy(); // Clear the current session
    session = null; // Reset session
  }
  conversation.splice(0); // Clear the conversation array
};
</script>
