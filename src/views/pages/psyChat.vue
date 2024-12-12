<template>
  <div class="ai-chat">
    <div class="chat-history" ref="chatHistoryRef">
      <div
          v-for="(message, index) in chatHistory"
          :key="index"
          :class="['message', message.sender]"
      >
        <p>{{ message.content }}</p>
      </div>
    </div>
    <div class="input-container">
      <input
          v-model="userInput"
          @keyup.enter="sendMessage"
          placeholder="Please enter your question..."
      />
      <button @click="sendMessage">Send</button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue';
import axios from 'axios';

const userInput = ref('');
const chatHistory = ref([]); // Empty initial chat history

// Predefined random responses for AI
const randomContents = [
  "Did you know? I can help you with all kinds of questions!",
  "Hello! How can I assist you today?",
  "Hi there! Feel free to ask me anything.",
  "Greetings! I'm here to answer your questions.",
  "Hello! Ready to chat?"
];

// Psychological test question list (GAD-7)
const questions = [
  "Over the last two weeks, how often have you been bothered by the following problem: Feeling nervous, anxious, or on edge? (0-3)",
  "Over the last two weeks, how often have you been bothered by the following problem: Not being able to stop or control worrying? (0-3)",
  "Over the last two weeks, how often have you been bothered by the following problem: Worrying too much about different things? (0-3)",
  "Over the last two weeks, how often have you been bothered by the following problem: Trouble relaxing? (0-3)",
  "Over the last two weeks, how often have you been bothered by the following problem: Being so restless that it's hard to sit still? (0-3)",
  "Over the last two weeks, how often have you been bothered by the following problem: Becoming easily annoyed or irritable? (0-3)",
  "Over the last two weeks, how often have you been bothered by the following problem: Feeling afraid as if something awful might happen? (0-3)"
];

// Response options
const responseOptions = "Please answer with a number between 0 and 3, where:\n0 = Not at all\n1 = Several days\n2 = More than half the days\n3 = Nearly every day";

let currentQuestionIndex = -1; // Psychological test current question index
let inPsychTestMode = ref(false); // Psychological test mode status
const userResponses = ref([]); // User's response record

const chatHistoryRef = ref(null);

// Function to get a random message from predefined responses
function getRandomContent() {
  const randomIndex = Math.floor(Math.random() * randomContents.length);
  return randomContents[randomIndex];
}

// Function to send a random AI message
function sendRandomMessage() {
  const randomMessage = getRandomContent();
  chatHistory.value.push({
    content: randomMessage,
    sender: 'ai'
  });
  scrollToBottom();
}

// Display a random message only once when the component loads
onMounted(() => {
  sendRandomMessage();
});

// Function to scroll to bottom
function scrollToBottom() {
  nextTick(() => {
    if (chatHistoryRef.value) {
      chatHistoryRef.value.scrollTop = chatHistoryRef.value.scrollHeight;
    }
  });
}

// API configuration
const API_KEY = 'sk-q9f1xoMC3y2Up7WkZC7H9k3OAEHCZ6ez08qWbNmz5BDIfR79'; // Replace with your actual API key
const API_URL = 'https://zzzzapi.com/v1/chat/completions'; // Updated API endpoint

const sendMessage = async () => {
  if (userInput.value.trim() === '') return;

  // Add user message to chat history
  chatHistory.value.push({sender: 'user', content: userInput.value});
  scrollToBottom();

  // Check if the user wants to start the psychological test
  if (userInput.value.trim().toLowerCase() === "start psychological test") {
    inPsychTestMode.value = true;
    currentQuestionIndex = 0;
    chatHistory.value.push({
      content: "Great! Let's begin the psychological self-assessment.\n" + responseOptions,
      sender: 'ai'
    });
    scrollToBottom();
    askNextQuestion();
  } else if (inPsychTestMode.value) {
    // Record the user's response to the psychological test and continue to the next question
    recordPsychTestResponse(userInput.value);
  } else {
    // Normal AI conversation mode
    await generateAIResponse(userInput.value);
  }

  // Clear the input field
  userInput.value = '';
};

