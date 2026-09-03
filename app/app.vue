<template>
  <div class="game-wrapper">
    <div class="game-container">
      <h1 class="title">Wordle<span>Pro</span></h1>
      
      <div class="legend">
        <div class="legend-item">
          <div class="legend-box correct"></div>
          <span>Correct Position</span>
        </div>
        <div class="legend-item">
          <div class="legend-box present"></div>
          <span>Wrong Position</span>
        </div>
        <div class="legend-item">
          <div class="legend-box absent"></div>
          <span>Not in Word</span>
        </div>
      </div>

      <div class="grid">
        <div v-for="(row, rIndex) in grid" :key="rIndex" class="row">
          <input
            v-for="(cell, cIndex) in row"
            :key="cIndex"
            :id="`input-${rIndex}-${cIndex}`"
            v-model="cell.letter"
            type="text"
            maxlength="1"
            :disabled="rIndex !== currentRow || gameStatus !== 'playing'"
            :class="['cell', cell.status, { active: rIndex === currentRow }]"
            @input="handleInput(rIndex, cIndex, $event)"
            @keydown="handleKeydown(rIndex, cIndex, $event)"
            @focus="handleFocus(rIndex, cIndex)"
          />
        </div>
      </div>

      <div class="keyboard">
        <div v-for="(kRow, i) in keyboardRows" :key="i" class="keyboard-row">
          <button 
            v-for="key in kRow" 
            :key="key" 
            :class="['key-btn', keyStatus[key] || '', { 'wide-key': key === 'Enter' || key === 'Backspace' }]"
            @click="handleKeyClick(key)"
          >
            <span v-if="key === 'Backspace'">⌫</span>
            <span v-else>{{ key }}</span>
          </button>
        </div>
      </div>

      <div class="message-overlay" v-if="gameStatus !== 'playing'">
        <div class="message-card">
          <h2 v-if="gameStatus === 'won'" class="win-text">🎉 Brilliant!</h2>
          <div v-else class="loss-text">
            <h2>Game Over</h2>
            <p>The word was: <strong>{{ targetWord }}</strong></p>
          </div>
          <button class="play-again-btn" @click="resetGame">Play Again</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { ref, onMounted, nextTick } from 'vue';

const rows = 6;
const cols = 5;

// Initial empty grid state
const getEmptyGrid = () => Array.from({ length: rows }, () =>
  Array.from({ length: cols }, () => ({
    letter: '',
    status: '' // 'correct', 'present', 'absent', or ''
  }))
);

const grid = ref(getEmptyGrid());
const currentRow = ref(0);
const targetWord = ref('');
const gameStatus = ref('playing');

// Keyboard State
const keyboardRows = [
  ['Q', 'W', 'E', 'R', 'T', 'Y', 'U', 'I', 'O', 'P'],
  ['A', 'S', 'D', 'F', 'G', 'H', 'J', 'K', 'L'],
  ['Enter', 'Z', 'X', 'C', 'V', 'B', 'N', 'M', 'Backspace']
];
const keyStatus = ref<Record<string, string>>({});

// Keep track of the last focused column in the current row
const focusedCol = ref(0);

const fetchWord = async () => {
  try {
    const res = await fetch('https://random-word-api.herokuapp.com/word?length=5');
    const data = await res.json();
    targetWord.value = data[0].toUpperCase();
  } catch (e) {
    console.error("Failed to fetch word:", e);
    targetWord.value = 'APPLE'; // Fallback
  }
};

onMounted(async () => {
  await fetchWord();
  focusInput(0, 0);
});

const handleFocus = (rIndex: number, cIndex: number) => {
  if (rIndex === currentRow.value) {
    focusedCol.value = cIndex;
  }
};

const handleInput = (rIndex: number, cIndex: number, event: Event) => {
  const value = (event.target as HTMLInputElement).value;
  
  // Only accept alphabetical characters
  if (!/^[a-zA-Z]$/.test(value)) {
    grid.value[rIndex][cIndex].letter = '';
    return;
  }
  
  grid.value[rIndex][cIndex].letter = value.toUpperCase();
  
  // Focus next input in the row
  if (cIndex < cols - 1) {
    focusInput(rIndex, cIndex + 1);
  }
};

const handleKeydown = (rIndex: number, cIndex: number, event: KeyboardEvent) => {
  if (event.key === 'Backspace' && !grid.value[rIndex][cIndex].letter && cIndex > 0) {
    focusInput(rIndex, cIndex - 1);
  } else if (event.key === 'Enter') {
    submitGuess();
  } else if (event.key === 'ArrowLeft' && cIndex > 0) {
    focusInput(rIndex, cIndex - 1);
  } else if (event.key === 'ArrowRight' && cIndex < cols - 1) {
    focusInput(rIndex, cIndex + 1);
  }
};

