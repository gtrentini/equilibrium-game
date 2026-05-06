<script setup>
import { ref, watch } from "vue";

const props = defineProps({
  showTutorial: Boolean,
});
const emit = defineEmits(["close"]);

const currentStep = ref(1);
const activeTutorialToken = ref(null);
const tutorialComplete = ref(false);

const step2Grid = ref(Array(9).fill(0));
const handleStep2Click = (index) => {
  if (index === 4) {
    step2Grid.value[4] += 2;
    step2Grid.value[1] += 1;
    step2Grid.value[3] += 1;
    step2Grid.value[5] += 1;
    step2Grid.value[7] += 1;
  }
};

const step3Grid = ref(Array(9).fill(0));
const handleStep3Click = (index) => {
  if (index === 4) {
    step3Grid.value[4] += 2;
    step3Grid.value[1] += 1;
    step3Grid.value[3] += 1;
    step3Grid.value[7] += 1;
    step3Grid.value[5] -= 1;
  }
};

const step4Grid = ref(Array(9).fill(0));
const handleStep4Click = (index) => {
  if (index === 0 || index === 8) {
    step4Grid.value[0] += 1;
    step4Grid.value[8] += 1;
  }
};

const step5Grid = ref(Array(9).fill(0));
const handleStep5Click = (index) => {
  if (index === 4 && activeTutorialToken.value === "minus") {
    step5Grid.value[4] -= 2;
    tutorialComplete.value = true;
  }
};

watch(
  () => props.showTutorial,
  (newVal) => {
    if (newVal) {
      currentStep.value = 1;
      resetGrids();
    }
  },
);

watch(currentStep, () => {
  resetGrids();
});

const resetGrids = () => {
  step2Grid.value = Array(9).fill(0);
  step3Grid.value = Array(9).fill(0);
  step4Grid.value = Array(9).fill(0);

  step5Grid.value = Array(9).fill(0);
  step5Grid.value[4] = 2; // Center cell is 2 initially

  activeTutorialToken.value = null;
  tutorialComplete.value = false;
};

const nextStep = () => {
  if (currentStep.value < 5) {
    currentStep.value++;
  } else {
    emit("close");
  }
};

const prevStep = () => {
  if (currentStep.value > 1) {
    currentStep.value--;
  }
};

const getCellClass = (val, isSpecial = null) => {
  let cls = "cell";
  if (val === 0) cls += " zero";
  else if (val > 0) cls += " positive";
  else if (val < 0) cls += " negative";

  if (isSpecial === "inverter") cls += " inverter";
  if (isSpecial === "quantum") cls += " quantum";

  return cls;
};
</script>

