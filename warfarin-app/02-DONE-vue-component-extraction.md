# Task: Vue Component Extraction from `App.vue`

## Objective
Refactor the monolithic `App.vue` into smaller, reusable, and Single Responsibility components. This improves maintainability, readability, and testability.

**Constraints:**
1.  **Functionality First:** The UI must behave **exactly** the same as before. Do not change logic, only move it.
2.  **Style Consistency:** Continue using Tailwind CSS for styling. Keep the visual design (glassmorphism, colors, fonts) identical.
3.  **Props & Events:** Use explicit `defineProps` and `defineEmits` for data flow. Use `v-model` where appropriate.

## Target Architecture

Create the following components in the `src/components/` directory:
1.  **`DoseInput.vue`**: Handles the weekly dose number input and +/- percentage buttons.
2.  **`PillSelector.vue`**: Handles the selection of available pill strengths (1mg, 2mg, etc.).
3.  **`ResultCard.vue`**: Renders a single calculated regimen option (displaying the weekly grid and pill counts).

---

## Implementation Guide

### 1. Component: `src/components/DoseInput.vue`

**Responsibilities:**
*   Display the large input field for `weeklyDose`.
*   Display quick adjustment buttons (-10%, Same, +10%).
*   Handle the `keydown.enter` event.

**Props & Emits:**
*   `modelValue`: Number (The current dose).
*   `@update:modelValue`: Event to update the parent.

**Code Skeleton:**
```vue
<script setup>
defineProps({
  modelValue: { type: Number, default: 0 }
});

const emit = defineEmits(['update:modelValue']);

const handleInput = (event) => {
  emit('update:modelValue', parseFloat(event.target.value));
};

const adjustDose = (percent) => {
  const current = props.modelValue || 0;
  const next = current * (1 + percent / 100);
  // Emit rounded value
  emit('update:modelValue', Math.round(next * 2) / 2);
};
</script>

<template>
  <div class="mb-10 text-center">
    <label class="block text-sm font-semibold text-gray-400 uppercase tracking-wider mb-4">
      Target Weekly Dose (mg)
    </label>
    <!-- Large Input -->
    <div class="relative inline-block group">
      <input 
        type="number" 
        :value="modelValue"
        @input="handleInput"
        @keydown.enter="$emit('calculate')"
        class="input-reset text-6xl sm:text-7xl font-light text-gray-900 w-full max-w-[300px] border-b-2 border-transparent focus:border-blue-500 transition-colors duration-300 pb-2"
        placeholder="0" 
        step="0.5"
      >
      <span class="text-xl text-gray-400 absolute top-4 -right-8 font-light">mg</span>
    </div>

    <!-- Quick Adjustments -->
    <div class="flex justify-center gap-3 mt-6">
      <button @click="adjustDose(-10)" class="...">-10%</button>
      <button @click="adjustDose(0)" class="...">Same</button>
      <button @click="adjustDose(10)" class="...">+10%</button>
    </div>
  </div>
</template>
```

---

### 2. Component: `src/components/PillSelector.vue`

**Responsibilities:**
*   Render the list of pill types (1mg, 2mg, 3mg, 5mg).
*   Toggle selection state.
*   Visual feedback (grayscale when disabled, checkmark when enabled).

**Props & Emits:**
*   `modelValue`: Object/Map `{ 1: boolean, 2: boolean, ... }`
*   `pillTypes`: Array of config objects `{ mg, colorClass }`.
*   `@update:modelValue`: Event to update the specific pill state.

**Code Skeleton:**
```vue
<script setup>
defineProps({
  modelValue: { type: Object, required: true },
  pillTypes: { type: Array, required: true }
});

const emit = defineEmits(['update:modelValue']);

const togglePill = (mg) => {
  // Create a shallow copy to trigger reactivity correctly
  const newValue = { ...props.modelValue, [mg]: !props.modelValue[mg] };
  emit('update:modelValue', newValue);
};
</script>

<template>
  <div>
    <label class="block text-xs font-bold text-gray-400 uppercase tracking-wider mb-4">
      Available Pills
    </label>
    <div class="flex flex-wrap gap-3">
      <button 
        v-for="pill in pillTypes" 
        :key="pill.mg" 
        @click="togglePill(pill.mg)"
        class="..."
        :class="modelValue[pill.mg] ? 'bg-white border-gray-200 shadow-md ring-1 ring-black/5' : 'bg-gray-50 border-transparent opacity-60 grayscale'"
      >
        <!-- Pill Visual -->
        <div class="w-8 h-8 rounded-full ..." :class="pill.colorClass">
          {{ pill.mg }}
        </div>
        <span>{{ pill.mg }} mg</span>
        
        <!-- Checkmark -->
        <div v-if="modelValue[pill.mg]" class="absolute top-0 right-0 ...">
           <!-- SVG Check Icon -->
        </div>
      </button>
    </div>
  </div>
</template>
```

