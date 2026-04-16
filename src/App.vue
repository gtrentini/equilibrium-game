<script setup>
import { ref, computed, watch } from "vue";
import { Haptics, ImpactStyle } from "@capacitor/haptics";

const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
const playSound = (type) => {
  if (audioCtx.state === "suspended") audioCtx.resume();
  const osc = audioCtx.createOscillator();
  const gainNode = audioCtx.createGain();
  osc.connect(gainNode);
  gainNode.connect(audioCtx.destination);

  if (type === "click") {
    osc.type = "sine";
    osc.frequency.setValueAtTime(600, audioCtx.currentTime);
    osc.frequency.exponentialRampToValueAtTime(300, audioCtx.currentTime + 0.1);
    gainNode.gain.setValueAtTime(0.3, audioCtx.currentTime);
    gainNode.gain.exponentialRampToValueAtTime(
      0.01,
      audioCtx.currentTime + 0.1,
    );
    osc.start();
    osc.stop(audioCtx.currentTime + 0.1);
  } else if (type === "special") {
    osc.type = "square";
    osc.frequency.setValueAtTime(200, audioCtx.currentTime);
    gainNode.gain.setValueAtTime(0.2, audioCtx.currentTime);
    gainNode.gain.exponentialRampToValueAtTime(
      0.01,
      audioCtx.currentTime + 0.15,
    );
    osc.start();
    osc.stop(audioCtx.currentTime + 0.15);
  } else if (type === "purchase") {
    osc.type = "sine";
    osc.frequency.setValueAtTime(880, audioCtx.currentTime);
    osc.frequency.setValueAtTime(1108.73, audioCtx.currentTime + 0.05);
    osc.frequency.setValueAtTime(1318.51, audioCtx.currentTime + 0.1);
    gainNode.gain.setValueAtTime(0.2, audioCtx.currentTime);
    gainNode.gain.exponentialRampToValueAtTime(
      0.01,
      audioCtx.currentTime + 0.3,
    );
    osc.start();
    osc.stop(audioCtx.currentTime + 0.3);
  } else if (type === "win") {
    osc.type = "triangle";
    osc.frequency.setValueAtTime(440, audioCtx.currentTime);
    osc.frequency.setValueAtTime(554.37, audioCtx.currentTime + 0.1);
    osc.frequency.setValueAtTime(659.25, audioCtx.currentTime + 0.2);
    gainNode.gain.setValueAtTime(0.3, audioCtx.currentTime);
    gainNode.gain.linearRampToValueAtTime(0, audioCtx.currentTime + 0.5);
    osc.start();
    osc.stop(audioCtx.currentTime + 0.5);
  }
};

const gameState = ref("menu");
const showShop = ref(false);
const shopFeedback = ref("");

const showDevMenu = ref(false);
let devTapCount = 0;
let devTapTimer = null;
const handleDevTap = () => {
  devTapCount++;
  clearTimeout(devTapTimer);
  devTapTimer = setTimeout(() => {
    devTapCount = 0;
  }, 1500);
  if (devTapCount >= 5) {
    showDevMenu.value = true;
    devTapCount = 0;
  }
};
const devForceWin = () => {
  grid.value.forEach((row) =>
    row.forEach((cell) => {
      if (cell.type !== "void") cell.value = 0;
    }),
  );
  showDevMenu.value = false;
};
const devAddCoins = () => {
  coinBank.value += 1000;
};
const devResetAll = () => {
  playerProfile.value = {
    highestUnlockedLevel: 1,
    levelStats: {},
    daily: { lastWinDate: null, streak: 0 },
  };
  coinBank.value = 0;
  currentLevel.value = 1;
  isDailyMode.value = false;
  generateLevel();
  showDevMenu.value = false;
};

const devTargetLevel = ref(1);
const devGoToLevel = () => { if (devTargetLevel.value > 0) { currentLevel.value = devTargetLevel.value; if (devTargetLevel.value > playerProfile.value.highestUnlockedLevel) { playerProfile.value.highestUnlockedLevel = devTargetLevel.value; } isDailyMode.value = false; generateLevel(); showDevMenu.value = false; } };

// Load initial state from localStorage if available
const savedProfile = localStorage.getItem("equilibrium_profile");
const defaultProfile = { highestUnlockedLevel: 1, levelStats: {}, daily: { lastWinDate: null, streak: 0 }, themes: { unlocked: ['default'], active: 'default' } };
const playerProfile = ref(
  savedProfile ? JSON.parse(savedProfile) : defaultProfile,
);
if (!playerProfile.value.themes) { playerProfile.value.themes = { unlocked: ['default'], active: 'default' }; }
const currentLevel = ref(playerProfile.value.highestUnlockedLevel);

const savedCoins = localStorage.getItem("equilibrium_coins");
const coinBank = ref(savedCoins ? parseInt(savedCoins, 10) : 0);

const reversedLevels = computed(() => {
  const levels = [];
  for (let i = playerProfile.value.highestUnlockedLevel; i >= 1; i--) {
    levels.push(i);
  }
  return levels;
});

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
const movesTaken = ref(0);
const levelPar = ref(0);

const earnedStars = computed(() => {
  if (movesTaken.value <= levelPar.value) return 3;
  if (movesTaken.value <= levelPar.value + 2) return 2;
  return 1;
});