const handleKeyClick = (key: string) => {
  if (gameStatus.value !== 'playing') return;
  
  const row = grid.value[currentRow.value];
  
  if (key === 'Backspace') {
    // If current focused cell has a letter, delete it
    if (row[focusedCol.value].letter) {
      row[focusedCol.value].letter = '';
      focusInput(currentRow.value, focusedCol.value);
    } 
    // Else if we can go back, go back and delete
    else if (focusedCol.value > 0) {
      row[focusedCol.value - 1].letter = '';
      focusInput(currentRow.value, focusedCol.value - 1);
    }
  } else if (key === 'Enter') {
    submitGuess();
  } else {
    // Type the letter in the current focused cell
    row[focusedCol.value].letter = key;
    // Move to next if possible
    if (focusedCol.value < cols - 1) {
      focusInput(currentRow.value, focusedCol.value + 1);
    } else {
      // Keep focus on the last cell so you can see what was typed
      focusInput(currentRow.value, focusedCol.value);
    }
  }
};

const focusInput = (rIndex: number, cIndex: number) => {
  focusedCol.value = cIndex;
  nextTick(() => {
    const el = document.getElementById(`input-${rIndex}-${cIndex}`);
    if (el) {
      el.focus();
    }
  });
};

const submitGuess = () => {
  const row = grid.value[currentRow.value];
  const guess = row.map(c => c.letter).join('');
  
  if (guess.length !== cols) {
    // Shake animation could be triggered here
    return;
  }
  
  // Validation logic
  const targetChars = targetWord.value.split('');
  
  // Pass 1: correct positions (Green)
  row.forEach((cell, i) => {
    if (cell.letter === targetChars[i]) {
      cell.status = 'correct';
      targetChars[i] = null; // Consume letter
    }
  });
  
  // Pass 2: present but wrong position (Yellow)
  row.forEach((cell) => {
    if (cell.status !== 'correct') {
      const idx = targetChars.indexOf(cell.letter);
      if (idx !== -1) {
        cell.status = 'present';
        targetChars[idx] = null; // Consume letter
      } else {
        cell.status = 'absent';
      }
    }
  });
  
  // Update keyboard state
  row.forEach((cell) => {
    const currentStatus = keyStatus.value[cell.letter];
    if (cell.status === 'correct') {
      keyStatus.value[cell.letter] = 'correct';
    } else if (cell.status === 'present' && currentStatus !== 'correct') {
      keyStatus.value[cell.letter] = 'present';
    } else if (cell.status === 'absent' && currentStatus !== 'correct' && currentStatus !== 'present') {
      keyStatus.value[cell.letter] = 'absent';
    }
  });
  
  // Check win/loss state
  if (guess === targetWord.value) {
    gameStatus.value = 'won';
  } else if (currentRow.value === rows - 1) {
    gameStatus.value = 'lost';
  } else {
    currentRow.value++;
    focusInput(currentRow.value, 0);
  }
};

const resetGame = async () => {
  grid.value = getEmptyGrid();
  keyStatus.value = {};
  currentRow.value = 0;
  focusedCol.value = 0;
  gameStatus.value = 'playing';
  await fetchWord();
  focusInput(0, 0);
};
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Outfit:wght@400;600;800&display=swap');

body {
  margin: 0;
  background: #f1f5f9; /* Light gray background */
  color: #0f172a;      /* Dark slate text */
  font-family: 'Outfit', sans-serif;
  min-height: 100vh;
}

.game-wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  padding: 2rem;
}

.game-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  background: #ffffff; /* White card */
  border: 1px solid #e2e8f0;
  padding: 3rem;
  border-radius: 24px;
  box-shadow: 0 10px 25px -5px rgba(0, 0, 0, 0.05), 0 8px 10px -6px rgba(0, 0, 0, 0.01);
  position: relative;
  width: 100%;
  max-width: 500px;
}

.title {
  margin-top: 0;
  margin-bottom: 2rem;
  font-size: 2.8rem;
  font-weight: 800;
  letter-spacing: 1px;
  color: #1e293b;
  text-align: center;
}

.title span {
  font-weight: 400;
  color: #94a3b8;
}

.legend {
  display: flex;
  gap: 1.5rem;
  margin-bottom: 2rem;
  justify-content: center;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.9rem;
  color: #64748b;
  font-weight: 600;
}