<template>
  <Transition name="modal">
    <div v-if="showTutorial" class="shop-modal-overlay" style="z-index: 1000">
      <div class="shop-modal-content" style="max-width: 500px; width: 90%">
        <div class="shop-modal-header">
          <h2>📖 How to Play</h2>
          <button class="close-shop-btn" @click="emit('close')">Close</button>
        </div>

        <div
          style="
            color: #e0e0e0;
            text-align: left;
            display: flex;
            flex-direction: column;
            gap: 1.5rem;
            margin-top: 1rem;
            line-height: 1.5;
            min-height: 300px;
          "
        >
          <!-- Step 1 -->
          <div v-if="currentStep === 1">
            <strong style="font-size: 1.2rem; color: #00d2ff"
              >Step 1: The Goal</strong
            >
            <p style="margin-top: 0.5rem; margin-bottom: 1.5rem">
              The objective is to make every cell equal exactly zero.
            </p>

            <div
              class="grid"
              style="
                grid-template-columns: repeat(3, 1fr);
                max-width: 250px;
                margin: 0 auto;
              "
            >
              <div v-for="i in 9" :key="i" class="cell zero">0</div>
            </div>
          </div>

          <!-- Step 2 -->
          <div v-if="currentStep === 2">
            <strong style="font-size: 1.2rem; color: #00d2ff"
              >Step 2: The Ripple</strong
            >
            <p style="margin-top: 0.5rem; margin-bottom: 1.5rem">
              Tap the center cell. It adds +2 to the center, and +1 to the
              adjacent cells.
            </p>

            <div
              class="grid"
              style="
                grid-template-columns: repeat(3, 1fr);
                max-width: 250px;
                margin: 0 auto;
              "
            >
              <div
                v-for="(val, index) in step2Grid"
                :key="index"
                :class="getCellClass(val)"
                :style="{ pointerEvents: index === 4 ? 'auto' : 'none' }"
                @click="handleStep2Click(index)"
              >
                {{ val }}
              </div>
            </div>
          </div>

          <!-- Step 3 -->
          <div v-if="currentStep === 3">
            <strong style="font-size: 1.2rem; color: #00d2ff"
              >Step 3: The Inverter</strong
            >
            <p style="margin-top: 0.5rem; margin-bottom: 1.5rem">
              Tap the center cell. The Inverter subtracts 1 instead of adding 1.
            </p>

            <div
              class="grid"
              style="
                grid-template-columns: repeat(3, 1fr);
                max-width: 250px;
                margin: 0 auto;
              "
            >
              <div
                v-for="(val, index) in step3Grid"
                :key="index"
                :class="getCellClass(val, index === 5 ? 'inverter' : null)"
                :style="{ pointerEvents: index === 4 ? 'auto' : 'none' }"
                @click="handleStep3Click(index)"
              >
                {{ val }}
                <span v-if="index === 5" class="inverter-icon">!</span>
              </div>
            </div>
          </div>

          <!-- Step 4 -->
          <div v-if="currentStep === 4">
            <strong style="font-size: 1.2rem; color: #00d2ff"
              >Step 4: The Quantum</strong
            >
            <p style="margin-top: 0.5rem; margin-bottom: 1.5rem">
              Tap a Quantum cell. It applies the exact same value to both
              Quantum cells simultaneously.
            </p>

            <div
              class="grid"
              style="
                grid-template-columns: repeat(3, 1fr);
                max-width: 250px;
                margin: 0 auto;
              "
            >
              <div
                v-for="(val, index) in step4Grid"
                :key="index"
                :class="
                  getCellClass(
                    val,
                    index === 0 || index === 8 ? 'quantum' : null,
                  )
                "
                :style="{
                  pointerEvents: index === 0 || index === 8 ? 'auto' : 'none',
                }"
                @click="handleStep4Click(index)"
              >
                {{ val }}
                <span v-if="index === 0 || index === 8" class="quantum-icon"
                  >∞</span
                >
              </div>
            </div>
          </div>

          <!-- Step 5 -->
          <div v-if="currentStep === 5">
            <strong style="font-size: 1.2rem; color: #00d2ff"
              >Step 5: Equip & Solve</strong
            >
            <p style="margin-top: 0.5rem; margin-bottom: 1.5rem">
              Equip the token below and tap the center cell to zero it out.
            </p>

            <div
              style="
                margin-bottom: 1.5rem;
                display: flex;
                justify-content: center;
                gap: 1rem;
              "
            >
              <button
                class="token-btn"
                :class="{ active: activeTutorialToken === 'minus' }"
                :style="{
                  pointerEvents:
                    activeTutorialToken === 'minus' ? 'none' : 'auto',
                }"
                @click="activeTutorialToken = 'minus'"
              >
                <span class="token-name">- Ripple</span>
                <span class="token-amount">Tutorial</span>
              </button>
            </div>

            <div
              class="grid"
              style="
                grid-template-columns: repeat(3, 1fr);
                max-width: 250px;
                margin: 0 auto;
              "
            >
              <div
                v-for="(val, index) in step5Grid"
                :key="index"
                :class="getCellClass(val)"
                :style="{
                  pointerEvents:
                    index === 4 &&
                    activeTutorialToken === 'minus' &&
                    !tutorialComplete
                      ? 'auto'
                      : 'none',
                }"
                @click="handleStep5Click(index)"
              >
                {{ val }}
              </div>
            </div>

            <div
              v-if="tutorialComplete"
              style="
                margin-top: 1.5rem;
                text-align: center;
                color: #4ade80;
                font-size: 1.5rem;
                font-weight: bold;
              "
            >
              Tutorial Complete!
            </div>
          </div>
        </div>

        <!-- Footer for Navigation -->
        <div
          style="
            display: flex;
            justify-content: space-between;
            margin-top: 2rem;
            padding-top: 1rem;
            border-top: 1px solid #333;
          "
        >
          <button
            class="control-btn"
            :disabled="currentStep === 1"
            @click="prevStep"
            style="flex: none; min-width: 120px"
          >
            Previous
          </button>

          <button
            class="control-btn"
            @click="nextStep"
            style="
              flex: none;
              min-width: 120px;
              background: #00d2ff;
              color: #000;
              font-weight: bold;
              border-color: #00d2ff;
            "
          >
            {{ currentStep === 5 ? "Close Tutorial" : "Next" }}
          </button>
        </div>
      </div>
    </div>
  </Transition>
</template>
