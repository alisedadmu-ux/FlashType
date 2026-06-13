<template>
  <div class="monkeytype-container" @click="focusEngine">
    <header class="app-header">
      <div class="logo">
        <span class="logo-accent">⚡</span> flashType
      </div>
      
      <div class="controls-wrapper" v-if="gameState === 'idle' || gameState === 'typing'">
        <div class="selector-group">
          <span class="selector-label">difficulty:</span>
          <button 
            v-for="diff in ['easy', 'medium', 'hard']" 
            :key="diff"
            :class="{ active: currentDifficulty === diff }"
            @click="changeDifficulty(diff)"
            :disabled="gameState === 'typing'"
          >
            {{ diff }}
          </button>
        </div>

        <div class="selector-group">
          <span class="selector-label">time:</span>
          <button 
            v-for="timeOption in timeConfigs" 
            :key="timeOption.seconds"
            :class="{ active: maxTime === timeOption.seconds }"
            @click="changeTimeMode(timeOption.seconds)"
            :disabled="gameState === 'typing'"
          >
            {{ timeOption.label }}
          </button>
        </div>
      </div>
    </header>

    <div v-if="gameState !== 'results'" class="test-view">
      <div class="test-meta">
        <div class="live-timer" :class="{ active: gameState === 'typing' }">
          {{ formattedTimeLeft }}
        </div>
        <div class="test-tracker">
          Test {{ currentTestNumber }} of 5 
          <span class="dots">
            <span 
              v-for="n in 5" 
              :key="n" 
              class="dot" 
              :class="{ active: n <= currentTestNumber, current: n === currentTestNumber }"
            ></span>
          </span>
        </div>
      </div>

      <div class="words-wrapper">
        <input 
          ref="hiddenInput"
          v-model="userInput"
          type="text"
          class="hidden-input"
          @input="processInput"
          @blur="handleBlur"
          @focus="handleFocus"
          :disabled="timeLeft === 0"
        />
        
        <div class="words-container">
          <div 
            v-for="(word, wIdx) in targetWords" 
            :key="wIdx" 
            class="word-block"
          >
            <span 
              v-for="(char, cIdx) in word" 
              :key="cIdx"
              :class="getLetterClass(wIdx, cIdx)"
              class="letter"
            >
              <span v-if="isCaretHere(wIdx, cIdx)" class="custom-caret"></span>
              {{ char }}
            </span>
            <span v-if="isCaretAtSpace(wIdx)" class="space-caret-container">
              <span class="custom-caret"></span>
            </span>
          </div>
        </div>
      </div>

      <div class="focus-notice" v-if="!isInputFocused">
        <span class="icon">🖱️</span> Click anywhere on the screen to focus and type
      </div>
    </div>

    <div v-else class="results-view">
      <h2 class="section-title">
        {{ currentTestNumber === 5 ? '🎉 Difficulty Tier Completed!' : 'Test Results' }}
      </h2>
      
      <div class="performance-grid">
        <div class="metric-big">
          <span class="num">{{ finalWpm }}</span>
          <span class="lbl">wpm</span>
        </div>
        <div class="metric-big">
          <span class="num">{{ finalAccuracy }}%</span>
          <span class="lbl">accuracy</span>
        </div>
        
        <div class="diagnostic-card">
          <h3>🔧 Tier Progress Insights</h3>
          <p v-if="currentTestNumber === 5">
            Amazing job! You have conquered all 5 tests on <strong>{{ currentDifficulty.toUpperCase() }}</strong> mode. Try moving up a tier or resetting to beat your speed scores!
          </p>
          <p v-else>
            Good run on Test {{ currentTestNumber }}! Clicking "Next Test" below will advance you to Test {{ currentTestNumber + 1 }} with a fresh set of specialized {{ currentDifficulty }} words.
          </p>
        </div>
      </div>

      <div class="leaderboard-section">
        <h3>🏆 Personal Leaderboard (Top Runs)</h3>
        <table class="leaderboard-table">
          <thead>
            <tr>
              <th>Rank</th>
              <th>Speed (WPM)</th>
              <th>Accuracy</th>
              <th>Difficulty</th>
              <th>Time Mode</th>
              <th>Date</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(run, index) in leaderboard" :key="index" :class="{ 'top-run': index === 0 }">
              <td>#{{ index + 1 }}</td>
              <td class="wpm-column">{{ run.wpm }} WPM</td>
              <td>{{ run.accuracy }}%</td>
              <td class="diff-tag" :class="run.difficulty">{{ run.difficulty }}</td>
              <td>{{ formatTimeLabel(run.mode) }}</td>
              <td>{{ run.date }}</td>
            </tr>
            <tr v-if="leaderboard.length === 0">
              <td colspan="6" class="empty-lead">No entries logged yet. Finish a test to set a record!</td>
            </tr>
          </tbody>
        </table>
      </div>

      <div class="action-row">
        <button v-if="currentTestNumber < 5" class="action-btn primary-btn" @click="advanceToNextTest">
          Next Test ({{ currentTestNumber + 1 }}/5) ➔
        </button>
        <button class="action-btn secondary-btn" @click="resetDifficultyTier">
          Reset Tier (Test 1)
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick } from 'vue'