// Watch for changes to save to localStorage
watch(
  playerProfile,
  (newVal) => {
    localStorage.setItem("equilibrium_profile", JSON.stringify(newVal));
  },
  { deep: true },
);

watch(coinBank, (newVal) => {
  localStorage.setItem("equilibrium_coins", newVal.toString());
});

// Reactive inventory of playable tokens
const inventory = ref([
  {
    id: "plus",
    name: "+ Ripple",
    primary: 2,
    secondary: 1,
    amount: 0,
    unlockLevel: 1,
    cost: 50,
  },
  {
    id: "minus",
    name: "- Ripple",
    primary: -2,
    secondary: -1,
    amount: 0,
    unlockLevel: 1,
    cost: 50,
  },
  {
    id: "multiply",
    name: "x Multiplier",
    primary: 0,
    secondary: -1,
    amount: 0,
    unlockLevel: 75,
    cost: 100,
  },
]);

const visibleInventory = computed(() => {
  return inventory.value.filter((t) => currentLevel.value >= t.unlockLevel);
});

// Computed property to find the currently active token object
const activeToken = computed(() => {
  return inventory.value.find((t) => t.id === activeTokenId.value);
});

// Win state condition: all non-void cells equal 0
const isWin = computed(() => {
  if (grid.value.length === 0) return false;
  return grid.value.every((row) =>
    row.every((cell) => cell.type === "void" || cell.value === 0),
  );
});

const isDailyMode = ref(false);

const hasPlayedDailyToday = computed(() => {
  return playerProfile.value.daily?.lastWinDate === getTodayString();
});

const formattedDailyDate = computed(() => {
  return new Date().toLocaleDateString("en-US", {
    month: "short",
    day: "numeric",
    year: "numeric",
  });
});

const getDailySeed = () => {
  const d = new Date();
  return parseInt(`${d.getFullYear()}${d.getMonth()}${d.getDate()}`);
};

const getTodayString = () => new Date().toISOString().split("T")[0];
const getYesterdayString = () => {
  const d = new Date();
  d.setDate(d.getDate() - 1);
  return d.toISOString().split("T")[0];
};

// Seeded Random Number Generator
const mulberry32 = (a) => {
  return function () {
    var t = (a += 0x6d2b79f5);
    t = Math.imul(t ^ (t >>> 15), t | 1);
    t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
};

// Award coins upon winning
watch(isWin, (newVal) => {
  if (newVal) {
    playSound("win");
    let reward = Math.floor(Math.random() * (25 - 10 + 1)) + 10;
    if (isDailyMode.value) {
      reward += 100;
    }
    earnedCoins.value = reward;
    coinBank.value += reward;

    if (isDailyMode.value) {
      if (!playerProfile.value.daily)
        playerProfile.value.daily = { lastWinDate: null, streak: 0 };
      const todayStr = getTodayString();
      if (playerProfile.value.daily.lastWinDate !== todayStr) {
        if (playerProfile.value.daily.lastWinDate === getYesterdayString()) {
          playerProfile.value.daily.streak++;
        } else {
          playerProfile.value.daily.streak = 1;
        }
        playerProfile.value.daily.lastWinDate = todayStr;
      }
    }

    if (!isDailyMode.value) {
      const currentStats = playerProfile.value.levelStats[
        currentLevel.value
      ] || {
        cleared: false,
        stars: 0,
      };
      const currentStars = earnedStars.value;
      playerProfile.value.levelStats[currentLevel.value] = {
        cleared: true,
        stars: Math.max(currentStats.stars || 0, currentStars),
      };

      const next = currentLevel.value + 1;
      if (next > playerProfile.value.highestUnlockedLevel) {
        playerProfile.value.highestUnlockedLevel = next;
      }
    }
  }
});

// Action to select a token from inventory
const selectToken = (id) => {
  activeTokenId.value = id;
};

// Shop Action: Buy a token
const buyToken = (tokenId) => {
  const token = inventory.value.find((t) => t.id === tokenId);
  if (token && coinBank.value >= token.cost) {
    coinBank.value -= token.cost;
    token.amount++;
    playSound("purchase");
    shopFeedback.value = "Purchased " + token.name + "!";
    setTimeout(() => {
      shopFeedback.value = "";
    }, 2000);

    // Track purchased tokens so they are not lost on Restart Level
    if (tokenId === "plus") initialPlusAmount.value++;
    if (tokenId === "minus") initialMinusAmount.value++;
    if (tokenId === "multiply") initialMultiplyAmount.value++;

    // Add to history so it's not lost on undo
    moveHistory.value.forEach((historyState) => {
      const historyToken = historyState.inventory.find((t) => t.id === tokenId);
      if (historyToken) {
        historyToken.amount++;
      }
    });
  }
};

const availableThemes = ref([ { id: 'default', name: 'Dark Void (Default)', cost: 0 }, { id: 'neon', name: 'Neon Cyber', cost: 2000 }, { id: 'retro', name: 'Retro Arcade', cost: 2000 } ]);
const setPlayerTheme = (id) => { if (playerProfile.value.themes.unlocked.includes(id)) { playerProfile.value.themes.active = id; playSound('click'); } else { const theme = availableThemes.value.find(t => t.id === id); if (coinBank.value >= theme.cost) { coinBank.value -= theme.cost; playerProfile.value.themes.unlocked.push(id); playerProfile.value.themes.active = id; playSound('purchase'); shopFeedback.value = "Unlocked " + theme.name + "!"; setTimeout(() => { shopFeedback.value = ""; }, 2000); } } };

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
  if (targetCell.type === "quantum_a") {
    for (let ir = 0; ir < size; ir++) {
      for (let ic = 0; ic < size; ic++) {
        if (gridState[ir][ic].type === "quantum_b")
          primaryTargets.push({ r: ir, c: ic });
      }
    }
  } else if (targetCell.type === "quantum_b") {
    for (let ir = 0; ir < size; ir++) {
      for (let ic = 0; ic < size; ic++) {
        if (gridState[ir][ic].type === "quantum_a")
          primaryTargets.push({ r: ir, c: ic });
      }
    }
  }

  let secondaryTargets = [];
  primaryTargets.forEach((pt) => {
    const adjs = [
      [pt.r - 1, pt.c],
      [pt.r + 1, pt.c],
      [pt.r, pt.c - 1],
      [pt.r, pt.c + 1],
    ];
    adjs.forEach(([ar, ac]) => {
      if (
        ar >= 0 &&
        ar < size &&
        ac >= 0 &&
        ac < size &&
        gridState[ar][ac].type !== "void"
      ) {
        secondaryTargets.push({ r: ar, c: ac });
      }
    });
  });

  return { primaryTargets, secondaryTargets };
};