// Generate AI response
const generateAIResponse = async (message) => {
  try {
    const response = await processMessageToChatGPT(message);
    const content = response?.data?.choices[0]?.message?.content;
    if (content) {
      chatHistory.value.push({sender: 'ai', content: content});
    } else {
      sendRandomMessage();
    }
    scrollToBottom();
  } catch (error) {
    console.error('Error processing message:', error);
    chatHistory.value.push({
      sender: 'ai',
      content: 'Sorry, there was an error with the AI service.',
    });
    scrollToBottom();
  }
};

// Send message to ChatGPT API
const processMessageToChatGPT = async (message) => {
  const apiRequestBody = {
    model: 'gpt-3.5-turbo',
    messages: [
      {role: 'system', content: "You are a learning assistant helping students with their questions."},
      {role: 'user', content: message}
    ],
  };

  return axios.post(API_URL, apiRequestBody, {
    headers: {
      'Authorization': `Bearer ${API_KEY}`,
      'Content-Type': 'application/json',
    }
  });
};

// Ask the next psychological test question
function askNextQuestion() {
  if (currentQuestionIndex < questions.length) {
    chatHistory.value.push({
      content: questions[currentQuestionIndex],
      sender: 'ai'
    });
    scrollToBottom();
  } else {
    // Test finished, provide feedback
    givePsychTestFeedback();
    inPsychTestMode.value = false;
    currentQuestionIndex = -1;
    userResponses.value = [];
  }
}

// Record the user's response to the psychological test
function recordPsychTestResponse(response) {
  const score = parseInt(response);
  if (isNaN(score) || score < 0 || score > 3) {
    chatHistory.value.push({
      sender: 'ai',
      content: 'Invalid input. ' + responseOptions
    });
    scrollToBottom();
    return;
  }
  userResponses.value.push(score);
  currentQuestionIndex++;
  askNextQuestion();
}

// Provide psychological test feedback
function givePsychTestFeedback() {
  const totalScore = userResponses.value.reduce((acc, val) => acc + val, 0);

  let feedback = `Your total score is ${totalScore}. `;
  feedback += "Please note that this is a simple self-assessment and not a diagnosis. If you have concerns about your mental health, consider reaching out to a mental health professional for further evaluation.";

  chatHistory.value.push({
    content: feedback,
    sender: 'ai'
  });
  scrollToBottom();
}
</script>

<style scoped>
.ai-chat {
  display: flex;
  flex-direction: column;
  max-width: 600px;
  height: 80vh;
  margin: 0 auto;
  padding: 20px;
  border-radius: 10px;
  background-color: #f0f2f5;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

.chat-history {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow-y: auto;
  margin-bottom: 15px;
  padding: 20px;
  border-radius: 10px;
  background-color: #ffffff;
  box-shadow: inset 0 0 5px rgba(0, 0, 0, 0.05);
}

.message {
  max-width: 75%;
  padding: 12px 18px;
  margin: 8px 0;
  border-radius: 20px;
  font-size: 15px;
  line-height: 1.6;
  word-wrap: break-word;
}

.user {
  align-self: flex-end;
  background-color: #0084ff;
  color: #ffffff;
  text-align: right;
  border-bottom-right-radius: 4px;
}

.ai {
  align-self: flex-start;
  background-color: #e5e5ea;
  color: #000000;
  text-align: left;
  border-bottom-left-radius: 4px;
}

.input-container {
  display: flex;
  align-items: center;
  padding: 10px;
  background-color: #ffffff;
  border-top: 1px solid #ccc;
}

input {
  flex: 1;
  padding: 12px;
  font-size: 16px;
  border: 1px solid #ccc;
  border-radius: 20px;
  margin-right: 10px;
}

button {
  padding: 12px 20px;
  font-size: 16px;
  color: #ffffff;
  background-color: #0084ff;
  border: none;
  border-radius: 20px;
  cursor: pointer;
  transition: background-color 0.3s;
}

button:hover {
  background-color: #006bb3;
}

button:disabled {
  background-color: #a0cfff;
  cursor: not-allowed;
}
</style>