const timeConfigs = [
  { label: '15s', seconds: 15 },
  { label: '30s', seconds: 30 },
  { label: '60s', seconds: 60 },
  { label: '2m', seconds: 120 },
  { label: '5m', seconds: 300 }
]

const difficulties = {
  easy: [
    "the", "and", "cat", "dog", "big", "run", "you", "it", "he", "was", "for", "on", "are", 
    "as", "with", "his", "they", "at", "be", "this", "have", "from", "or", "one", "had", "by"
  ],
  medium: [
    "people", "around", "system", "through", "because", "another", "problem", "against", "without",
    "country", "general", "program", "question", "company", "business", "product", "however", "provide"
  ],
  hard: [
    "const", "function()", "let", "array.length;", "interface", "export", "async/await", "return",
    "development", "synchronous", "environment", "optimization", "infrastructure", "architecture"
  ]
}

const currentDifficulty = ref("easy")
const currentTestNumber = ref(1) 

const targetWords = ref([])
const userInput = ref("")
const maxTime = ref(30)
const timeLeft = ref(30)
const gameState = ref("idle")
const isInputFocused = ref(false)

const finalWpm = ref(0)
const finalAccuracy = ref(0)
const leaderboard = ref([])

let timerInterval = null
const hiddenInput = ref(null)

const fullTargetString = computed(() => {
  return targetWords.value.join(' ') + ' '
})

const formattedTimeLeft = computed(() => {
  if (timeLeft.value < 60) return `${timeLeft.value}s`
  const minutes = Math.floor(timeLeft.value / 60)
  const seconds = timeLeft.value % 60
  return `${minutes}:${seconds < 10 ? '0' : ''}${seconds}`
})

const formatTimeLabel = (seconds) => {
  const cfg = timeConfigs.find(c => c.seconds === seconds)
  return cfg ? cfg.label : `${seconds}s`
}

const generateText = () => {
  const selected = []
  const pool = difficulties[currentDifficulty.value]
  const count = maxTime.value >= 120 ? 350 : 80
  for (let i = 0; i < count; i++) {
    selected.push(pool[Math.floor(Math.random() * pool.length)])
  }
  targetWords.value = selected
}

const focusEngine = () => {
  if (gameState.value !== 'results') {
    hiddenInput.value?.focus()
    isInputFocused.value = true
  }
}

const changeDifficulty = (newDiff) => {
  currentDifficulty.value = newDiff
  currentTestNumber.value = 1
  resetTest()
}

const changeTimeMode = (newTime) => {
  maxTime.value = newTime
  timeLeft.value = newTime
  resetTest()
}

const getAbsoluteIndex = (wordIdx, charIdx) => {
  let positionCounter = 0
  for (let i = 0; i < wordIdx; i++) {
    positionCounter += targetWords.value[i].length + 1
  }
  return positionCounter + charIdx
}

const isCaretHere = (wordIdx, charIdx) => {
  return getAbsoluteIndex(wordIdx, charIdx) === userInput.value.length
}

const isCaretAtSpace = (wordIdx) => {
  const wordLength = targetWords.value[wordIdx].length
  return getAbsoluteIndex(wordIdx, wordLength) === userInput.value.length
}

const getLetterClass = (wordIdx, charIdx) => {
  const absIdx = getAbsoluteIndex(wordIdx, charIdx)
  if (absIdx >= userInput.value.length) return 'letter-default'
  return userInput.value[absIdx] === fullTargetString.value[absIdx] ? 'letter-correct' : 'letter-wrong'
}

const processInput = () => {
  if (gameState.value === 'idle' && userInput.value.length > 0) {
    gameState.value = 'typing'
    startTimer()
  }

  if (userInput.value.length >= fullTargetString.value.length - 1) {
    generateText()
    userInput.value = ""
  }
}