// Applies the mathematical effect of a token to the grid
const applyMove = (gridState, r, c, token, isReverse) => {
  const { primaryTargets, secondaryTargets } = getAffectedCells(
    gridState,
    r,
    c,
  );

  // Apply Primary Effect
  primaryTargets.forEach((pt) => {
    const cell = gridState[pt.r][pt.c];
    if (token.id === "multiply") {
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
  secondaryTargets.forEach((st) => {
    const cell = gridState[st.r][st.c];
    if (token.id === "multiply") {
      // Inverting the inversion is the same operation, so reverse is identical
      cell.value *= -1;
    } else {
      let val = token.secondary * (isReverse ? -1 : 1);
      if (cell.type === "inverter") val *= -1;
      cell.value += val;
    }
  });
};

// Core action: apply token to a cell and its neighbors
const applyToken = async (rowIndex, colIndex) => {
  await Haptics.impact({ style: ImpactStyle.Light });

  if (isWin.value) return; // Prevent moves after winning
  if (grid.value[rowIndex][colIndex].type === "void") return; // Ignore void cells

  const token = activeToken.value;
  if (!token || token.amount <= 0) return;

  playSound(token.id === "multiply" ? "special" : "click");

  // Save current state to history BEFORE modifying
  moveHistory.value.push({
    grid: grid.value.map((row) => row.map((cell) => ({ ...cell }))),
    inventory: inventory.value.map((t) => ({ ...t })),
    activeTokenId: activeTokenId.value,
  });

  const { primaryTargets, secondaryTargets } = getAffectedCells(
    grid.value,
    rowIndex,
    colIndex,
  );

  applyMove(grid.value, rowIndex, colIndex, token, false);

  // Visual effects
  primaryTargets.forEach((pt) => triggerAnimation(pt.r, pt.c, 0));
  secondaryTargets.forEach((st) => triggerAnimation(st.r, st.c, 150));

  // Deduct the used token from inventory
  token.amount--;
  movesTaken.value++;

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
const generateLevel = (overrideSeed = null) => {
  const randomizer = overrideSeed ? mulberry32(overrideSeed) : Math.random;
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
    pool = ["void"];
  } else if (epoch === 3) {
    if (localLevel <= 5) {
      pool = ["inverter"];
    } else {
      const choices = [["inverter"], ["void"], ["inverter", "void"]];
      pool = choices[Math.floor(randomizer() * choices.length)];
    }
  } else if (epoch === 4) {
    if (localLevel <= 5) {
      pool = ["quantum"];
    } else {
      const choices = [
        ["quantum", "inverter"],
        ["quantum", "void"],
        ["quantum", "inverter", "void"],
      ];
      pool = choices[Math.floor(randomizer() * choices.length)];
    }
  } else if (epoch === 5) {
    hasMultiplier = true;
    if (localLevel <= 5) {
      pool = []; // Normal cells to practice 'x'
    } else {
      const choices = [
        ["quantum", "inverter"],
        ["quantum", "void"],
        ["quantum", "inverter", "void"],
        ["inverter", "void"],
        ["quantum"],
      ];
      pool = choices[Math.floor(randomizer() * choices.length)];
    }
  }

  const hasVoids = pool.includes("void");
  const hasInverters = pool.includes("inverter");
  const hasQuantum = pool.includes("quantum");

  let voidCount = 0;
  if (hasVoids) {
    // Max 15% of the board to keep it reasonable
    const maxObs = Math.floor(size * size * 0.15);
    voidCount = Math.max(1, Math.min(maxObs, Math.floor(randomizer() * 4) + 1));
  }

  let inverterCount = 0;
  if (hasInverters)
    inverterCount = Math.max(1, Math.floor(randomizer() * 3) + 1); // 1 to 3

  let quantumCount = 0;
  if (hasQuantum) quantumCount = 1; // Exactly one pair

  // Ensure we don't accidentally generate an already solved board
  while (!isLevelValid) {
    newGrid = Array.from({ length: size }, (_, r) =>
      Array.from({ length: size }, (_, c) => ({
        id: `${r}-${c}`,
        value: 0,
        type: "normal",
      })),
    );

    // Distribute special cells
    const specialCells = [];
    for (let i = 0; i < voidCount; i++) specialCells.push("void");
    for (let i = 0; i < inverterCount; i++) specialCells.push("inverter");
    if (quantumCount > 0) {
      specialCells.push("quantum_a");
      specialCells.push("quantum_b");
    }

    let playableCellsCount = size * size;

    specialCells.forEach((type) => {
      let r, c;
      do {
        r = Math.floor(randomizer() * size);
        c = Math.floor(randomizer() * size);
      } while (newGrid[r][c].type !== "normal");

      newGrid[r][c].type = type;
      if (type === "void") playableCellsCount--;
    });

    // Scramble Depth scaling based on localLevel
    const baseScramble = Math.min(35, 2 + Math.floor(localLevel * 1.5));
    let scrambleDepth = Math.min(playableCellsCount, baseScramble);

    if (localLevel <= 5) {
      scrambleDepth = 3;
    }

    levelPar.value = scrambleDepth;

    plusUsed = 0;
    minusUsed = 0;
    multiplyUsed = 0;

    for (let i = 0; i < scrambleDepth; i++) {
      let r, c;
      do {
        r = Math.floor(randomizer() * size);
        c = Math.floor(randomizer() * size);
      } while (newGrid[r][c].type === "void");

      let tokenToUse;
      // Guarantee the multiplier is used by forcing it on the very first move if hasMultiplier is true
      let attemptMultiply =
        hasMultiplier &&
        (randomizer() < 0.25 || (multiplyUsed === 0 && i === 0));

      if (attemptMultiply) {
        // Guarantee solvability: Reverse multiplication is only valid if target cell is exactly 0
        const { primaryTargets } = getAffectedCells(newGrid, r, c);
        const canMultiply = primaryTargets.every(
          (pt) => newGrid[pt.r][pt.c].value === 0,
        );

        if (canMultiply) {
          tokenToUse = inventory.value.find((t) => t.id === "multiply");
          multiplyUsed++;
        } else {
          // Fallback to safe math
          if (randomizer() > 0.5) {
            tokenToUse = inventory.value.find((t) => t.id === "plus");
            plusUsed++;
          } else {
            tokenToUse = inventory.value.find((t) => t.id === "minus");
            minusUsed++;
          }
        }
      } else if (randomizer() > 0.5) {
        tokenToUse = inventory.value.find((t) => t.id === "plus");
        plusUsed++;
      } else {
        tokenToUse = inventory.value.find((t) => t.id === "minus");
        minusUsed++;
      }

      applyMove(newGrid, r, c, tokenToUse, true); // true for isReverse
    }

    // Check if the board is actually scrambled
    isLevelValid = !newGrid.every((row) =>
      row.every((cell) => cell.type === "void" || cell.value === 0),
    );

    // Guarantee that Epoch 5 actually uses a multiplier at least once to ensure puzzle integrity
    if (isLevelValid && hasMultiplier && multiplyUsed === 0) {
      isLevelValid = false;
    }
  }

  // Update game state
  grid.value = newGrid;

  // Save initial state for the 'Restart Level' feature
  initialGridState.value = newGrid.map((row) =>
    row.map((cell) => ({ ...cell })),
  );

  initialPlusAmount.value = plusUsed + 1; // Give 1 extra for error room
  initialMinusAmount.value = minusUsed + 1; // Give 1 extra for error room
  initialMultiplyAmount.value = multiplyUsed + (hasMultiplier ? 1 : 0);

  inventory.value.find((t) => t.id === "plus").amount = initialPlusAmount.value;
  inventory.value.find((t) => t.id === "minus").amount =
    initialMinusAmount.value;

  const multiplyToken = inventory.value.find((t) => t.id === "multiply");
  if (multiplyToken) {
    multiplyToken.amount = initialMultiplyAmount.value;
  }

  activeTokenId.value = null;
  moveHistory.value = [];
  animatingCells.value = {};
  earnedCoins.value = 0;
  movesTaken.value = 0;
};

// Initialize the first level
generateLevel();

const selectLevel = (lvl) => {
  currentLevel.value = lvl;
  generateLevel();
  gameState.value = "playing";
};

const nextLevel = () => {
  if (isDailyMode.value) {
    isDailyMode.value = false;
    gameState.value = "menu"; // Send them back to menu after winning the daily
  } else {
    currentLevel.value++;
    generateLevel();
  }
};

const startDailyChallenge = () => {
  isDailyMode.value = true;

  // Force the Daily Challenge to always be a Level 60 difficulty
  // (Epoch 4: 6x6 grid, Quantums, Inverters, high scramble)
  currentLevel.value = 60;

  generateLevel(getDailySeed());
  gameState.value = "playing";
};

const resetLevel = () => {
  grid.value = initialGridState.value.map((row) =>
    row.map((cell) => ({ ...cell })),
  );
  inventory.value.find((t) => t.id === "plus").amount = initialPlusAmount.value;
  inventory.value.find((t) => t.id === "minus").amount =
    initialMinusAmount.value;

  const multiplyToken = inventory.value.find((t) => t.id === "multiply");
  if (multiplyToken) {
    multiplyToken.amount = initialMultiplyAmount.value;
  }

  activeTokenId.value = null;
  moveHistory.value = [];
  animatingCells.value = {};
  earnedCoins.value = 0;
  movesTaken.value = 0;
};

const rerollLevel = () => {
  if (isDailyMode.value) {
    generateLevel(getDailySeed()); // Regenerates the exact same daily board
  } else {
    generateLevel();
  }
};

// Global Developer Cheat
if (typeof window !== "undefined") {
  window.dev_setLevel = (level) => {
    if (typeof level === "number" && level > 0) {
      currentLevel.value = level;
      if (level > playerProfile.value.highestUnlockedLevel) {
        playerProfile.value.highestUnlockedLevel = level;
      }
      generateLevel();
      console.log(`Successfully skipped to Level ${level}!`);
    } else {
      console.error("Please provide a valid level number greater than 0.");
    }
  };

  window.dev_winLevel = () => {
    grid.value.forEach((row) => {
      row.forEach((cell) => {
        if (cell.type !== "void") cell.value = 0;
      });
    });
    console.log("Level auto-completed! The win state should now trigger.");
  };
}
</script>

<template>
  <div
    class="game-container"
    :class="playerProfile.themes?.active || 'default'"
  >
    <Transition name="page" mode="out-in">
      <div v-if="gameState === 'menu'" class="title-screen">
        <h1>Equilibrium</h1>
        <button class="start-btn" @click="gameState = 'levelSelect'">
          Start Game
        </button>
        <div
          style="
            display: flex;
            flex-direction: column;
            align-items: center;
            gap: 0.5rem;
          "
        >
          <button
            class="start-btn daily-btn"
            @click="startDailyChallenge"
            :disabled="hasPlayedDailyToday"
          >
            {{
              hasPlayedDailyToday ? "Daily Complete ✅" : "Daily Challenge 📅"
            }}
          </button>
          <span
            v-if="playerProfile.daily?.streak > 0"
            style="
              font-size: 1.1rem;
              color: #ffd700;
              font-weight: bold;
              text-shadow: 0 2px 4px rgba(0, 0, 0, 0.5);
            "
            >🔥 Streak: {{ playerProfile.daily?.streak }}</span
          >
        </div>
      </div>

      <div v-else-if="gameState === 'levelSelect'" class="level-select-wrapper">
        <header>
          <h1>Select Level</h1>
        </header>
        <div class="level-grid">
          <button
            v-for="lvl in reversedLevels"
            :key="lvl"
            class="level-btn"
            :class="{ cleared: playerProfile.levelStats[lvl]?.cleared }"
            @click="selectLevel(lvl)"
          >
            <span class="lvl-number">Level {{ lvl }}</span>
            <span
              class="lvl-stars"
              v-if="playerProfile.levelStats[lvl]?.cleared"
              >[{{ "⭐".repeat(playerProfile.levelStats[lvl]?.stars || 0)
              }}{{
                "☆".repeat(3 - (playerProfile.levelStats[lvl]?.stars || 0))
              }}]</span
            >
          </button>
        </div>
        <button class="back-btn" @click="gameState = 'menu'">
          Back to Menu
        </button>
      </div>

      <div v-else-if="gameState === 'playing'" class="game-ui-wrapper">
        <div class="app-bar">
          <button
            class="quit-btn"
            @click="
              gameState = 'levelSelect';
              isDailyMode = false;
            "
          >
            ⬅️ Map
          </button>
          <div class="coin-bank" @click="handleDevTap">
            <span class="coin-icon">🪙</span>
            {{ coinBank }}
          </div>
        </div>

        <header>
          <p v-if="currentLevel <= 10">
            Balance the grid. Make every cell equal to 0.
          </p>
          <div class="level-indicator">
            <span v-if="isDailyMode"
              >Daily Challenge: {{ formattedDailyDate }}</span
            >
            <span v-else>Level {{ currentLevel }}</span>
            <span class="par-indicator">
              Moves: {{ movesTaken }} / Par: {{ levelPar }}</span
            >
          </div>
        </header>

        <div v-if="isWin" class="win-message">
          <h2>Level Complete!</h2>
          <div class="stars-display">
            {{ "⭐".repeat(earnedStars) }}{{ "☆".repeat(3 - earnedStars) }}
          </div>
          <p>Perfect harmony achieved.</p>
          <p class="stats-text">
            Moves: {{ movesTaken }} / Par: {{ levelPar }}
          </p>
          <div class="reward">
            <span class="reward-text">+{{ earnedCoins }} Coins!</span>
          </div>
          <button @click="nextLevel">Next Level</button>
        </div>

        <div class="tokens">
          <button
            v-for="token in visibleInventory"
            :key="token.id"
            class="token-btn"
            :class="{
              active: activeTokenId === token.id,
              disabled: token.amount === 0,
            }"
            @click="selectToken(token.id)"
            :disabled="token.amount === 0"
          >
            <span class="token-name">{{ token.name }}</span>
            <span class="token-amount">x{{ token.amount }}</span>
          </button>
        </div>

        <Transition name="modal">
          <div v-if="showShop" class="shop-modal-overlay">
            <div class="shop-modal-content">
              <div class="shop-modal-header">
                <h2>Shop</h2>
                <button class="close-shop-btn" @click="showShop = false">
                  Close
                </button>
              </div>
              <div
                v-if="shopFeedback"
                style="color: #4ade80; margin-bottom: 1rem; font-weight: bold"
              >
                {{ shopFeedback }}
              </div>
              <div class="shop-items">
                <button
                  v-for="token in visibleInventory"
                  :key="'shop-' + token.id"
                  class="shop-btn"
                  @click="buyToken(token.id)"
                  :disabled="coinBank < token.cost"
                >
                  <span class="shop-title"
                    >Buy {{ token.name.split(" ")[0] }}</span
                  >
                  <span class="shop-cost">{{ token.cost }} 🪙</span>
                </button>
              </div>

              <h3
                style="
                  color: #ffd700;
                  margin-top: 2rem;
                  margin-bottom: 1rem;
                  text-transform: uppercase;
                "
              >
                Cosmetic Themes
              </h3>
              <div class="shop-items">
                <button
                  v-for="theme in availableThemes"
                  :key="'theme-' + theme.id"
                  class="shop-btn"
                  @click="setPlayerTheme(theme.id)"
                  :disabled="
                    !playerProfile.themes?.unlocked.includes(theme.id) &&
                    coinBank < theme.cost
                  "
                  :style="
                    playerProfile.themes?.active === theme.id
                      ? 'border-color: #00d2ff; background: rgba(0, 210, 255, 0.1);'
                      : ''
                  "
                >
                  <span class="shop-title">{{ theme.name }}</span>
                  <span
                    class="shop-cost"
                    v-if="!playerProfile.themes?.unlocked.includes(theme.id)"
                    >{{ theme.cost }} 🪙</span
                  >
                  <span
                    class="shop-cost"
                    v-else-if="playerProfile.themes?.active === theme.id"
                    style="color: #00d2ff"
                    >Active</span
                  >
                  <span class="shop-cost" v-else style="color: #a0a0a0"
                    >Equip</span
                  >
                </button>
              </div>
            </div>
          </div>
        </Transition>

        <div class="grid-wrapper">
          <div
            class="grid"
            :class="{ 'game-won': isWin }"
            :style="{ gridTemplateColumns: `repeat(${grid.length}, 1fr)` }"
          >
            <template v-for="(row, rIndex) in grid" :key="rIndex">
              <div
                v-for="(cell, cIndex) in row"
                :key="cell.id"
                class="cell"
                :class="{
                  void: cell.type === 'void',
                  inverter: cell.type === 'inverter',
                  quantum:
                    cell.type === 'quantum_a' || cell.type === 'quantum_b',
                  zero: cell.value === 0 && cell.type !== 'void',
                  positive: cell.value > 0 && cell.type !== 'void',
                  negative: cell.value < 0 && cell.type !== 'void',
                  animating: animatingCells[cell.id],
                }"
                @click="
                  cell.type !== 'void' ? applyToken(rIndex, cIndex) : null
                "
              >
                {{ cell.type !== "void" ? cell.value : "" }}
                <div v-if="cell.type === 'inverter'" class="inverter-icon">
                  🔃
                </div>
                <div
                  v-if="cell.type === 'quantum_a' || cell.type === 'quantum_b'"
                  class="quantum-icon"
                >
                  ⚛️
                </div>
              </div>
            </template>
          </div>
        </div>

        <div class="controls">
          <button
            class="control-btn"
            @click="undoMove"
            :disabled="moveHistory.length === 0 || isWin"
          >
            ↩️ Undo
          </button>
          <button
            class="control-btn restart"
            @click="resetLevel"
            :disabled="isWin"
          >
            🔄 Reset
          </button>
          <button
            v-if="!isDailyMode"
            class="control-btn reroll"
            @click="rerollLevel"
            :disabled="isWin"
          >
            🎲 Scrap
          </button>
          <button
            class="control-btn shop-toggle"
            @click="showShop = true"
            :disabled="isWin"
          >
            🛒 Shop
          </button>
        </div>
      </div>
    </Transition>

    <Transition name="modal">
      <div v-if="showDevMenu" class="shop-modal-overlay" style="z-index: 999">
        <div class="shop-modal-content" style="border-color: #ff0055">
          <h2 style="color: #ff0055">🛠️ Dev Menu</h2>
          <div
            style="
              display: flex;
              flex-direction: column;
              gap: 1rem;
              margin: 2rem 0;
            "
          >
            <div style="display: flex; gap: 0.5rem;">
<input type="number" v-model="devTargetLevel" style="width: 80px; text-align: center; background: #333; color: white; border: 1px solid #555; border-radius: 6px; font-size: 1.2rem;" min="1" />
<button class="shop-btn" style="flex: 1; min-height: 44px; padding: 0.5rem;" @click="devGoToLevel">Jump to Level</button>
</div>
            <button class="shop-btn" @click="devForceWin">
              Force Win (Current Level)
            </button>
            <button class="shop-btn" @click="devAddCoins">+1000 Coins</button>
            <button
              class="shop-btn"
              style="border-color: #ff0055; color: #ffcaea"
              @click="devResetAll"
            >
              ⚠️ HARD RESET ALL DATA
            </button>
          </div>
          <button class="close-shop-btn" @click="showDevMenu = false">
            Close
          </button>
        </div>
      </div>
    </Transition>
  </div>
</template>

<style scoped>
.game-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family:
    "Inter",
    system-ui,
    -apple-system,
    sans-serif;
  background-color: #121212;
  color: #e0e0e0;
  min-height: 100vh;
  padding: clamp(0.5rem, 2vw, 1rem);
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
  font-size: clamp(1.8rem, 6vw, 3rem);
  line-height: 1.2;
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
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.2rem;
}