.legend-box {
  width: 16px;
  height: 16px;
  border-radius: 4px;
}

.legend-box.correct {
  background: #22c55e;
}

.legend-box.present {
  background: #eab308;
}

.legend-box.absent {
  background: #94a3b8;
}

.grid {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 2rem;
}

.row {
  display: flex;
  gap: 8px;
  justify-content: center;
}

.cell {
  width: 60px;
  height: 60px;
  border: 2px solid #cbd5e1; /* Light gray border */
  background: #f8fafc;
  border-radius: 12px;
  color: #0f172a;
  font-size: 2.2rem;
  text-align: center;
  text-transform: uppercase;
  font-weight: 800;
  font-family: 'Outfit', sans-serif;
  box-sizing: border-box;
  caret-color: transparent;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.cell:focus {
  outline: none;
  border-color: #64748b;
  background: #ffffff;
  transform: translateY(-2px);
  box-shadow: 0 10px 15px -3px rgba(100, 116, 139, 0.1);
}

.cell.active {
  border-color: #94a3b8;
}

.cell.correct {
  background: #22c55e;
  border-color: #22c55e;
  color: white;
  animation: popIn 0.5s ease;
}

.cell.present {
  background: #eab308;
  border-color: #eab308;
  color: white;
  animation: popIn 0.5s ease;
}

.cell.absent {
  background: #94a3b8;
  border-color: #94a3b8;
  color: white;
  animation: popIn 0.5s ease;
}

.cell:disabled {
  opacity: 1;
  cursor: default;
}

/* Keyboard Styles */
.keyboard {
  display: flex;
  flex-direction: column;
  gap: 6px;
  width: 100%;
}

.keyboard-row {
  display: flex;
  justify-content: center;
  gap: 6px;
}

.key-btn {
  flex: 1;
  height: 52px;
  border-radius: 6px;
  border: none;
  background: #e2e8f0;
  color: #1e293b;
  font-size: 1.1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s, transform 0.1s, color 0.2s;
  font-family: 'Outfit', sans-serif;
  padding: 0;
  display: flex;
  justify-content: center;
  align-items: center;
}

.key-btn:hover {
  background: #cbd5e1;
}

.key-btn.wide-key {
  flex: 1.5;
  font-size: 0.9rem;
}

.key-btn:active {
  transform: scale(0.95);
}

/* Keyboard States */
.key-btn.correct { background: #22c55e; color: white; }
.key-btn.present { background: #eab308; color: white; }
.key-btn.absent { background: #94a3b8; color: white; }

.key-btn.correct:hover { background: #16a34a; }
.key-btn.present:hover { background: #ca8a04; }
.key-btn.absent:hover { background: #64748b; }

/* Overlays */
.message-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(255, 255, 255, 0.85);
  backdrop-filter: blur(4px);
  border-radius: 24px;
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 10;
  animation: fadeIn 0.4s ease;
}

.message-card {
  background: #ffffff;
  border: 1px solid #e2e8f0;
  padding: 3rem;
  border-radius: 20px;
  text-align: center;
  box-shadow: 0 20px 40px rgba(0,0,0,0.1);
  transform: scale(0.9);
  animation: scaleUp 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275) forwards;
}

.win-text {
  font-size: 2.5rem;
  color: #10b981;
  margin-bottom: 2rem;
  margin-top: 0;
}

.loss-text h2 {
  font-size: 2.5rem;
  color: #ef4444;
  margin-top: 0;
  margin-bottom: 0.5rem;
}

.loss-text p {
  font-size: 1.2rem;
  color: #64748b;
  margin-bottom: 2rem;
}

.loss-text strong {
  color: #0f172a;
  font-weight: 800;
  letter-spacing: 2px;
}

.play-again-btn {
  background: #3b82f6;
  color: white;
  border: none;
  padding: 14px 32px;
  font-size: 1.2rem;
  font-weight: 600;
  font-family: 'Outfit', sans-serif;
  border-radius: 9999px;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 4px 14px rgba(59, 130, 246, 0.3);
}

.play-again-btn:hover {
  background: #2563eb;
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(59, 130, 246, 0.4);
}

.play-again-btn:active {
  transform: translateY(1px);
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes scaleUp {
  from { transform: scale(0.8); opacity: 0; }
  to { transform: scale(1); opacity: 1; }
}

@keyframes popIn {
  0% { transform: scale(1); }
  50% { transform: scale(1.1); }
  100% { transform: scale(1); }
}
</style>