const startTimer = () => {
  timerInterval = setInterval(() => {
    if (timeLeft.value > 1) {
      timeLeft.value--
    } else {
      timeLeft.value = 0
      clearInterval(timerInterval)
      calculateFinalMetrics()
    }
  }, 1000)
}

const calculateFinalMetrics = () => {
  gameState.value = 'results'
  const totalCharacters = userInput.value.length
  if (totalCharacters === 0) {
    finalWpm.value = 0
    finalAccuracy.value = 100
    return
  }

  let correctHits = 0
  for (let i = 0; i < totalCharacters; i++) {
    if (userInput.value[i] === fullTargetString.value[i]) correctHits++
  }
  finalAccuracy.value = Math.round((correctHits / totalCharacters) * 100)

  const timeInMinutes = maxTime.value / 60
  finalWpm.value = Math.round((totalCharacters / 5) / timeInMinutes)

  saveRunToLeaderboard(finalWpm.value, finalAccuracy.value)
}

const advanceToNextTest = () => {
  if (currentTestNumber.value < 5) {
    currentTestNumber.value++
  }
  resetTest()
}

const resetDifficultyTier = () => {
  currentTestNumber.value = 1
  resetTest()
}

const resetTest = () => {
  clearInterval(timerInterval)
  userInput.value = ""
  timeLeft.value = maxTime.value
  gameState.value = 'idle'
  generateText()
  nextTick(() => focusEngine())
}

const loadLeaderboard = () => {
  const savedData = localStorage.getItem('ft_leaderboard_multi')
  if (savedData) leaderboard.value = JSON.parse(savedData)
}

const saveRunToLeaderboard = (wpmScore, accuracyScore) => {
  const currentRun = {
    wpm: wpmScore,
    accuracy: accuracyScore,
    difficulty: currentDifficulty.value,
    mode: maxTime.value,
    date: new Date().toLocaleDateString()
  }
  leaderboard.value.push(currentRun)
  leaderboard.value.sort((a, b) => b.wpm - a.wpm)
  if (leaderboard.value.length > 5) leaderboard.value = leaderboard.value.slice(0, 5)
  localStorage.setItem('ft_leaderboard_multi', JSON.stringify(leaderboard.value))
}

const handleFocus = () => { isInputFocused.value = true }
const handleBlur = () => { isInputFocused.value = false }

onMounted(() => {
  generateText()
  loadLeaderboard()
  window.addEventListener('focus', handleFocus)
  window.addEventListener('blur', handleBlur)
  nextTick(() => focusEngine())
})

onUnmounted(() => {
  window.removeEventListener('focus', handleFocus)
  window.removeEventListener('blur', handleBlur)
  clearInterval(timerInterval)
})
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Roboto+Mono:wght@400;700&family=Lexend+Deca:wght@300;400;700&display=swap');

.monkeytype-container {
  background-color: #323437;
  color: #e2b714;
  min-height: 100vh;
  padding: 4rem;
  font-family: 'Roboto Mono', monospace;
  width: 100%;
  max-width: 100vw;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
}

.app-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 5rem;
  font-family: 'Lexend Deca', sans-serif;
  max-width: 1100px;
  width: 100%;
  margin-left: auto;
  margin-right: auto;
}