.par-indicator {
  color: #a0a0a0;
  font-size: 1rem;
  font-weight: normal;
  letter-spacing: normal;
  text-transform: none;
}

.app-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
  padding: clamp(0.25rem, 1.5vw, 0.5rem) clamp(0.5rem, 2vw, 1rem);
  margin-bottom: 1rem;
}

.quit-btn {
  padding: clamp(0.4rem, 1.5vw, 0.8rem) clamp(1rem, 3vw, 2rem);
  font-size: 1.2rem;
  background: #333;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-weight: bold;
  transition:
    transform 0.2s,
    box-shadow 0.2s;
  min-height: 50px;
}

.quit-btn:hover {
  transform: translateY(-2px);
  background: #555;
}

.open-shop-btn {
  padding: 0.8rem 2rem;
  font-size: 1.2rem;
  background: linear-gradient(45deg, #10b981 0%, #059669 100%);
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-weight: bold;
  box-shadow: 0 4px 15px rgba(16, 185, 129, 0.3);
  transition:
    transform 0.2s,
    box-shadow 0.2s;
  min-height: 50px;
}

.open-shop-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(16, 185, 129, 0.4);
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

.inventory {
  text-align: center;
  flex: 1;
  min-width: 250px;
}

.tokens {
  display: flex;
  gap: 1rem;
  justify-content: center;
}

.token-btn,
.shop-btn {
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
  min-height: 50px;
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

.shop-modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.85);
  backdrop-filter: blur(5px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
}

.shop-modal-content {
  background: #1e1e1e;
  padding: 2rem;
  border-radius: 16px;
  border: 1px solid #ffd700;
  box-shadow:
    0 10px 40px rgba(0, 0, 0, 0.8),
    0 0 20px rgba(255, 215, 0, 0.2);
  width: 90%;
  max-width: 500px;
  text-align: center;
}

.shop-modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
}

