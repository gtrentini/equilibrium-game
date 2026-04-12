<script setup>
import { ref, computed, watch } from 'vue';

// Load initial state from localStorage if available
const savedLevel = localStorage.getItem('equilibrium_level');
const currentLevel = ref(savedLevel ? parseInt(savedLevel, 10) : 1);

const savedCoins = localStorage.getItem('equilibrium_coins');
const coinBank = ref(savedCoins ? parseInt(savedCoins, 10) : 0);

// Reactive state
const grid = ref([]);
const initialGridState = ref([]);
const initialPlusAmount = ref(0);
const initialMinusAmount = ref(0);
const initialMultiplyAmount = ref(0);
const activeTokenId = ref(null);
const moveHistory = ref([]);
const animatingCells = ref({});
const earnedCoins = ref(0);

// Watch for changes to save to localStorage
watch(currentLevel, (newVal) => {
  localStorage.setItem('equilibrium_level', newVal.toString());
});

watch(coinBank, (newVal) => {
  localStorage.setItem('equilibrium_coins', newVal.toString());
});

// Reactive inventory of playable tokens
const inventory = ref([
  { id: 'plus', name: '+1 Ripple', primary: 2, secondary: 1, amount: 0, unlockLevel: 1, cost: 50 },
  { id: 'minus', name: '-1 Ripple', primary: -2, secondary: -1, amount: 0, unlockLevel: 1, cost: 50 },
  { id: 'multiply', name: 'x Multiplier', primary: 0, secondary: -1, amount: 0, unlockLevel: 75, cost: 100 }
]);

const visibleInventory = computed(() => {
  return inventory.value.filter(t => currentLevel.value >= t.unlockLevel);
});

// Computed property to find the currently active token object
const activeToken = computed(() => {
  return inventory.value.find(t => t.id === activeTokenId.value);
});

// Win state condition: all non-void cells equal 0
const isWin = computed(() => {
  if (grid.value.length === 0) return false;
  return grid.value.every(row => row.every(cell => cell.type === 'void' || cell.value === 0));
});

// Award coins upon winning
watch(isWin, (newVal) => {
  if (newVal) {
    const reward = Math.floor(Math.random() * (25 - 10 + 1)) + 10; // Random 10 to 25 coins
    earnedCoins.value = reward;
    coinBank.value += reward;
  }
});

// Action to select a token from inventory
const selectToken = (id) => {
  activeTokenId.value = id;
};

// Shop Action: Buy a token
const buyToken = (tokenId) => {
  const token = inventory.value.find(t => t.id === tokenId);
  if (token && coinBank.value >= token.cost) {
    coinBank.value -= token.cost;
    token.amount++;
    
    // Add to history so it's not lost on undo
    moveHistory.value.forEach(historyState => {
      const historyToken = historyState.inventory.find(t => t.id === tokenId);
      if (historyToken) {
        historyToken.amount++;
      }
    });
  }
};

// Triggers visual animation class on a cell
const triggerAnimation = (r, c, delay = 0) => {
  const key = `${r}-${c}`;
  setTimeout(() => {
    // Use a unique token to prevent quick successive animations from cancelling each other out
    const token = Date.now() + Math.random();
    animatingCells.value[key] = token;
    setTimeout(() => {
      if (animatingCells.value[key] === token) {
        animatingCells.value[key] = false;
      }
    }, 300); // 300ms matches the CSS animation duration
  }, delay);
};

