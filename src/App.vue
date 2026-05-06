<script setup>
import { ref, computed, watch } from "vue";
import { Haptics, ImpactStyle } from "@capacitor/haptics";

import TitleScreen from "./components/TitleScreen.vue";
import LevelSelect from "./components/LevelSelect.vue";
import ShopModal from "./components/ShopModal.vue";
import DevMenu from "./components/DevMenu.vue";
import SettingsModal from "./components/SettingsModal.vue";
import WinMessage from "./components/WinMessage.vue";
import GameGrid from "./components/GameGrid.vue";
import GameControls from "./components/GameControls.vue";
import TokenInventory from "./components/TokenInventory.vue";
import HowToPlayModal from "./components/HowToPlayModal.vue";

const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
const playSound = (type) => {
  if (!gameSettings.value.soundEnabled) return;
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
const showSettings = ref(false);
const showTutorial = ref(false);
const savedSettings = localStorage.getItem("equilibrium_settings");
const defaultSettings = { soundEnabled: true, hapticsEnabled: true };
const gameSettings = ref(
  savedSettings ? JSON.parse(savedSettings) : defaultSettings,
);

watch(
  gameSettings,
  (newVal) => {
    localStorage.setItem("equilibrium_settings", JSON.stringify(newVal));
  },
  { deep: true },
);

const wipeData = () => {
  if (
    confirm(
      "Are you sure you want to wipe all progress? This cannot be undone.",
    )
  ) {
    localStorage.clear();
    window.location.reload();
  }
};

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
const devGoToLevel = () => {
  if (devTargetLevel.value > 0) {
    currentLevel.value = devTargetLevel.value;
    if (devTargetLevel.value > playerProfile.value.highestUnlockedLevel) {
      playerProfile.value.highestUnlockedLevel = devTargetLevel.value;
    }
    isDailyMode.value = false;
    generateLevel();
    showDevMenu.value = false;
  }
};

// Load initial state from localStorage if available
const savedProfile = localStorage.getItem("equilibrium_profile");
const defaultProfile = {
  highestUnlockedLevel: 1,
  levelStats: {},
  daily: { lastWinDate: null, streak: 0 },
  themes: { unlocked: ["default"], active: "default" },
};
const playerProfile = ref(
  savedProfile ? JSON.parse(savedProfile) : defaultProfile,
);
if (!playerProfile.value.themes) {
  playerProfile.value.themes = { unlocked: ["default"], active: "default" };
}
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

const availableThemes = ref([
  { id: "daylight", name: "Daylight Mode", cost: 0 },
  { id: "default", name: "Dark Void (Default)", cost: 0 },
  { id: "neon", name: "Neon Cyber", cost: 2000 },
  { id: "retro", name: "Retro Arcade", cost: 2000 },
]);
const setPlayerTheme = (id) => {
  if (playerProfile.value.themes.unlocked.includes(id)) {
    playerProfile.value.themes.active = id;
    playSound("click");
  } else {
    const theme = availableThemes.value.find((t) => t.id === id);
    if (coinBank.value >= theme.cost) {
      coinBank.value -= theme.cost;
      playerProfile.value.themes.unlocked.push(id);
      playerProfile.value.themes.active = id;
      playSound("purchase");
      shopFeedback.value = "Unlocked " + theme.name + "!";
      setTimeout(() => {
        shopFeedback.value = "";
      }, 2000);
    }
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
  if (gameSettings.value.hapticsEnabled) {
    await Haptics.impact({ style: ImpactStyle.Light });
  }

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
      <TitleScreen
        v-if="gameState === 'menu'"
        :hasPlayedDailyToday="hasPlayedDailyToday"
        :dailyStreak="playerProfile.daily?.streak || 0"
        @start-game="gameState = 'levelSelect'"
        @start-daily="startDailyChallenge"
        @open-settings="showSettings = true"
        @open-tutorial="showTutorial = true"
      />

      <LevelSelect
        v-else-if="gameState === 'levelSelect'"
        :reversedLevels="reversedLevels"
        :playerProfile="playerProfile"
        @select-level="selectLevel"
        @back-to-menu="gameState = 'menu'"
      />

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

        <WinMessage
          :isWin="isWin"
          :earnedStars="earnedStars"
          :movesTaken="movesTaken"
          :levelPar="levelPar"
          :earnedCoins="earnedCoins"
          @next-level="nextLevel"
        />

        <TokenInventory
          :visibleInventory="visibleInventory"
          :activeTokenId="activeTokenId"
          @select-token="selectToken"
        />

        <ShopModal
          :showShop="showShop"
          :shopFeedback="shopFeedback"
          :visibleInventory="visibleInventory"
          :availableThemes="availableThemes"
          :coinBank="coinBank"
          :playerProfile="playerProfile"
          @close="showShop = false"
          @buy-token="buyToken"
          @set-theme="setPlayerTheme"
        />

        <GameGrid
          :grid="grid"
          :isWin="isWin"
          :animatingCells="animatingCells"
          @apply-token="applyToken"
        />

        <GameControls
          :isWin="isWin"
          :isDailyMode="isDailyMode"
          :canUndo="moveHistory.length > 0"
          @undo="undoMove"
          @reset="resetLevel"
          @reroll="rerollLevel"
          @open-shop="showShop = true"
        />
      </div>
    </Transition>

    <DevMenu
      :showDevMenu="showDevMenu"
      :initialTargetLevel="devTargetLevel"
      @close="showDevMenu = false"
      @go-to-level="devGoToLevel"
      @force-win="devForceWin"
      @add-coins="devAddCoins"
      @reset-all="devResetAll"
      @update-target-level="(val) => (devTargetLevel = val)"
    />

    <SettingsModal
      :showSettings="showSettings"
      :settings="gameSettings"
      @close="showSettings = false"
      @update-settings="(val) => (gameSettings = val)"
      @wipe-data="wipeData"
    />

    <HowToPlayModal
      :showTutorial="showTutorial"
      @close="showTutorial = false"
    />
  </div>
</template>
