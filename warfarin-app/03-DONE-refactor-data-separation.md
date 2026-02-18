# Architectural Refactoring - Data & View Separation

## Mission Context
We are refactoring the architecture of the Warfarin App to eliminate the use of `v-html` in the Vue frontend. Generating HTML strings in Rust and injecting them is a dangerous Anti-Pattern (XSS risk) and prevents the use of native Vue features (animations, event handling, reactivity).

**Objective:**
1.  **Rust (Logic Layer):** Stop generating HTML strings. Return clean, type-safe JSON Data Structures.
2.  **Vue (Presentation Layer):** Accept raw data and use Vue's Template System (`v-for`, Components) to render the UI entirely.

---

## Phase 1: Rust Backend Transformation (`warfarin_logic/src/lib.rs`)

### 1.1 Define New Data Structures
Create new structs to represent the schedule data instead of HTML strings. Ensure `Serialize` is derived.

```rust
#[derive(Serialize, Deserialize, Clone, Debug)]
pub struct PillRenderData {
    pub mg: u8,
    pub count: u8,
    pub is_half: bool,
}

#[derive(Serialize, Deserialize, Clone, Debug)]
pub struct DaySchedule {
    pub day_index: usize, // 0 = Mon, ..., 6 = Sun
    pub total_dose: f64,
    pub pills: Vec<PillRenderData>,
    pub is_stop_day: bool, // Flag to style the background differently
}
```

### 1.2 Update `FinalOutput` Struct
Modify the struct that sends data back to the frontend. Replace the HTML string field with a data array.

```rust
#[derive(Serialize)]
pub struct FinalOutput {
    pub description: String,
    pub weekly_dose_actual: f64,
    // CHANGE: Replace 'weekly_schedule_html: String' with this array:
    pub weekly_schedule: Vec<DaySchedule>,
    pub total_pills_message: String, // Leave as is for now
}
```

### 1.3 Refactor `render_option` Function
Instead of calling `render_day` to build HTML strings, map the data to the new `DaySchedule` struct.

```rust
// DELETE the old 'render_day' function entirely.

// MODIFY render_option to build Data instead of HTML
fn render_option(option: &DosageOption, input: &CalculationInput) -> FinalOutput {
    // ... (Keep existing description logic) ...

    // Create schedule data array
    let mut weekly_schedule: Vec<DaySchedule> = Vec::new();
    
    // Loop through days 0..6 (Mon..Sun)
    for day_idx in 0..7 {
        let combo = match &option.option_type {
            OptionType::Uniform(c) => c,
            OptionType::NonUniform(cw) => &cw[day_idx],
        };

        // Determine if this is a stop day based on logic
        let is_stop_day = option.stop_days.contains(&day_idx) || 
                          (combo.is_empty() && day_idx > 0);
        
        let total_dose: f64 = combo.iter()
            .map(|p| p.mg as f64 * p.count as f64 * if p.half { 0.5 } else { 1.0 })
            .sum();

        let pills: Vec<PillRenderData> = combo.iter().map(|p| PillRenderData {
            mg: p.mg,
            count: p.count,
            is_half: p.half,
        }).collect();

        weekly_schedule.push(DaySchedule {
            day_index: day_idx,
            total_dose,
            pills,
            is_stop_day,
        });
    }
    
    FinalOutput {
        description,
        weekly_dose_actual: option.weekly_dose_actual,
        weekly_schedule, // Send the Array instead of HTML String
        total_pills_message: "...".to_string(), // Keep existing logic
    }
}
```

**Validation Step:** Verify that the Rust code compiles successfully (`wasm-pack build`) and the JSON output structure matches the new types.

---

## Phase 2: Vue Frontend Transformation (`src/App.vue`)

### 2.1 Create Reusable Component: `PillVisual.vue`
Create a dedicated component for rendering a single pill to ensure code reusability and cleanliness.