// Helper to determine all affected cells based on grid mechanics
const getAffectedCells = (gridState, r, c) => {
  const size = gridState.length;
  const targetCell = gridState[r][c];
  
  let primaryTargets = [{ r, c }];
  
  // Quantum mirroring logic
  if (targetCell.type === 'quantum_a') {
    for (let ir = 0; ir < size; ir++) {
      for (let ic = 0; ic < size; ic++) {
        if (gridState[ir][ic].type === 'quantum_b') primaryTargets.push({ r: ir, c: ic });
      }
    }
  } else if (targetCell.type === 'quantum_b') {
    for (let ir = 0; ir < size; ir++) {
      for (let ic = 0; ic < size; ic++) {
        if (gridState[ir][ic].type === 'quantum_a') primaryTargets.push({ r: ir, c: ic });
      }
    }
  }
  
  let secondaryTargets = [];
  primaryTargets.forEach(pt => {
    const adjs = [
      [pt.r - 1, pt.c], [pt.r + 1, pt.c],
      [pt.r, pt.c - 1], [pt.r, pt.c + 1]
    ];
    adjs.forEach(([ar, ac]) => {
      if (ar >= 0 && ar < size && ac >= 0 && ac < size && gridState[ar][ac].type !== 'void') {
        secondaryTargets.push({ r: ar, c: ac });
      }
    });
  });
  
  return { primaryTargets, secondaryTargets };
};

// Applies the mathematical effect of a token to the grid
const applyMove = (gridState, r, c, token, isReverse) => {
  const { primaryTargets, secondaryTargets } = getAffectedCells(gridState, r, c);
  
  // Apply Primary Effect
  primaryTargets.forEach(pt => {
    const cell = gridState[pt.r][pt.c];
    if (token.id === 'multiply') {
      if (isReverse) {
        // Reverse of a multiplier to 0 is any non-zero value
        let randVal = Math.floor(Math.random() * 4) + 2; 
        cell.value = (Math.random() > 0.5 ? 1 : -1) * randVal;
      } else {
        cell.value = 0;
      }
    } else {
      let val = token.primary * (isReverse ? -1 : 1);
      cell.value += val;
    }
  });
  
  // Apply Secondary (Ripple) Effect
  secondaryTargets.forEach(st => {
    const cell = gridState[st.r][st.c];
    if (token.id === 'multiply') {
      // Inverting the inversion is the same operation, so reverse is identical
      cell.value *= -1;
    } else {
      let val = token.secondary * (isReverse ? -1 : 1);
      if (cell.type === 'inverter') val *= -1;
      cell.value += val;
    }
  });
};

// Core action: apply token to a cell and its neighbors
const applyToken = (rowIndex, colIndex) => {
  if (isWin.value) return; // Prevent moves after winning
  if (grid.value[rowIndex][colIndex].type === 'void') return; // Ignore void cells

  const token = activeToken.value;
  if (!token || token.amount <= 0) return;

  // Save current state to history BEFORE modifying
  moveHistory.value.push({
    grid: grid.value.map(row => row.map(cell => ({ ...cell }))),
    inventory: inventory.value.map(t => ({ ...t })),
    activeTokenId: activeTokenId.value
  });

  const { primaryTargets, secondaryTargets } = getAffectedCells(grid.value, rowIndex, colIndex);
  
  applyMove(grid.value, rowIndex, colIndex, token, false);
  
  // Visual effects
  primaryTargets.forEach(pt => triggerAnimation(pt.r, pt.c, 0));
  secondaryTargets.forEach(st => triggerAnimation(st.r, st.c, 150));

  // Deduct the used token from inventory
  token.amount--;
  
  // Optionally auto-deselect if the token runs out
  if (token.amount === 0) {
    activeTokenId.value = null;
  }
};

// Undo the last move
const undoMove = () => {
  if (moveHistory.value.length === 0 || isWin.value) return;
  
  const lastState = moveHistory.value.pop();
  grid.value = lastState.grid;
  inventory.value = lastState.inventory;
  activeTokenId.value = lastState.activeTokenId;
  animatingCells.value = {}; // Clear any running animations
};