.logo {
  font-size: 1.8rem;
  font-weight: 700;
  color: #d1d0c5;
}
.logo-accent { color: #e2b714; }

.controls-wrapper {
  display: flex;
  gap: 2rem;
}

.selector-group {
  background: #2c2e31;
  padding: 0.4rem 0.8rem;
  border-radius: 8px;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.selector-label {
  color: #646669;
  font-size: 0.85rem;
  font-weight: bold;
  padding-right: 0.25rem;
}

.selector-group button {
  background: none;
  border: none;
  color: #646669;
  font-family: inherit;
  font-size: 0.9rem;
  font-weight: 700;
  padding: 0.3rem 0.6rem;
  cursor: pointer;
  border-radius: 4px;
  transition: all 0.2s ease;
  text-transform: capitalize;
}

.selector-group button.active {
  color: #e2b714;
  background: #323437;
}

.test-view {
  max-width: 1100px;
  width: 100%;
  margin-left: auto;
  margin-right: auto;
}

.test-meta {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
  font-family: 'Lexend Deca', sans-serif;
}

.live-timer {
  font-size: 2.5rem;
  color: #646669;
  font-weight: 700;
  opacity: 0;
  transition: opacity 0.2s ease;
}
.live-timer.active { opacity: 1; color: #e2b714; }

.test-tracker {
  color: #d1d0c5;
  font-size: 1.1rem;
  display: flex;
  align-items: center;
  gap: 1rem;
}

.dots { display: flex; gap: 0.4rem; }
.dot { width: 8px; height: 8px; background: #2c2e31; border-radius: 50%; }
.dot.active { background: #646669; }
.dot.current { background: #e2b714; box-shadow: 0 0 8px #e2b714; }

.words-wrapper {
  position: relative;
  margin-bottom: 3rem;
}

.hidden-input { position: absolute; opacity: 0; z-index: -10; }

.words-container {
  display: flex;
  flex-wrap: wrap;
  font-size: 1.85rem;
  line-height: 1.8;
  user-select: none;
}

/* FIX: Keep complete words intact together, and space out gaps horizontally */
.word-block { 
  display: inline-block; 
  white-space: nowrap;
  margin-right: 1.25rem; 
  position: relative;
}

.letter { position: relative; display: inline-block; }

.letter-default { color: #646669; }
.letter-correct { color: #d1d0c5; }
.letter-wrong { color: #ca4754; background: rgba(202, 71, 84, 0.12); border-radius: 2px; }

.space-caret-container {
  position: absolute;
  right: -0.5rem;
  top: 0;
  width: 0.5rem;
  height: 100%;
}

.custom-caret {
  position: absolute;
  left: 0;
  top: 15%;
  width: 2.5px;
  height: 70%;
  background-color: #e2b714;
  animation: caretBlink 1.2s infinite;
}

@keyframes caretBlink { 0%, 100% { opacity: 1; } 50% { opacity: 0; } }

.focus-notice {
  text-align: center;
  color: #646669;
  margin-top: 5rem;
  font-size: 1rem;
  font-family: 'Lexend Deca', sans-serif;
}

.results-view {
  display: flex;
  flex-direction: column;
  gap: 3rem;
  animation: fadeIn 0.4s ease-out;
  font-family: 'Lexend Deca', sans-serif;
  max-width: 1100px;
  width: 100%;
  margin-left: auto;
  margin-right: auto;
}

@keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }

.section-title {
  color: #646669;
  font-size: 1.6rem;
  border-bottom: 2px solid #2c2e31;
  padding-bottom: 0.75rem;
}

.performance-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(240px, 1fr)); gap: 2rem; }
.metric-big { background: #2c2e31; border-radius: 14px; padding: 2.5rem; display: flex; flex-direction: column; }
.metric-big .num { font-size: 5rem; font-weight: 700; color: #e2b714; line-height: 0.9; }
.metric-big .lbl { font-size: 1.1rem; color: #646669; text-transform: uppercase; margin-top: 0.75rem; font-weight: 700; }

.diagnostic-card { background: #2c2e31; border-left: 5px solid #e2b714; border-radius: 0 14px 14px 0; padding: 2rem; }
.diagnostic-card h3 { margin: 0 0 1rem 0; font-size: 1.3rem; color: #d1d0c5; }
.diagnostic-card p { margin: 0; line-height: 1.7; color: #9a998e; font-size: 1.05rem; }

.leaderboard-section h3 { color: #d1d0c5; margin-bottom: 1.5rem; font-size: 1.4rem; }
.leaderboard-table { width: 100%; border-collapse: collapse; text-align: left; font-size: 1.1rem; }
.leaderboard-table th { color: #646669; padding: 1rem; border-bottom: 3px solid #2c2e31; }
.leaderboard-table td { padding: 1.25rem 1rem; color: #646669; border-bottom: 1px solid #2c2e31; }
.leaderboard-table tr.top-run td { color: #d1d0c5; }
.leaderboard-table tr.top-run .wpm-column { color: #e2b714; font-weight: 700; }

.diff-tag { text-transform: uppercase; font-size: 0.85rem; font-weight: bold; }
.diff-tag.easy { color: #a3be8c; }
.diff-tag.medium { color: #ebcb8b; }
.diff-tag.hard { color: #bf616a; }

.empty-lead { text-align: center; padding: 3rem !important; color: #646669; font-style: italic; }

.action-row { display: flex; justify-content: center; gap: 1.5rem; }
.action-btn {
  font-family: inherit;
  font-size: 1.2rem;
  font-weight: 700;
  padding: 1rem 3rem;
  border-radius: 10px;
  cursor: pointer;
  border: none;
  transition: background 0.2s;
}
.primary-btn { background: #e2b714; color: #323437; }
.primary-btn:hover { background: #f3cb30; }
.secondary-btn { background: #2c2e31; color: #d1d0c5; }
.secondary-btn:hover { background: #3e4145; }
</style>