---

### 3. Component: `src/components/ResultCard.vue`

**Responsibilities:**
*   Display the description and actual weekly dose of a result.
*   Render the 7-day grid (Mon-Sun).
*   Display pill counts using the `PillVisual` component (assume it exists at `./PillVisual.vue`).

**Props:**
*   `option`: Object (The data structure returned from Rust).

**Code Skeleton:**
```vue
<script setup>
import PillVisual from './PillVisual.vue';

defineProps({
  option: { type: Object, required: true }
});

const getDayName = (index) => {
  const days = ["จ.", "อ.", "พ.", "พฤ.", "ศ.", "ส.", "อา."];
  return days[index] || "";
};
</script>

<template>
  <div class="glass-card-white bg-white rounded-2xl p-6 shadow-sm border border-gray-100 hover:shadow-md transition-shadow">
    <!-- Header -->
    <div class="flex justify-between mb-4 pb-4 border-b border-gray-50">
      <div>
        <div class="text-xs font-bold text-blue-600 uppercase">Option {{ index + 1 }}</div>
        <div class="font-medium text-lg">{{ option.description }}</div>
      </div>
      <div class="text-right">
        <div class="text-2xl font-bold text-gray-900">
          {{ option.weekly_dose_actual.toFixed(1) }} <span class="text-sm font-normal text-gray-400">mg/wk</span>
        </div>
      </div>
    </div>

    <!-- Schedule Grid -->
    <div class="grid grid-cols-4 sm:grid-cols-7 gap-2 mt-4">
      <div 
        v-for="day in option.weekly_schedule" 
        :key="day.day_index"
        class="rounded-lg border p-2 flex flex-col items-center justify-center text-center"
        :class="day.is_stop_day ? 'bg-gray-50 opacity-70' : 'bg-white border-gray-100'"
      >
        <div class="text-xs font-semibold text-gray-500 mb-2">
          {{ getDayName(day.day_index) }}
        </div>

        <!-- Pills -->
        <div class="flex flex-wrap justify-center mb-2">
          <PillVisual 
            v-for="(pill, pIdx) in day.pills" 
            :key="pIdx" 
            :mg="pill.mg" 
            :is-half="pill.is_half"
          />
        </div>

        <div class="text-sm font-medium text-gray-900">
          {{ day.total_dose.toFixed(1) }} mg
        </div>
      </div>
    </div>

    <!-- Footer Info -->
    <div class="mt-4 pt-4 border-t border-gray-50 text-sm text-gray-600 bg-gray-50/50 rounded-lg p-3">
       <div v-html="option.total_pills_message"></div>
    </div>
  </div>
</template>
```

---

## Step 4: Integration in `App.vue`

After creating the components, refactor `App.vue`:

1.  **Imports:**
    ```javascript
    import DoseInput from './components/DoseInput.vue';
    import PillSelector from './components/PillSelector.vue';
    import ResultCard from './components/ResultCard.vue';
    ```

2.  **Template Replacement:**
    *   Find the "Step 1: Target Dose Input" section and replace it with `<DoseInput v-model="weeklyDose" @calculate="handleCalculation" />`.
    *   Find the "Pill Availability" section and replace it with `<PillSelector v-model="availablePills" :pill-types="pillTypes" />`.
    *   Find the `v-for="(option, index) in results"` loop inside the Results section and replace the inner content with `<ResultCard :option="option" :key="index" />`.

3.  **Cleanup:**
    *   Remove any CSS in `<style>` that is now exclusively used inside the components (e.g., `.pill`, `.glass-card-white` if fully moved). Keep global styles in `App.vue`.

## Validation Steps

1.  **Visual Check:** Run `npm run dev`.
2.  **Interaction Test:**
    *   Change the dose number -> Does the parent state update?
    *   Click a pill button -> Does it toggle?
    *   Click "Generate" -> Do the results render correctly?
3.  **No Regression:** Ensure the `watch` logic in `App.vue` still triggers recalculation when props change (since we are using `v-model`, reactivity should be preserved).

## Final Note
The code should be clean, self-documenting, and strictly follow the Single Responsibility Principle. Each component should be agnostic of the business logic, only handling its own UI state and emitting changes up.