// Procedural Scrambler to generate a level
const generateLevel = () => {
  let isLevelValid = false;
  let newGrid = [];
  let plusUsed = 0;
  let minusUsed = 0;
  let multiplyUsed = 0;
  
  // Epoch Logic & Local Levels
  let epoch = 1;
  let localLevel = currentLevel.value;
  
  if (currentLevel.value >= 75) {
    epoch = 5;
    localLevel = currentLevel.value - 74;
  } else if (currentLevel.value >= 50) {
    epoch = 4;
    localLevel = currentLevel.value - 49;
  } else if (currentLevel.value >= 30) {
    epoch = 3;
    localLevel = currentLevel.value - 29;
  } else if (currentLevel.value >= 15) {
    epoch = 2;
    localLevel = currentLevel.value - 14;
  }
  
  // Calculate dynamic scaling parameters based on localLevel
  // Size starts at 3 and increases by 1 every 4 localLevels, capping at 7
  const size = Math.min(7, 3 + Math.floor((localLevel - 1) / 4)); 
  
  // Mechanic Pool logic based on Epoch and Tutorial Phase
  let pool = [];
  let hasMultiplier = false;
  
  if (epoch === 2) {
    pool = ['void'];
  } else if (epoch === 3) {
    if (localLevel <= 5) {
      pool = ['inverter'];
    } else {
      const choices = [['inverter'], ['void'], ['inverter', 'void']];
      pool = choices[Math.floor(Math.random() * choices.length)];
    }
  } else if (epoch === 4) {
    if (localLevel <= 5) {
      pool = ['quantum'];
    } else {
      const choices = [['quantum', 'inverter'], ['quantum', 'void'], ['quantum', 'inverter', 'void']];
      pool = choices[Math.floor(Math.random() * choices.length)];
    }
  } else if (epoch === 5) {
    hasMultiplier = true;
    if (localLevel <= 5) {
      pool = []; // Normal cells to practice 'x'
    } else {
      const choices = [
        ['quantum', 'inverter'], 
        ['quantum', 'void'], 
        ['quantum', 'inverter', 'void'],
        ['inverter', 'void'],
        ['quantum']
      ];
      pool = choices[Math.floor(Math.random() * choices.length)];
    }
  }

  const hasVoids = pool.includes('void');
  const hasInverters = pool.includes('inverter');
  const hasQuantum = pool.includes('quantum');

  let voidCount = 0;
  if (hasVoids) {
    // Max 15% of the board to keep it reasonable
    const maxObs = Math.floor(size * size * 0.15);
    voidCount = Math.max(1, Math.min(maxObs, Math.floor(Math.random() * 4) + 1));
  }
  
  let inverterCount = 0;
  if (hasInverters) inverterCount = Math.max(1, Math.floor(Math.random() * 3) + 1); // 1 to 3
  
  let quantumCount = 0;
  if (hasQuantum) quantumCount = 1; // Exactly one pair
  
  // Ensure we don't accidentally generate an already solved board
  while (!isLevelValid) {
    newGrid = Array.from({ length: size }, (_, r) => 
      Array.from({ length: size }, (_, c) => ({
        id: `${r}-${c}`,
        value: 0,
        type: 'normal'
      }))
    );
    
    // Distribute special cells
    const specialCells = [];
    for (let i = 0; i < voidCount; i++) specialCells.push('void');
    for (let i = 0; i < inverterCount; i++) specialCells.push('inverter');
    if (quantumCount > 0) {
      specialCells.push('quantum_a');
      specialCells.push('quantum_b');
    }
    
    let playableCellsCount = size * size;
    
    specialCells.forEach(type => {
      let r, c;
      do {
        r = Math.floor(Math.random() * size);
        c = Math.floor(Math.random() * size);
      } while (newGrid[r][c].type !== 'normal');
      
      newGrid[r][c].type = type;
      if (type === 'void') playableCellsCount--;
    });
    
    // Scramble Depth scaling based on localLevel
    const baseScramble = Math.min(35, 2 + Math.floor(localLevel * 1.5));
    const scrambleDepth = Math.min(playableCellsCount, baseScramble);
    
    plusUsed = 0;
    minusUsed = 0;
    multiplyUsed = 0;

    for (let i = 0; i < scrambleDepth; i++) {
      let r, c;
      do {
        r = Math.floor(Math.random() * size);
        c = Math.floor(Math.random() * size);
      } while (newGrid[r][c].type === 'void');
      
      let tokenToUse;
      // Guarantee the multiplier is used by forcing it on the very first move if hasMultiplier is true
      let attemptMultiply = hasMultiplier && (Math.random() < 0.25 || (multiplyUsed === 0 && i === 0));
      
      if (attemptMultiply) {
        // Guarantee solvability: Reverse multiplication is only valid if target cell is exactly 0
        const { primaryTargets } = getAffectedCells(newGrid, r, c);
        const canMultiply = primaryTargets.every(pt => newGrid[pt.r][pt.c].value === 0);
        
        if (canMultiply) {
           tokenToUse = inventory.value.find(t => t.id === 'multiply');
           multiplyUsed++;
        } else {
           // Fallback to safe math
           if (Math.random() > 0.5) {
             tokenToUse = inventory.value.find(t => t.id === 'plus');
             plusUsed++;
           } else {
             tokenToUse = inventory.value.find(t => t.id === 'minus');
             minusUsed++;
           }
        }
      } else if (Math.random() > 0.5) {
        tokenToUse = inventory.value.find(t => t.id === 'plus');
        plusUsed++;
      } else {
        tokenToUse = inventory.value.find(t => t.id === 'minus');
        minusUsed++;
      }

      applyMove(newGrid, r, c, tokenToUse, true); // true for isReverse
    }
    
    // Check if the board is actually scrambled
    isLevelValid = !newGrid.every(row => row.every(cell => cell.type === 'void' || cell.value === 0));
    
    // Guarantee that Epoch 5 actually uses a multiplier at least once to ensure puzzle integrity
    if (isLevelValid && hasMultiplier && multiplyUsed === 0) {
      isLevelValid = false;
    }
  }

  // Update game state
  grid.value = newGrid;
  
  // Save initial state for the 'Restart Level' feature
  initialGridState.value = newGrid.map(row => row.map(cell => ({ ...cell })));
  
  initialPlusAmount.value = plusUsed + 1; // Give 1 extra for error room
  initialMinusAmount.value = minusUsed + 1; // Give 1 extra for error room
  initialMultiplyAmount.value = multiplyUsed + (hasMultiplier ? 1 : 0);
  
  inventory.value.find(t => t.id === 'plus').amount = initialPlusAmount.value;
  inventory.value.find(t => t.id === 'minus').amount = initialMinusAmount.value;
  
  const multiplyToken = inventory.value.find(t => t.id === 'multiply');
  if (multiplyToken) {
    multiplyToken.amount = initialMultiplyAmount.value;
  }
  
  activeTokenId.value = null;
  moveHistory.value = [];
  animatingCells.value = {};
  earnedCoins.value = 0;
};

