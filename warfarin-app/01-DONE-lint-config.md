# Task: Setup Strict ESLint & Prettier

## Objective
Establish a world-class code quality and formatting standard using ESLint and Prettier. This ensures type safety, readability, and consistency across the codebase.

**Constraints:**
1.  **Strict Mode:** Do not allow `any` types, unused variables, or `console.log` (in production logic) without explicit suppression.
2.  **Auto-Fix:** Configure tools to auto-fix formatting issues where possible.
3.  **Compatibility:** Use **Flat Config** (`eslint.config.js`) as required by the Vite 7+ stack.

---

## Phase 1: Installation

Install the necessary development dependencies. Run this command in the project root:

```bash
npm install -D \
  eslint \
  @eslint/js \
  typescript-eslint \
  @vue/eslint-config-typescript \
  @vue/eslint-config-prettier \
  eslint-plugin-vue \
  prettier
```

---

## Phase 2: Configuration Files

### 1. Prettier Configuration (`prettier.config.js`)
Create a `prettier.config.js` file in the root directory. Enforce consistent formatting rules (Single quotes, no semicolons, trailing commas).

```javascript
export default {
  semi: false,
  singleQuote: true,
  tabWidth: 2,
  trailingComma: 'es5',
  printWidth: 100,
  endOfLine: 'lf',
  arrowParens: 'always',
}
```

### 2. ESLint Configuration (`eslint.config.js`)
Create an `eslint.config.js` file in the root directory. This replaces the old `.eslintrc.js`.

```javascript
import js from '@eslint/js'
import pluginVue from 'eslint-plugin-vue'
import tsParser from 'vue-eslint-parser'
import * as parserTypeScript from '@typescript-eslint/parser'
import * as parser from '@typescript-eslint/parser'
import configTypeScript from '@typescript-eslint/eslint-plugin'
import vueTsEslintConfig from '@vue/eslint-config-typescript'

export default [
  // 1. Global Ignore patterns
  {
    ignores: ['**/dist/**', '**/node_modules/**', '**/warfarin_logic/pkg/**', '**/.vercel/**'],
  },

  // 2. Base JS Config
  js.configs.recommended,

  // 3. Vue 3 + TypeScript Config
  ...pluginVue.configs['flat/recommended'],
  ...vueTsEslintConfig(),
  ...configTypeScript.configs.recommended,

  // 4. Custom Overrides & Strict Rules
  {
    files: ['**/*.{js,mjs,cjs,ts,vue}'],
    languageOptions: {
      ecmaVersion: 'latest',
      sourceType: 'module',
      parser: parserTypeScript, // Use TypeScript parser
      parserOptions: {
        ecmaVersion: 'latest',
        extraFileExtensions: ['.vue'],
        parser: {
          ts: parser,
          '<template>': tsParser, // Parse Vue templates
        },
      },
    },
    rules: {
      // --- Strict TypeScript Rules ---
      '@typescript-eslint/no-explicit-any': 'error', // Ban 'any'
      '@typescript-eslint/no-unused-vars': [
        'error',
        { argsIgnorePattern: '^_', varsIgnorePattern: '^_' },
      ],
      '@typescript-eslint/explicit-function-return-type': 'off', // Too strict for Vue setup scripts usually, keep off for now
      '@typescript-eslint/no-non-null-assertion': 'warn',

      // --- Vue Specific ---
      'vue/multi-word-component-names': 'off', // Allow single-word component names
      'vue/require-default-prop': 'off',
      'vue/v-on-event-hyphenation': 'off', // Allow kebab-case in templates
      'vue/attributes-order': 'error', // Enforce attribute ordering for consistency

      // --- General Code Quality ---
      'no-console': process.env.NODE_ENV === 'production' ? 'warn' : 'off',
      'no-debugger': process.env.NODE_ENV === 'production' ? 'error' : 'off',
      'no-var': 'error',
      'prefer-const': 'error',
      eqeqeq: ['error', 'always'], // Require ===
    },
  },

  // 5. Prettier Integration (Must be last to override conflicting rules)
  {
    name: 'app/vue-to-prettier',
    plugins: {
      prettier: 'eslint-plugin-prettier', // Note: You might need to install eslint-plugin-prettier or rely on @vue/eslint-config-prettier
    },
    rules: {
      // Turn off ESLint rules that conflict with Prettier
      'prettier/prettier': 'warn',
      '@typescript-eslint/indent': 'off',
      'vue/html-indent': 'off',
      'vue/max-attributes-per-line': 'off',
    },
  },
]
```
*Note: If you see errors regarding 'eslint-plugin-prettier', you can install it (`npm install -D eslint-plugin-prettier`) or simply rely on `@vue/eslint-config-prettier` which usually handles conflicts without explicit rules. The configuration above assumes `@vue/eslint-config-prettier` handles the suppression, which is standard in Vue stacks.*

---

## Phase 3: Update `package.json`

Add the following scripts to `scripts` in `package.json` to automate linting:

```json
{
  "scripts": {
    "lint": "eslint . --max-warnings 0",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write src/"
  }
}
```

---

## Phase 4: Execution & Auto-Fixing

Once the configuration files are created, the Agent must perform the following actions:

1.  **Run Linter with Fix:** Execute `npm run lint:fix` to automatically correct all formatting and fixable logic errors in the `src/` directory.
2.  **Check for Residual Errors:** Run `npm run lint` (without fix).
    *   **If errors persist:** Review them carefully. Common issues in strict mode might be implicit `any` types in the Rust WASM binding imports (e.g., `import type { ... } from '../warfarin_logic/...'`).
    *   **Action:** Create `src/env.d.ts` or type declarations for the WASM module if `any` errors occur, rather than disabling the rule globally.
3.  **Format Files:** Run `npm run format` to ensure Prettier is applied to everything.

## Validation

1.  The codebase should be free of console logs/warnings during a dev build.
2.  All `.vue` and `.ts` files must strictly follow the Prettier formatting.
3.  The development server (`npm run dev`) should start without ESLint warnings blocking the terminal.
4.  The build process (`npm run build`) should succeed, assuming logic correctness.