.shop-modal-header h2 {
  margin: 0;
  color: #ffd700;
  font-size: 1.8rem;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.close-shop-btn {
  background: #333;
  color: #fff;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 6px;
  cursor: pointer;
  font-weight: bold;
  transition: background 0.2s;
  min-height: 50px;
}

.close-shop-btn:hover {
  background: #555;
}

.shop-items {
  display: flex;
  gap: 1rem;
  justify-content: center;
  flex-wrap: wrap;
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
  width: 100%;
  max-width: 450px;
  margin: 0 auto;
}

.grid {
  display: grid;
  gap: clamp(4px, 1vw, 8px);
  background: #1e1e1e;
  background-image: radial-gradient(
    rgba(255, 255, 255, 0.05) 2px,
    transparent 2px
  );
  background-size: 14px 14px;
  padding: clamp(8px, 2vw, 16px);
  border-radius: 12px;
  box-shadow:
    0 10px 30px rgba(0, 0, 0, 0.5),
    inset 0 0 0 1px rgba(255, 255, 255, 0.05);
  transition:
    filter 0.5s ease,
    transform 0.5s ease;
}

.grid.game-won {
  filter: drop-shadow(0 0 30px rgba(0, 210, 255, 0.6));
  transform: scale(1.02);
}

.cell {
  aspect-ratio: 1 / 1;
  width: 100%;
  height: auto;
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
  content: "";
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
  0% {
    box-shadow:
      0 0 5px rgba(59, 130, 246, 0.4),
      inset 0 0 5px rgba(59, 130, 246, 0.4);
  }
  100% {
    box-shadow:
      0 0 20px rgba(59, 130, 246, 0.8),
      inset 0 0 15px rgba(59, 130, 246, 0.6);
  }
}

.inverter-icon,
.quantum-icon {
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
  box-shadow:
    0 20px 50px rgba(0, 0, 0, 0.8),
    0 0 30px rgba(0, 210, 255, 0.2);
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

.stars-display {
  font-size: 3rem;
  margin: 1rem 0;
  letter-spacing: 0.5rem;
}

.stats-text {
  font-size: 1.1rem;
  color: #ccc;
  margin: 0.5rem 0;
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
  margin-top: 1rem; /* Reduced from 2rem */
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: clamp(0.5rem, 2vw, 1rem); /* Slightly larger, fluid gap */
  width: 100%;
  padding: 0;
}

.control-btn {
  flex: 1 1 calc(25% - 1rem); /* This makes them stretch evenly! */
  background: #2a2a2a;
  border: 1px solid #555;
  color: #e0e0e0;
  padding: 0.5rem clamp(0.5rem, 2vw, 1rem);
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s ease;
  font-size: clamp(0.85rem, 2.5vw, 1rem);
  font-weight: 500;
  min-height: 44px;
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

/* Title Screen Styles */
.title-screen {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  flex: 1;
  width: 100%;
  gap: 2rem;
  margin-top: 10vh;
}

.title-screen h1 {
  font-size: clamp(2.5rem, 10vw, 4.5rem);
  line-height: 1.2;
  margin-bottom: 1rem;
}

.start-btn {
  padding: 1.2rem 3.5rem;
  font-size: 1.5rem;
  background: linear-gradient(45deg, #00d2ff 0%, #3a7bd5 100%);
  color: #fff;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  font-weight: bold;
  transition: all 0.2s ease;
  box-shadow: 0 4px 15px rgba(0, 210, 255, 0.3);
  text-transform: uppercase;
  letter-spacing: 2px;
  min-height: 50px;
}

.start-btn:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 25px rgba(0, 210, 255, 0.5);
}

.start-btn:disabled {
  opacity: 0.5;
  filter: grayscale(100%);
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

.game-ui-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
}

/* Level Select Styles */
.level-select-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  gap: 2rem;
  margin-top: 5vh;
}

.level-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
  justify-content: center;
  max-width: 450px;
}

.level-btn {
  min-width: 100px;
  height: auto;
  min-height: 60px;
  padding: 0.5rem 1rem;
  border-radius: 12px;
  background: #2a2a2a;
  color: #fff;
  border: 2px solid #555;
  font-size: 1.2rem;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.2rem;
}

.level-btn.cleared {
  border-color: #00d2ff;
  background: rgba(0, 210, 255, 0.1);
}

.level-btn:hover {
  transform: translateY(-2px);
  border-color: #00d2ff;
}

.lvl-stars {
  font-size: 0.9rem;
  white-space: pre;
}

.back-btn {
  padding: clamp(0.4rem, 1.5vw, 0.8rem) clamp(1rem, 3vw, 2rem);
  font-size: 1.2rem;
  background: #333;
  color: #fff;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-weight: bold;
  transition: all 0.2s ease;
  min-height: 50px;
}

.back-btn:hover {
  background: #555;
}

/* --- VUE TRANSITION PHYSICS --- */

/* Page Transitions */
.page-enter-active,
.page-leave-active {
  transition: all 0.25s ease-out;
}
.page-enter-from,
.page-leave-to {
  opacity: 0;
  transform: translateY(10px);
}

/* Modal Overlay Fade (The dark background) */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.25s ease;
}
.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