// Initialize the first level
generateLevel();

const nextLevel = () => {
  currentLevel.value++;
  generateLevel();
};

const resetLevel = () => {
  grid.value = initialGridState.value.map(row => row.map(cell => ({ ...cell })));
  inventory.value.find(t => t.id === 'plus').amount = initialPlusAmount.value;
  inventory.value.find(t => t.id === 'minus').amount = initialMinusAmount.value;
  
  const multiplyToken = inventory.value.find(t => t.id === 'multiply');
  if (multiplyToken) {
    multiplyToken.amount = initialMultiplyAmount.value;
  }

  activeTokenId.value = null;
  moveHistory.value = [];
  animatingCells.value = {};
};

// Global Developer Cheat
if (typeof window !== 'undefined') {
  window.dev_setLevel = (level) => {
    if (typeof level === 'number' && level > 0) {
      currentLevel.value = level;
      generateLevel();
      console.log(`Successfully skipped to Level ${level}!`);
    } else {
      console.error('Please provide a valid level number greater than 0.');
    }
  };
}
</script>

<template>
  <div class="game-container">
    <header>
      <div class="header-content">
        <h1>Equilibrium</h1>
        <div class="coin-bank">
          <span class="coin-icon">🪙</span>
          {{ coinBank }}
        </div>
      </div>
      <p>Balance the grid. Make every cell equal to 0.</p>
      <div class="level-indicator">Level {{ currentLevel }}</div>
    </header>

    <!-- Win State Overlay -->
    <div v-if="isWin" class="win-message">
      <h2>Level Complete!</h2>
      <p>Perfect harmony achieved.</p>
      <div class="reward">
        <span class="reward-text">+{{ earnedCoins }} Coins!</span>
      </div>
      <button @click="nextLevel">Next Level</button>
    </div>

    <div class="panels-container">
      <!-- Inventory UI -->
      <div class="inventory">
        <h2>Tokens</h2>
        <div class="tokens">
          <button
            v-for="token in visibleInventory"
            :key="token.id"
            class="token-btn"
            :class="{ active: activeTokenId === token.id, disabled: token.amount === 0 }"
            @click="selectToken(token.id)"
            :disabled="token.amount === 0"
          >
            <span class="token-name">{{ token.name }}</span>
            <span class="token-amount">x{{ token.amount }}</span>
          </button>
        </div>
      </div>

      <!-- Shop UI -->
      <div class="shop">
        <h2>Shop</h2>
        <div class="shop-items">
          <button
            v-for="token in visibleInventory"
            :key="'shop-' + token.id"
            class="shop-btn"
            @click="buyToken(token.id)"
            :disabled="coinBank < token.cost"
          >
            <span class="shop-title">Buy {{ token.name.split(' ')[0] }}</span>
            <span class="shop-cost">{{ token.cost }} 🪙</span>
          </button>
        </div>
      </div>
    </div>

    <!-- The Grid -->
    <div class="grid-wrapper">
      <div class="grid" :class="{ 'game-won': isWin }" :style="{ gridTemplateColumns: `repeat(${grid.length}, 1fr)` }">
        <template v-for="(row, rIndex) in grid" :key="rIndex">
          <div
            v-for="(cell, cIndex) in row"
            :key="cell.id"
            class="cell"
            :class="{
              'void': cell.type === 'void',
              'inverter': cell.type === 'inverter',
              'quantum': cell.type === 'quantum_a' || cell.type === 'quantum_b',
              'zero': cell.value === 0 && cell.type !== 'void',
              'positive': cell.value > 0 && cell.type !== 'void',
              'negative': cell.value < 0 && cell.type !== 'void',
              'animating': animatingCells[cell.id]
            }"
            @click="cell.type !== 'void' ? applyToken(rIndex, cIndex) : null"
          >
            {{ cell.type !== 'void' ? cell.value : '' }}
            <div v-if="cell.type === 'inverter'" class="inverter-icon">🔃</div>
            <div v-if="cell.type === 'quantum_a' || cell.type === 'quantum_b'" class="quantum-icon">⚛️</div>
          </div>
        </template>
      </div>
    </div>

    <!-- Control Actions -->
    <div class="controls">
      <button class="control-btn" @click="undoMove" :disabled="moveHistory.length === 0 || isWin">
        Undo Last Move
      </button>
      <button class="control-btn restart" @click="resetLevel">
        Restart Level
      </button>
    </div>
  </div>
