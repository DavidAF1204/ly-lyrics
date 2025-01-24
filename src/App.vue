<template>
  <div class="game-container" :style="backgroundStyle">
    <div class="content">
      <img :src="currentIcon" class="song-icon" alt="Song icon" />
      <div class="stanza-number">第{{ currentStanzaNumber }}節</div>
      <div class="lyrics-container">
        <div class="lyrics">
          <template v-for="(line, lineIndex) in currentStanza" :key="lineIndex">
            <div class="line">
              <template v-for="(phrase, phraseIndex) in splitLine(line)" :key="`${lineIndex}-${phraseIndex}`">
                <span v-if="lineIndex === missingLineIndex && phraseIndex === missingPhraseIndex">
                  <div class="input-container">
                    <input 
                      v-model="userInput"
                      @keyup.enter="checkAnswer"
                      class="answer-input"
                      :class="{ 'incorrect': showIncorrect, 'correct': showCorrect }"
                      placeholder="輸入歌詞"
                    />
                    <div v-if="showIncorrect" class="feedback-message incorrect-message">錯誤</div>
                    <div v-if="showCorrect" class="feedback-message correct-message">正確</div>
                  </div>
                </span>
                <span v-else class="phrase">{{ phrase }}</span>
              </template>
            </div>
          </template>
        </div>
      </div>
      <div class="moov-text">歌詞參照於MOOV平台</div>
      <button @click="selectRandomStanza" class="skip-button">Skip</button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'

const lyrics = ref({})
const icons = ref({})
const currentSong = ref('')
const currentStanza = ref([])
const userInput = ref('')
const missingLineIndex = ref(0)
const missingPhraseIndex = ref(0)
const showIncorrect = ref(false)
const showCorrect = ref(false)
const currentStanzaNumber = ref(1)

const backgroundStyle = computed(() => ({
  backgroundImage: `url(${currentIcon.value})`,
}))

const currentIcon = computed(() => {
  return icons.value[currentSong.value]
})

function splitLine(line) {
  return line.trim().split(' ').filter(phrase => phrase.length > 0)
}

function hasNonParentheticalPhrase(line) {
  // Check if the line has at least one phrase without any parentheses
  const phrases = line.trim().split(' ').filter(phrase => phrase.length > 0)
  return phrases.some(phrase => !phrase.includes('(') && !phrase.includes(')'))
}

function isValidStanza(stanza) {
  return stanza.length > 0 && stanza.some(line => {
    // Check if at least one line has a non-parenthetical phrase
    return hasNonParentheticalPhrase(line)
  })
}

async function loadResources() {
  const lyricsModules = import.meta.glob('/public/lyrics/*.txt')
  for (const path in lyricsModules) {
    const songName = path.split('/').pop().replace('.txt', '')
    const content = await fetch(path.replace('/public', '')).then(res => res.text())
    
    const stanzas = content
      .split(/\n\s*\n+/)
      .map(stanza => stanza.split('\n').filter(line => line.trim()))
      .filter(stanza => isValidStanza(stanza))
    
    lyrics.value[songName] = stanzas
  }

  const iconModules = import.meta.glob('/public/icons/*.jpg', { eager: true })
  for (const path in iconModules) {
    const songName = path.split('/').pop().replace('.jpg', '')
    icons.value[songName] = path.replace('/public', '')
  }
}

function selectRandomStanza() {
  showIncorrect.value = false
  showCorrect.value = false
  const songNames = Object.keys(lyrics.value)
  currentSong.value = songNames[Math.floor(Math.random() * songNames.length)]
  const stanzas = lyrics.value[currentSong.value]
  const randomStanzaIndex = Math.floor(Math.random() * stanzas.length)
  currentStanza.value = stanzas[randomStanzaIndex]
  currentStanzaNumber.value = randomStanzaIndex + 1
  
  // Two-phase approach for selecting a line:
  // 1. Try random selection first (up to 50 attempts)
  // 2. If random fails, fall back to sequential scanning
  let validLineFound = false
  let attempts = 0
  const maxAttempts = 50

  while (!validLineFound && attempts < maxAttempts) {
    missingLineIndex.value = Math.floor(Math.random() * currentStanza.value.length)
    const line = currentStanza.value[missingLineIndex.value]
    const phrases = line.trim().split(' ').filter(phrase => phrase.length > 0)
    
    // Filter out positions of phrases with any parentheses
    const validPhraseIndices = phrases
      .map((phrase, index) => (!phrase.includes('(') && !phrase.includes(')') ? index : -1))
      .filter(index => index !== -1)
    
    if (validPhraseIndices.length > 0) {
      // Select a random index from valid positions
      const randomValidIndex = Math.floor(Math.random() * validPhraseIndices.length)
      missingPhraseIndex.value = validPhraseIndices[randomValidIndex]
      validLineFound = true
    }

    attempts++
  }

  if (!validLineFound) {
    for (let i = 0; i < currentStanza.value.length; i++) {
      const phrases = currentStanza.value[i].trim().split(' ').filter(phrase => phrase.length > 0)
      const validPhraseIndices = phrases
        .map((phrase, index) => (!phrase.includes('(') && !phrase.includes(')') ? index : -1))
        .filter(index => index !== -1)
      
      if (validPhraseIndices.length > 0) {
        missingLineIndex.value = i
        missingPhraseIndex.value = validPhraseIndices[0]
        break
      }
    }
  }
  
  userInput.value = ''
}

function checkAnswer() {
  const correctPhrase = splitLine(currentStanza.value[missingLineIndex.value])[missingPhraseIndex.value]
  
  if (userInput.value.trim().toLowerCase() === correctPhrase.toLowerCase()) {
    showIncorrect.value = false
    showCorrect.value = true
    setTimeout(() => {
      showCorrect.value = false
      selectRandomStanza()
    }, 1000)
  } else {
    showCorrect.value = false
    showIncorrect.value = true
    setTimeout(() => {
      showIncorrect.value = false
    }, 1000)
  }
}

onMounted(async () => {
  await loadResources()
  selectRandomStanza()
})
</script>