/* Modal Content Scale (The springy box) */
.modal-enter-active .shop-modal-content,
.modal-leave-active .shop-modal-content {
  /* Using a custom bezier curve for a slight "bounce" effect */
  transition: transform 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}
.modal-enter-from .shop-modal-content,
.modal-leave-to .shop-modal-content {
  transform: scale(0.8);
}

/* App Bar & Token Layout Fixes */
.app-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
  padding: clamp(0.25rem, 1.5vw, 0.5rem) clamp(0.5rem, 2vw, 1rem);
  margin-bottom: 1rem;
}

.tokens {
  display: flex;
  gap: 1rem;
  justify-content: center;
  margin-bottom: 1.5rem;
}

.game-container.neon { background-color: #050510; color: #00ffff; }
.game-container.neon h1 { background: linear-gradient(45deg, #ff00ff 0%, #00ffff 100%); -webkit-background-clip: text; background-clip: text; }
.game-container.neon .grid { background: #0a0a1a; box-shadow: 0 0 20px rgba(0, 255, 255, 0.2), inset 0 0 0 2px #ff00ff; background-image: linear-gradient(rgba(0, 255, 255, 0.1) 1px, transparent 1px), linear-gradient(90deg, rgba(0, 255, 255, 0.1) 1px, transparent 1px); background-size: 20px 20px; }
.game-container.neon .cell { background: rgba(0, 0, 0, 0.5); border: 1px solid #00ffff; box-shadow: inset 0 0 10px rgba(0, 255, 255, 0.2); }
.game-container.neon .cell.zero { background: rgba(255, 0, 255, 0.2); color: #ff00ff; border-color: #ff00ff; box-shadow: 0 0 15px rgba(255, 0, 255, 0.5); }
.game-container.neon .cell::after { display: none; }
.game-container.neon .level-btn.cleared { border-color: #ff00ff; background: rgba(255, 0, 255, 0.1); }

.game-container.retro { background-color: #2b2b2b; font-family: 'Courier New', Courier, monospace; }
.game-container.retro h1 { background: none; -webkit-text-fill-color: #ffd700; text-shadow: 4px 4px 0px #ff0055; letter-spacing: 0; }
.game-container.retro .grid { background: #111; border: 4px solid #ffd700; border-radius: 0; box-shadow: 8px 8px 0px #ff0055; background-image: none; }
.game-container.retro .cell { border-radius: 0; background: #222; border: 2px solid #555; box-shadow: inset -2px -2px 0px rgba(0,0,0,0.5); font-weight: bold; }
.game-container.retro .cell.zero { background: #ffd700; color: #111; border-color: #fff; box-shadow: none; }
.game-container.retro .cell::after { display: none; }
.game-container.retro .control-btn { border-radius: 0; border: 2px solid #fff; box-shadow: 2px 2px 0px #888; }
.game-container.retro .start-btn { border-radius: 0; background: #ff0055; box-shadow: 4px 4px 0px #ffd700; }
</style>