</template>

<style scoped>
.game-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: 'Inter', system-ui, -apple-system, sans-serif;
  background-color: #121212;
  color: #e0e0e0;
  min-height: 100vh;
  padding: 2rem;
  box-sizing: border-box;
}

header {
  text-align: center;
  margin-bottom: 2rem;
}

.header-content {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 1.5rem;
  margin-bottom: 0.5rem;
}

h1 {
  font-size: 3rem;
  margin: 0;
  background: linear-gradient(45deg, #00d2ff 0%, #3a7bd5 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  text-transform: uppercase;
  letter-spacing: 2px;
}

.coin-bank {
  background: rgba(255, 215, 0, 0.1);
  border: 1px solid rgba(255, 215, 0, 0.5);
  padding: 0.4rem 0.8rem;
  border-radius: 20px;
  font-weight: bold;
  color: #ffd700;
  display: flex;
  align-items: center;
  gap: 0.4rem;
  font-size: 1.1rem;
  box-shadow: 0 2px 8px rgba(255, 215, 0, 0.15);
}

p {
  color: #a0a0a0;
  font-size: 1.1rem;
  margin: 0.5rem 0;
}

.level-indicator {
  margin-top: 0.5rem;
  font-size: 1.2rem;
  color: #00d2ff;
  font-weight: bold;
  text-transform: uppercase;
  letter-spacing: 2px;
}

.panels-container {
  display: flex;
  gap: 1.5rem;
  margin-bottom: 2rem;
  flex-wrap: wrap;
  justify-content: center;
  width: 100%;
  max-width: 650px;
}

.inventory, .shop {
  background: #1e1e1e;
  padding: 1.5rem;
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.3);
  text-align: center;
  flex: 1;
  min-width: 250px;
}