**Path:** `src/components/PillVisual.vue`
```vue
<script setup>
defineProps({
  mg: { type: Number, required: true },
  isHalf: { type: Boolean, default: false }
})
</script>

<template>
  <span 
    class="pill inline-block rounded-full shadow-inner flex items-center justify-center" 
    :class="[
      `pill-${mg}`,
      isHalf ? 'pill-half-left' : ''
    ]"
  >
    <span class="text-[10px] font-bold text-white/90">
      {{ isHalf ? '½' : mg }}
    </span>
  </span>
</template>

<style scoped>
.pill {
  width: 28px;
  height: 28px;
  margin: 2px;
  vertical-align: middle;
}
.pill-half-left {
  clip-path: polygon(0 0, 50% 0, 50% 100%, 0% 100%);
}
/* Color classes mapping based on design system */
.pill-1 { @apply bg-gray-300 border border-gray-200; }
.pill-2 { @apply bg-orange-300; }
.pill-3 { @apply bg-sky-400; }
.pill-5 { @apply bg-pink-400; }
</style>
```

### 2.2 Refactor `App.vue` Results Section
In the `<template>` section of `App.vue`:

1.  Import the component: `import PillVisual from './components/PillVisual.vue'`
2.  **DELETE** the entire `<div class="rust-html-content" v-html="option.weekly_schedule_html"></div>` block.
3.  **ADD** the following loop to render from the new Data:

```vue
<!-- Results Section -->
<div v-for="(option, index) in results" :key="index" class="...">
  
  <!-- Header ... (Keep existing) -->

  <!-- NEW: Grid Rendered from JSON Data -->
  <div class="grid grid-cols-4 sm:grid-cols-7 gap-2 mt-4">
    <div 
      v-for="day in option.weekly_schedule" 
      :key="day.day_index"
      class="rounded-lg border p-2 flex flex-col items-center justify-center text-center transition-colors duration-300"
      :class="[
        'border-gray-100 bg-white',
        day.is_stop_day ? 'bg-gray-50 opacity-70 border-dashed' : ''
      ]"
    >
      <!-- Day Name -->
      <div class="text-xs font-semibold text-gray-500 mb-2">
        {{ getDayName(day.day_index) }}
      </div>

      <!-- Pills List -->
      <div class="flex flex-wrap justify-center mb-2">
        <PillVisual 
          v-for="(pill, pIdx) in day.pills" 
          :key="pIdx" 
          :mg="pill.mg" 
          :is-half="pill.is_half"
        />
      </div>

      <!-- Dose Text -->
      <div class="text-sm font-medium text-gray-900">
        {{ day.total_dose.toFixed(1) }} mg
      </div>
    </div>
  </div>
  
  <!-- Footer ... (Keep existing) -->
</div>
```

### 2.3 Add Helper Function
Add this utility function inside the `<script setup>` section of `src/App.vue` to map index to day names.

```javascript
const getDayName = (index) => {
  const days = ["จ.", "อ.", "พ.", "พฤ.", "ศ.", "ส.", "อา."];
  return days[index] || "";
};
```

---

## Phase 3: Safety & Validation

### Security Verification
1.  **XSS Prevention:** Audit `App.vue` to ensure **zero** usage of `v-html`. The UI must be rendered entirely via data binding.
2.  **Type Safety:** Ensure `weekly_schedule` data is accessed safely in the template (handle nulls/undefined if necessary, though the Rust struct guarantees existence).

### Build Verification
After refactoring, run the following commands to confirm system integrity:
1.  `wasm-pack build ./warfarin_logic --target web` (Must compile without errors)
2.  `npm run dev` (The web page must render correctly)
3.  Perform a manual test: Calculate a dose and verify that the visual UI matches the previous HTML output exactly.

## Final Note
The refactored code must be clean, strictly separated (Rust = Logic/Math, Vue = UI/Animation), and 100% free from XSS vulnerabilities. String injection into the DOM is strictly prohibited.