.inventory h2, .shop h2 {
  margin-top: 0;
  margin-bottom: 1rem;
  font-size: 1.2rem;
  color: #888;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.tokens, .shop-items {
  display: flex;
  gap: 1rem;
  justify-content: center;
}

.token-btn, .shop-btn {
  background: #2a2a2a;
  border: 2px solid #333;
  border-radius: 8px;
  padding: 0.8rem 1.2rem;
  color: #fff;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
  transition: all 0.2s ease;
  min-width: 100px;
}

.token-btn:hover:not(.disabled) {
  background: #3a3a3a;
  border-color: #555;
  transform: translateY(-2px);
}

.token-btn.active {
  border-color: #00d2ff;
  background: rgba(0, 210, 255, 0.1);
  box-shadow: 0 0 10px rgba(0, 210, 255, 0.3);
}

.token-btn.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.token-name {
  font-weight: bold;
  font-size: 1.1rem;
}

.token-amount {
  font-size: 0.9rem;
  color: #aaa;
}

.shop-btn {
  border-color: #555;
  padding: 0.8rem 1rem;
}

.shop-btn:hover:not(:disabled) {
  background: #3a3a3a;
  border-color: #ffd700;
  color: #fff;
  transform: translateY(-2px);
  box-shadow: 0 4px 10px rgba(255, 215, 0, 0.1);
}

.shop-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.shop-title {
  font-weight: bold;
  font-size: 1rem;
}

.shop-cost {
  font-size: 0.9rem;
  color: #ffd700;
}

.grid-wrapper {
  perspective: 1000px;
}

.grid {
  display: grid;
  gap: 8px;
  background: #1e1e1e;
  padding: 16px;
  border-radius: 12px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5), inset 0 0 0 1px rgba(255, 255, 255, 0.05);
  transition: filter 0.5s ease, transform 0.5s ease;
}

.grid.game-won {
  filter: drop-shadow(0 0 30px rgba(0, 210, 255, 0.6));
  transform: scale(1.02);
}

.cell {
  width: 60px;
  height: 60px;
  box-sizing: border-box;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.5rem;
  font-weight: bold;
  border-radius: 8px;
  cursor: pointer;
  user-select: none;
  transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);
  background: #2a2a2a;
  color: #fff;
  position: relative;
  overflow: hidden;
}

.cell::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  box-shadow: inset 0 0 0 2px rgba(255, 255, 255, 0.05);
  border-radius: 8px;
  pointer-events: none;
}

.cell:hover:not(.void) {
  transform: scale(1.08);
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.5);
  z-index: 1;
}

.cell:active:not(.void) {
  transform: scale(0.95);
}

.cell.zero {
  background: #1a3a2a;
  color: #4ade80;
}
.cell.zero::after {
  box-shadow: inset 0 0 0 2px rgba(74, 222, 128, 0.2);
}

.cell.positive {
  background: #3a2a1a;
  color: #fb923c;
}
.cell.positive::after {
  box-shadow: inset 0 0 0 2px rgba(251, 146, 60, 0.2);
}

.cell.negative {
  background: #1a2a3a;
  color: #60a5fa;
}
.cell.negative::after {
  box-shadow: inset 0 0 0 2px rgba(96, 165, 250, 0.2);
}

/* New Special Cell Styles */
.cell.void {
  background: transparent;
  box-shadow: none;
  cursor: default;
  pointer-events: none;
}
.cell.void::after {
  display: none;
}

.cell.inverter {
  border: 2px solid #a855f7;
  box-shadow: inset 0 0 15px rgba(168, 85, 247, 0.4);
}

.cell.quantum {
  border: 2px solid #3b82f6;
  animation: quantumPulse 2s infinite alternate;
}

@keyframes quantumPulse {
  0% { box-shadow: 0 0 5px rgba(59, 130, 246, 0.4), inset 0 0 5px rgba(59, 130, 246, 0.4); }
  100% { box-shadow: 0 0 20px rgba(59, 130, 246, 0.8), inset 0 0 15px rgba(59, 130, 246, 0.6); }
}

.inverter-icon, .quantum-icon {
  position: absolute;
  top: 4px;
  right: 4px;
  font-size: 1rem;
  opacity: 0.8;
  line-height: 1;
}

@keyframes ripplePop {
  0% {
    transform: scale(1);
    filter: brightness(1);
  }
  50% {
    transform: scale(1.15);
    filter: brightness(1.5);
  }
  100% {
    transform: scale(1);
    filter: brightness(1);
  }
}

.cell.animating {
  animation: ripplePop 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);
  z-index: 2; /* Bring animating cell slightly forward */
}

.win-message {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(18, 18, 18, 0.95);
  padding: 3rem;
  border-radius: 16px;
  border: 1px solid #00d2ff;
  text-align: center;
  z-index: 10;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.8), 0 0 30px rgba(0, 210, 255, 0.2);
  backdrop-filter: blur(10px);
  animation: popIn 0.5s cubic-bezier(0.175, 0.885, 0.32, 1.275) forwards;
}

@keyframes popIn {
  from {
    opacity: 0;
    transform: translate(-50%, -40%) scale(0.8);
  }
  to {
    opacity: 1;
    transform: translate(-50%, -50%) scale(1);
  }
}

.win-message h2 {
  font-size: 2.5rem;
  color: #00d2ff;
  margin-top: 0;
}

.win-message p {
  color: #fff;
  margin: 0.5rem 0;
}

.reward {
  margin: 1.5rem 0;
  font-size: 1.8rem;
  font-weight: bold;
  color: #ffd700;
  text-shadow: 0 0 15px rgba(255, 215, 0, 0.4);
  animation: rewardPop 0.6s cubic-bezier(0.175, 0.885, 0.32, 1.275) 0.3s both;
}

@keyframes rewardPop {
  0% {
    opacity: 0;
    transform: scale(0.5) translateY(20px);
  }
  100% {
    opacity: 1;
    transform: scale(1) translateY(0);
  }
}

.win-message button {
  margin-top: 0.5rem;
  padding: 0.8rem 2rem;
  font-size: 1.1rem;
  background: linear-gradient(45deg, #00d2ff 0%, #3a7bd5 100%);
  color: #fff;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-weight: bold;
  transition: all 0.2s ease;
  box-shadow: 0 4px 15px rgba(0, 210, 255, 0.3);
}

.win-message button:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0, 210, 255, 0.4);
}

.controls {
  margin-top: 2rem;
  display: flex;
  gap: 1rem;
}

.control-btn {
  background: #2a2a2a;
  border: 1px solid #555;
  color: #e0e0e0;
  padding: 0.6rem 1.2rem;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s ease;
  font-size: 1rem;
  font-weight: 500;
}

.control-btn:hover:not(:disabled) {
  background: #3a3a3a;
  border-color: #777;
  color: #fff;
}

.control-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.control-btn.restart {
  border-color: #f87171;
  color: #fca5a5;
  background: rgba(248, 113, 113, 0.1);
}

.control-btn.restart:hover {
  background: rgba(248, 113, 113, 0.2);
  border-color: #ef4444;
  color: #fecaca;
}
</style>
