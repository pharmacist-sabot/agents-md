 # Rate Limiting Implementation Guide

This document provides a comprehensive, step-by-step guide for implementing rate limiting in the Drug Requisition System using Lodash throttle/debounce combined with intelligent loading states. This ensures the application remains responsive, prevents accidental double-submissions, and protects backend resources.

---

## 🎯 Implementation Strategy: "Progressive Protection"

We will implement rate limiting at multiple layers:
1. **UI Layer** - Button debouncing and loading states
2. **Action Layer** - Throttled API calls and form submissions
3. **Global Layer** - Request deduplication and circuit breakers
4. **User Feedback** - Clear visual indicators of rate-limited actions

---

## Phase 0: Foundation Setup (Do First)

### Step 0.1: Install Lodash (ES Modules Version)

```bash
# Install lodash with ES module support (tree-shakeable)
npm install lodash-es@^4.17.21

# Install types for TypeScript
npm install -D @types/lodash-es
```

### Step 0.2: Create Rate Limiting Utilities

Create `src/utils/rateLimit.ts`:

```typescript
/**
 * Rate Limiting Utilities
 * Production-grade throttling, debouncing, and request control
 */

import { 
  throttle, 
  debounce, 
  memoize,
  type DebouncedFunc,
  type ThrottleSettings 
} from 'lodash-es'
import { ref, computed, type Ref } from 'vue'

// ==========================================
// Types
// ==========================================

export interface RateLimitOptions {
  /** Maximum requests allowed in window */
  maxRequests?: number
  
  /** Time window in milliseconds */
  windowMs?: number
  
  /** Delay between requests */
  delayMs?: number
  
  /** Show loading state during throttle */
  showLoading?: boolean
  
  /** Allow trailing execution after throttle period */
  trailing?: boolean
  
  /** Allow leading execution on first call */
  leading?: boolean
}

export interface RateLimitState {
  isThrottled: Ref<boolean>
  isLoading: Ref<boolean>
  remainingRequests: Ref<number>
  resetTime: Ref<number | null>
  lastRequestTime: Ref<number | null>
}

export interface ControlledFunction<T extends (...args: any[]) => any> {
  (...args: Parameters<T>): ReturnType<T> | Promise<ReturnType<T>>
  flush: () => ReturnType<T> | undefined
  cancel: () => void
  isPending: () => boolean
}

// ==========================================
// Token Bucket Rate Limiter (Advanced)
// ==========================================

export class TokenBucket {
  private tokens: number
  private lastRefill: number
  private readonly maxTokens: number
  private readonly refillRate: number // tokens per ms
  
  constructor(maxTokens: number, windowMs: number) {
    this.maxTokens = maxTokens
    this.tokens = maxTokens
    this.lastRefill = Date.now()
    this.refillRate = maxTokens / windowMs
  }
  
  consume(tokens: number = 1): boolean {
    this.refill()
    
    if (this.tokens >= tokens) {
      this.tokens -= tokens
      return true
    }
    
    return false
  }
  
  getTimeUntilNextRefill(): number {
    this.refill()
    if (this.tokens >= 1) return 0
    
    const tokensNeeded = 1 - this.tokens
    return Math.ceil(tokensNeeded / this.refillRate)
  }
  
  private refill(): void {
    const now = Date.now()
    const elapsed = now - this.lastRefill
    const tokensToAdd = elapsed * this.refillRate
    
    this.tokens = Math.min(this.maxTokens, this.tokens + tokensToAdd)
    this.lastRefill = now
  }
  
  get state() {
    return {
      tokens: this.tokens,
      maxTokens: this.maxTokens,
      timeUntilNext: this.getTimeUntilNextRefill(),
    }
  }
}

// ==========================================
// Request Deduplication
// ==========================================

export class RequestDeduplicator<T> {
  private pendingRequests = new Map<string, Promise<T>>()
  
  async execute(
    key: string,
    requestFn: () => Promise<T>
  ): Promise<T> {
    // Return existing promise if request is pending
    if (this.pendingRequests.has(key)) {
      return this.pendingRequests.get(key)!
    }
    
    // Create new request
    const promise = requestFn().finally(() => {
      this.pendingRequests.delete(key)
    })
    
    this.pendingRequests.set(key, promise)
    return promise
  }
  
  clear(key?: string): void {
    if (key) {
      this.pendingRequests.delete(key)
    } else {
      this.pendingRequests.clear()
    }
  }
  
  has(key: string): boolean {
    return this.pendingRequests.has(key)
  }
}

// Global deduplicator instance
export const globalDeduplicator = new RequestDeduplicator<any>()

// ==========================================
// Vue Composable: useRateLimit
// ==========================================

export function useRateLimit(options: RateLimitOptions = {}) {
  const {
    maxRequests = 5,
    windowMs = 60000, // 1 minute
    delayMs = 1000,
    showLoading = true,
    trailing = true,
    leading = false,
  } = options
  
  // State
  const isThrottled = ref(false)
  const isLoading = ref(false)
  const remainingRequests = ref(maxRequests)
  const resetTime = ref<number | null>(null)
  const lastRequestTime = ref<number | null>(null)
  
  // Token bucket for precise control
  const bucket = new TokenBucket(maxRequests, windowMs)
  
  // Computed
  const canExecute = computed(() => bucket.consume())
  const timeUntilReset = computed(() => bucket.getTimeUntilNextRefill())
  
  // Track request count in window
  const requestCount = ref(0)
  let windowTimeout: ReturnType<typeof setTimeout> | null = null
  
  const startWindow = () => {
    if (windowTimeout) return
    
    windowTimeout = setTimeout(() => {
      requestCount.value = 0
      remainingRequests.value = maxRequests
      resetTime.value = null
      windowTimeout = null
    }, windowMs)
    
    resetTime.value = Date.now() + windowMs
  }
  
  const recordRequest = () => {
    requestCount.value++
    remainingRequests.value = Math.max(0, maxRequests - requestCount.value)
    lastRequestTime.value = Date.now()
    
    if (requestCount.value === 1) {
      startWindow()
    }
  }
  
  return {
    // State
    isThrottled,
    isLoading,
    remainingRequests,
    resetTime,
    lastRequestTime,
    
    // Computed
    canExecute,
    timeUntilReset,
    
    // Methods
    recordRequest,
    bucket,
  }
}

// ==========================================
// Vue Composable: useThrottledFunction
// ==========================================

export function useThrottledFunction<T extends (...args: any[]) => any>(
  fn: T,
  waitMs: number = 1000,
  options: RateLimitOptions = {}
) {
  const state = useRateLimit({ ...options, delayMs: waitMs })
  
  let throttledFn: DebouncedFunc<T> | null = null
  
  const execute = async (...args: Parameters<T>): Promise<ReturnType<T>> => {
    // Check rate limit
    if (!state.canExecute.value) {
      const waitTime = state.timeUntilReset.value
      throw new RateLimitError(
        `กรุณารอ ${Math.ceil(waitTime / 1000)} วินาที ก่อนลองใหม่`,
        waitTime
      )
    }
    
    // Set loading state
    if (options.showLoading) {
      state.isLoading.value = true
    }
    
    try {
      // Record the request
      state.recordRequest()
      
      // Execute with throttle
      if (!throttledFn) {
        throttledFn = throttle(fn, waitMs, {
          leading: options.leading ?? false,
          trailing: options.trailing ?? true,
        })
      }
      
      const result = await throttledFn(...args)
      return result
    } finally {
      state.isLoading.value = false
    }
  }
  
  const cancel = () => {
    throttledFn?.cancel()
    state.isLoading.value = false
  }
  
  const flush = () => {
    return throttledFn?.flush()
  }
  
  return {
    execute,
    cancel,
    flush,
    ...state,
  }
}

// ==========================================
// Vue Composable: useDebouncedInput
// ==========================================

export function useDebouncedInput<T>(
  initialValue: T,
  onUpdate: (value: T) => void | Promise<void>,
  waitMs: number = 300,
  options: { maxWait?: number; leading?: boolean; trailing?: boolean } = {}
) {
  const value = ref<T>(initialValue)
  const isDebouncing = ref(false)
  const isUpdating = ref(false)
  
  const debouncedUpdate = debounce(
    async (newValue: T) => {
      isDebouncing.value = false
      isUpdating.value = true
      
      try {
        await onUpdate(newValue)
      } finally {
        isUpdating.value = false
      }
    },
    waitMs,
    {
      maxWait: options.maxWait ?? waitMs * 5,
      leading: options.leading ?? false,
      trailing: options.trailing ?? true,
    }
  )
  
  const setValue = (newValue: T) => {
    value.value = newValue
    isDebouncing.value = true
    debouncedUpdate(newValue)
  }
  
  const flush = () => {
    isDebouncing.value = false
    return debouncedUpdate.flush()
  }
  
  const cancel = () => {
    isDebouncing.value = false
    debouncedUpdate.cancel()
  }
  
  return {
    value,
    setValue,
    isDebouncing,
    isUpdating,
    flush,
    cancel,
  }
}

// ==========================================
// Error Classes
// ==========================================

export class RateLimitError extends Error {
  constructor(
    message: string,
    public readonly retryAfterMs: number,
    public readonly limit: number = 0,
    public readonly remaining: number = 0
  ) {
    super(message)
    this.name = 'RateLimitError'
  }
}

export class DuplicateRequestError extends Error {
  constructor(message: string = 'คำขอนี้กำลังดำเนินการอยู่') {
    super(message)
    this.name = 'DuplicateRequestError'
  }
}

// ==========================================
// Utility Functions
// ==========================================

export function formatRetryTime(ms: number): string {
  if (ms < 1000) return 'ไม่กี่วินาที'
  const seconds = Math.ceil(ms / 1000)
  if (seconds < 60) return `${seconds} วินาที`
  const minutes = Math.ceil(seconds / 60)
  return `${minutes} นาที`
}

export function createRequestKey(...parts: (string | number)[]): string {
  return parts.join(':')
}
```

### Step 0.3: Create Loading State Composable

Create `src/composables/useLoadingState.ts`:

```typescript
/**
 * Intelligent loading state management with rate limiting awareness
 */

import { ref, computed, type Ref } from 'vue'

export interface LoadingStateOptions {
  /** Minimum time to show loading (prevents flicker) */
  minDuration?: number
  
  /** Delay before showing loading (prevents flash for fast operations) */
  delay?: number
  
  /** Allow multiple concurrent operations */
  concurrent?: boolean
  
  /** Maximum concurrent operations */
  maxConcurrent?: number
}

export interface LoadingOperation {
  id: string
  name: string
  startTime: number
  abortController?: AbortController
}

export function useLoadingState(options: LoadingStateOptions = {}) {
  const {
    minDuration = 200,
    delay = 50,
    concurrent = false,
    maxConcurrent = 3,
  } = options
  
  const operations = ref<Map<string, LoadingOperation>>(new Map())
  const delayedShow = ref(false)
  let delayTimeout: ReturnType<typeof setTimeout> | null = null
  
  // Computed states
  const isLoading = computed(() => operations.value.size > 0)
  const isDelayedLoading = computed(() => isLoading.value && delayedShow.value)
  const operationCount = computed(() => operations.value.size)
  const canStartOperation = computed(() => 
    concurrent || operations.value.size < maxConcurrent
  )
  
  // Current operation info
  const currentOperation = computed(() => {
    const ops = Array.from(operations.value.values())
    return ops[ops.length - 1] || null
  })
  
  const loadingSince = computed(() => {
    const ops = Array.from(operations.value.values())
    if (ops.length === 0) return null
    return Math.min(...ops.map(op => op.startTime))
  })
  
  const loadingDuration = computed(() => {
    if (!loadingSince.value) return 0
    return Date.now() - loadingSince.value
  })
  
  // Start loading with delay support
  const start = (
    name: string = 'operation',
    id?: string
  ): { id: string; abort: () => void } => {
    const operationId = id || `${name}-${Date.now()}-${Math.random()}`
    
    // Check concurrent limit
    if (!canStartOperation.value) {
      throw new Error(`Maximum concurrent operations (${maxConcurrent}) reached`)
    }
    
    // Create abort controller for cancellation
    const abortController = new AbortController()
    
    const operation: LoadingOperation = {
      id: operationId,
      name,
      startTime: Date.now(),
      abortController,
    }
    
    operations.value.set(operationId, operation)
    
    // Setup delayed show
    if (delay > 0 && !delayTimeout) {
      delayTimeout = setTimeout(() => {
        delayedShow.value = true
        delayTimeout = null
      }, delay)
    } else {
      delayedShow.value = true
    }
    
    return {
      id: operationId,
      abort: () => stop(operationId),
    }
  }
  
  // Stop loading with minimum duration support
  const stop = async (id: string): Promise<void> => {
    const operation = operations.value.get(id)
    if (!operation) return
    
    const elapsed = Date.now() - operation.startTime
    const remaining = Math.max(0, minDuration - elapsed)
    
    // Wait for minimum duration
    if (remaining > 0) {
      await new Promise(resolve => setTimeout(resolve, remaining))
    }
    
    operations.value.delete(id)
    
    // Clear delayed show if no operations
    if (operations.value.size === 0) {
      delayedShow.value = false
      if (delayTimeout) {
        clearTimeout(delayTimeout)
        delayTimeout = null
      }
    }
  }
  
  // Stop all operations
  const stopAll = async (): Promise<void> => {
    const ids = Array.from(operations.value.keys())
    await Promise.all(ids.map(id => stop(id)))
  }
  
  // Abort specific operation
  const abort = (id: string): void => {
    const operation = operations.value.get(id)
    if (operation?.abortController) {
      operation.abortController.abort()
    }
    stop(id)
  }
  
  // Abort all operations
  const abortAll = (): void => {
    operations.value.forEach(op => {
      op.abortController?.abort()
    })
    operations.value.clear()
    delayedShow.value = false
  }
  
  // Check if specific operation is running
  const isOperationLoading = (name: string): boolean => {
    return Array.from(operations.value.values()).some(op => op.name === name)
  }
  
  return {
    // State
    isLoading,
    isDelayedLoading,
    operationCount,
    currentOperation,
    loadingDuration,
    canStartOperation,
    
    // Methods
    start,
    stop,
    stopAll,
    abort,
    abortAll,
    isOperationLoading,
  }
}

// Specialized loading states for common operations
export function useSubmitLoading() {
  return useLoadingState({
    minDuration: 500,
    delay: 100,
    concurrent: false,
  })
}

export function useFetchLoading() {
  return useLoadingState({
    minDuration: 200,
    delay: 50,
    concurrent: true,
    maxConcurrent: 5,
  })
}

export function useSearchLoading() {
  return useLoadingState({
    minDuration: 0,
    delay: 300, // Longer delay for search to prevent flash
    concurrent: false,
  })
}
```

---

## Phase 1: Button Component with Rate Limiting

### Step 1.1: Create SmartButton Component

Create `src/components/SmartButton.vue`:

```vue
<!-- src/components/SmartButton.vue -->
<template>
  <button
    :type="props.type"
    :class="buttonClasses"
    :disabled="isDisabled"
    @click="handleClick"
  >
    <!-- Loading State -->
    <span v-if="showLoadingState" class="btn-loading">
      <i class="fas fa-circle-notch fa-spin"></i>
      <span v-if="props.loadingText">{{ props.loadingText }}</span>
      <span v-else-if="countdown > 0">
        รอ {{ countdown }} วินาที
      </span>
    </span>
    
    <!-- Default Slot -->
    <span v-else class="btn-content">
      <slot></slot>
    </span>
    
    <!-- Rate Limit Indicator -->
    <span 
      v-if="showRateLimitIndicator && countdown > 0" 
      class="rate-limit-badge"
    >
      {{ countdown }}s
    </span>
  </button>
</template>

<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import { useThrottledFunction, useLoadingState, formatRetryTime } from '@/utils/rateLimit'

interface Props {
  type?: 'button' | 'submit' | 'reset'
  variant?: 'primary' | 'secondary' | 'success' | 'danger' | 'warning' | 'info'
  size?: 'sm' | 'md' | 'lg'
  disabled?: boolean
  
  // Rate limiting
  throttleMs?: number
  maxClicks?: number
  windowMs?: number
  
  // Loading
  loading?: boolean
  loadingText?: string
  showLoadingIndicator?: boolean
  
  // Async handling
  async?: boolean
  
  // Click handler
  onClick?: () => void | Promise<void>
}

const props = withDefaults(defineProps<Props>(), {
  type: 'button',
  variant: 'primary',
  size: 'md',
  throttleMs: 1000,
  maxClicks: 10,
  windowMs: 60000,
  showLoadingIndicator: true,
})

const emit = defineEmits<{
  click: []
  error: [error: Error]
}>()

// Loading state
const { 
  isLoading, 
  isDelayedLoading, 
  start, 
  stop 
} = useLoadingState({
  minDuration: props.async ? 300 : 0,
  delay: 50,
})

// Rate limiting
const clickCount = ref(0)
const lastClickTime = ref(0)
const countdown = ref(0)
let countdownInterval: ReturnType<typeof setInterval> | null = null

const isRateLimited = computed(() => {
  if (clickCount.value < props.maxClicks) return false
  
  const elapsed = Date.now() - lastClickTime.value
  return elapsed < props.windowMs
})

const isDisabled = computed(() => {
  return props.disabled || 
         isLoading.value || 
         (isRateLimited.value && countdown.value > 0)
})

const showLoadingState = computed(() => {
  return (props.loading || isDelayedLoading.value) && props.showLoadingIndicator
})

const showRateLimitIndicator = computed(() => {
  return isRateLimited.value && !isLoading.value
})

const buttonClasses = computed(() => [
  'btn',
  `btn-${props.variant}`,
  `btn-${props.size}`,
  {
    'btn-loading': showLoadingState.value,
    'btn-rate-limited': isRateLimited.value,
  }
])

// Throttled click handler
const throttledClick = useThrottledFunction(
  async () => {
    // Check rate limit
    if (isRateLimited.value) {
      const retryAfter = props.windowMs - (Date.now() - lastClickTime.value)
      throw new Error(`กรุณารอ ${formatRetryTime(retryAfter)} ก่อนลองใหม่`)
    }
    
    // Record click
    clickCount.value++
    lastClickTime.value = Date.now()
    
    // Start countdown if approaching limit
    if (clickCount.value >= props.maxClicks - 2) {
      startCountdown()
    }
    
    // Execute handler
    if (props.onClick) {
      const op = start('button-click')
      
      try {
        await props.onClick()
      } finally {
        await stop(op.id)
      }
    }
    
    emit('click')
  },
  props.throttleMs,
  { showLoading: true }
)

const handleClick = async () => {
  try {
    await throttledClick.execute()
  } catch (error) {
    emit('error', error instanceof Error ? error : new Error(String(error)))
  }
}

const startCountdown = () => {
  if (countdownInterval) return
  
  countdown.value = Math.ceil(props.windowMs / 1000)
  
  countdownInterval = setInterval(() => {
    countdown.value--
    
    if (countdown.value <= 0) {
      clickCount.value = 0
      clearInterval(countdownInterval!)
      countdownInterval = null
    }
  }, 1000)
}

// Cleanup
watch(() => props.loading, (newVal) => {
  if (!newVal && isLoading.value) {
    // External loading cleared
  }
})
</script>

<style scoped>
.btn {
  position: relative;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  transition: all 0.2s ease;
}

.btn-loading {
  cursor: wait;
  opacity: 0.8;
}

.btn-rate-limited {
  background-color: var(--warning-color);
  border-color: var(--warning-color);
}

.btn-content,
.btn-loading {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.rate-limit-badge {
  position: absolute;
  top: -8px;
  right: -8px;
  background: var(--danger-color);
  color: white;
  font-size: 0.7rem;
  padding: 0.15rem 0.4rem;
  border-radius: 999px;
  font-weight: 700;
  animation: pulse 1s infinite;
}

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.1); }
}

/* Size variants */
.btn-sm {
  padding: 0.4rem 0.8rem;
  font-size: 0.875rem;
}

.btn-md {
  padding: 0.7rem 1.4rem;
  font-size: 1rem;
}

.btn-lg {
  padding: 1rem 2rem;
  font-size: 1.125rem;
}
</style>
```

---

## Phase 2: Form Submission with Rate Limiting

### Step 2.1: Create useFormSubmit Composable

Create `src/composables/useFormSubmit.ts`:

```typescript
/**
 * Form submission with rate limiting and duplicate prevention
 */

import { ref, computed } from 'vue'
import { useLoadingState, useThrottledFunction, globalDeduplicator, createRequestKey } from '@/utils/rateLimit'
import { useToast } from '@/composables/useToast'
import type { RouteLocationRaw } from 'vue-router'

export interface FormSubmitOptions<T> {
  /** Submit handler */
  submitFn: (data: T) => Promise<void>
  
  /** Success message */
  successMessage?: string
  
  /** Redirect after success */
  redirectTo?: RouteLocationRaw
  
  /** Throttle time between submissions */
  throttleMs?: number
  
  /** Deduplication key */
  dedupeKey?: string
  
  /** Allow retry on error */
  allowRetry?: boolean
  
  /** Transform data before submit */
  transform?: (data: T) => T | Promise<T>
  
  /** Validate before submit */
  validate?: (data: T) => string[] | Promise<string[]>
  
  /** On success callback */
  onSuccess?: () => void
  
  /** On error callback */
  onError?: (error: Error) => void
}

export interface FormSubmitState<T> {
  isSubmitting: ReturnType<typeof useLoadingState>['isLoading']
  isThrottled: ReturnType<typeof useThrottledFunction>['isThrottled']
  submitCount: ReturnType<typeof useThrottledFunction>['remainingRequests']
  lastSubmitTime: ReturnType<typeof useThrottledFunction>['lastRequestTime']
  
  submit: (data: T) => Promise<boolean>
  reset: () => void
}

export function useFormSubmit<T extends Record<string, any>>(
  options: FormSubmitOptions<T>
) {
  const {
    submitFn,
    successMessage = 'บันทึกสำเร็จ',
    redirectTo,
    throttleMs = 2000,
    dedupeKey,
    allowRetry = true,
    transform,
    validate,
    onSuccess,
    onError,
  } = options
  
  const { success: showSuccess, error: showError } = useToast()
  
  const submitCount = ref(0)
  const lastError = ref<Error | null>(null)
  
  // Loading state with minimum duration for UX
  const loading = useLoadingState({
    minDuration: 500,
    delay: 100,
    concurrent: false, // Forms should not submit concurrently
  })
  
  // Throttled submission
  const throttledSubmit = useThrottledFunction(
    async (data: T): Promise<boolean> => {
      // Check for duplicate submission
      if (dedupeKey && globalDeduplicator.has(dedupeKey)) {
        throw new Error('คำขอนี้กำลังดำเนินการอยู่ กรุณารอสักครู่')
      }
      
      // Validate
      if (validate) {
        const errors = await validate(data)
        if (errors.length > 0) {
          throw new Error(errors[0])
        }
      }
      
      // Transform
      const finalData = transform ? await transform(data) : data
      
      // Execute with deduplication
      const executeSubmit = async () => {
        await submitFn(finalData)
      }
      
      if (dedupeKey) {
        await globalDeduplicator.execute(dedupeKey, executeSubmit)
      } else {
        await executeSubmit()
      }
      
      return true
    },
    throttleMs,
    { 
      showLoading: true,
      maxRequests: 3,
      windowMs: 60000,
    }
  )
  
  const submit = async (data: T): Promise<boolean> => {
    const op = loading.start('form-submit', 'form-submit')
    lastError.value = null
    
    try {
      const result = await throttledSubmit.execute(data)
      
      if (result) {
        submitCount.value++
        showSuccess(successMessage, { title: 'สำเร็จ' })
        onSuccess?.()
        
        // Redirect if specified
        if (redirectTo) {
          // Use router or window.location based on type
          if (typeof redirectTo === 'string' && redirectTo.startsWith('http')) {
            window.location.href = redirectTo
          } else {
            // Assume router navigation
            const router = (await import('@/router')).default
            await router.push(redirectTo)
          }
        }
      }
      
      return result
    } catch (error) {
      const err = error instanceof Error ? error : new Error(String(error))
      lastError.value = err
      
      // Don't show toast for validation errors (handled by form)
      if (!err.message.includes('จำเป็นต้องกรอก')) {
        showError(err.message, { 
          title: 'ไม่สำเร็จ',
          persistent: allowRetry,
        })
      }
      
      onError?.(err)
      return false
    } finally {
      await loading.stop(op.id)
    }
  }
  
  const reset = () => {
    submitCount.value = 0
    lastError.value = null
    throttledSubmit.cancel?.()
  }
  
  return {
    // State
    isSubmitting: loading.isLoading,
    isThrottled: throttledSubmit.isThrottled,
    submitCount,
    lastError: computed(() => lastError.value),
    
    // Methods
    submit,
    reset,
    
    // Rate limit info
    canSubmit: throttledSubmit.canExecute,
    timeUntilNextSubmit: throttledSubmit.timeUntilReset,
  }
}
```

### Step 2.2: Update Login.vue with Form Submit

```vue
<!-- src/views/Login.vue -->
<template>
  <div class="login-wrapper">
    <div class="card login-container">
      <!-- ... header ... -->
      
      <form @submit.prevent="handleSubmit">
        <!-- ... inputs with validation ... -->
        
        <div v-if="submitError" class="alert alert-error">
          {{ submitError }}
        </div>
        
        <SmartButton
          type="submit"
          variant="primary"
          :throttle-ms="2000"
          :loading="isSubmitting"
          loading-text="กำลังเข้าสู่ระบบ..."
          :disabled="!isValid"
          async
          @click="handleSubmit"
          @error="handleButtonError"
        >
          <i class="fas fa-sign-in-alt"></i>
          เข้าสู่ระบบ
        </SmartButton>
        
        <!-- ... register link ... -->
      </form>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import { useRouter } from 'vue-router'
import { useAuth } from '@/composables/useAuth'
import { useFormSubmit } from '@/composables/useFormSubmit'
import { useFormValidation } from '@/composables/useValidation'
import { loginSchema } from '@/schemas'
import SmartButton from '@/components/SmartButton.vue'
import type { LoginInput } from '@/schemas'

const router = useRouter()
const { login } = useAuth()

const { 
  formData, 
  validate, 
  errors, 
  isValid 
} = useFormValidation(loginSchema, { 
  email: '', 
  password: '' 
})

const {
  submit,
  isSubmitting,
  lastError,
  canSubmit,
} = useFormSubmit<LoginInput>({
  submitFn: async (data) => {
    await login(data.email, data.password)
  },
  successMessage: 'เข้าสู่ระบบสำเร็จ',
  redirectTo: '/',
  throttleMs: 2000,
  dedupeKey: 'login',
  validate: () => {
    const result = validate()
    return result ? [] : Object.values(errors.value)
  },
})

const submitError = computed(() => lastError.value?.message)
const timeUntilRetry = computed(() => 
  canSubmit.value ? 0 : 2000 // Simplified, use actual from composable
)

const handleSubmit = async () => {
  if (!isValid.value) return
  await submit(formData)
}

const handleButtonError = (error: Error) => {
  // Error already handled by useFormSubmit, but can add custom logic here
  console.log('Button error:', error.message)
}
</script>
```

---

## Phase 3: Search and Input Debouncing

### Step 3.1: Create SearchInput Component

Create `src/components/SearchInput.vue`:

```vue
<!-- src/components/SearchInput.vue -->
<template>
  <div class="search-input-wrapper">
    <div class="input-group" :class="{ 'is-searching': isSearching }">
      <i class="fas fa-search search-icon"></i>
      
      <input
        :value="displayValue"
        @input="handleInput"
        @blur="flush"
        :placeholder="props.placeholder"
        :disabled="props.disabled"
        class="search-input"
        type="text"
      />
      
      <button
        v-if="displayValue"
        @click="clear"
        class="clear-btn"
        type="button"
      >
        <i class="fas fa-times"></i>
      </button>
      
      <span v-if="isDebouncing" class="debounce-indicator">
        <i class="fas fa-ellipsis-h fa-pulse"></i>
      </span>
    </div>
    
    <span v-if="isSearching" class="search-status">
      กำลังค้นหา...
    </span>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import { useDebouncedInput } from '@/utils/rateLimit'

interface Props {
  modelValue: string
  placeholder?: string
  disabled?: boolean
  debounceMs?: number
  minChars?: number
}

const props = withDefaults(defineProps<Props>(), {
  placeholder: 'ค้นหา...',
  debounceMs: 300,
  minChars: 2,
})

const emit = defineEmits<{
  'update:modelValue': [value: string]
  search: [value: string]
  'search-start': []
  'search-end': []
}>()

const {
  value: displayValue,
  setValue,
  isDebouncing,
  isUpdating: isSearching,
  flush,
  cancel,
} = useDebouncedInput(
  props.modelValue,
  async (newValue) => {
    if (newValue.length > 0 && newValue.length < props.minChars) {
      return // Don't search below minimum
    }
    
    emit('update:modelValue', newValue)
    emit('search-start')
    
    try {
      emit('search', newValue)
    } finally {
      emit('search-end')
    }
  },
  props.debounceMs,
  { maxWait: 2000 }
)

const handleInput = (event: Event) => {
  const target = event.target as HTMLInputElement
  setValue(target.value)
}

const clear = () => {
  cancel()
  setValue('')
  flush()
}
</script>

<style scoped>
.search-input-wrapper {
  position: relative;
}

.input-group {
  position: relative;
  display: flex;
  align-items: center;
}

.search-icon {
  position: absolute;
  left: 1rem;
  color: var(--text-muted);
  pointer-events: none;
}

.search-input {
  width: 100%;
  padding: 0.7rem 2.5rem;
  padding-left: 2.5rem;
  border: 1px solid var(--border-color);
  border-radius: var(--border-radius);
  transition: all 0.2s ease;
}

.search-input:focus {
  outline: none;
  border-color: var(--primary-color);
  box-shadow: 0 0 0 3px rgba(0, 90, 156, 0.1);
}

.is-searching .search-input {
  padding-right: 3rem;
}

.clear-btn {
  position: absolute;
  right: 2.5rem;
  background: none;
  border: none;
  color: var(--text-muted);
  cursor: pointer;
  padding: 0.25rem;
}

.clear-btn:hover {
  color: var(--danger-color);
}

.debounce-indicator {
  position: absolute;
  right: 1rem;
  color: var(--primary-color);
}

.search-status {
  display: block;
  font-size: 0.8rem;
  color: var(--text-muted);
  margin-top: 0.25rem;
}
</style>
```

### Step 3.2: Update ItemManagement.vue with Debounced Search

```vue
<!-- src/views/admin/ItemManagement.vue -->
<template>
  <div class="container">
    <!-- ... header ... -->
    
    <div class="card">
      <div class="toolbar">
        <SearchInput
          v-model="searchTerm"
          placeholder="ค้นหารายการยา..."
          :debounce-ms="300"
          :min-chars="2"
          @search="performSearch"
        />
        
        <SmartButton
          variant="success"
          :throttle-ms="500"
          @click="openAddModal"
        >
          <i class="fas fa-plus"></i>
          เพิ่มรายการ
        </SmartButton>
      </div>
      
      <!-- ... table ... -->
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import SearchInput from '@/components/SearchInput.vue'
import SmartButton from '@/components/SmartButton.vue'
import { itemService } from '@/services/supabaseService'
import { useFetchLoading } from '@/composables/useLoadingState'
import type { Item } from '@/types'

const searchTerm = ref('')
const items = ref<Item[]>([])
const filteredItems = computed(() => {
  if (!searchTerm.value) return items.value
  return items.value.filter(item =>
    item.name.toLowerCase().includes(searchTerm.value.toLowerCase())
  )
})

const { isLoading: isSearching, start, stop } = useFetchLoading()

const performSearch = async (term: string) => {
  const op = start('item-search')
  
  try {
    // Client-side search for now, can be API search later
    // Debounce prevents this from running on every keystroke
    console.log('Searching for:', term)
  } finally {
    await stop(op.id)
  }
}

const openAddModal = () => {
  // Throttled to prevent accidental double-clicks
  showAddItemModal.value = true
}
</script>
```

---

## Phase 4: API Request Rate Limiting

### Step 4.1: Create API Rate Limiter

Create `src/services/apiRateLimiter.ts`:

```typescript
/**
 * API Request Rate Limiter
 * Prevents API abuse and handles 429 responses
 */

import { TokenBucket, RequestDeduplicator, RateLimitError } from '@/utils/rateLimit'
import { useToast } from '@/composables/useToast'

// ==========================================
// Rate Limit Configuration by Endpoint
// ==========================================

interface EndpointConfig {
  bucket: TokenBucket
  deduplicator: RequestDeduplicator<any>
  priority: 'high' | 'normal' | 'low'
}

const endpointConfigs = new Map<string, EndpointConfig>()

function getOrCreateConfig(endpoint: string): EndpointConfig {
  if (!endpointConfigs.has(endpoint)) {
    // Different limits for different endpoint types
    const isWrite = /(insert|update|delete|create)/i.test(endpoint)
    const isAuth = /(login|register|auth)/i.test(endpoint)
    
    const maxRequests = isAuth ? 5 : isWrite ? 10 : 30
    const windowMs = isAuth ? 60000 : 60000 // 1 minute
    
    endpointConfigs.set(endpoint, {
      bucket: new TokenBucket(maxRequests, windowMs),
      deduplicator: new RequestDeduplicator<any>(),
      priority: isWrite ? 'high' : isAuth ? 'high' : 'normal',
    })
  }
  
  return endpointConfigs.get(endpoint)!
}

// ==========================================
// Main Rate Limit Function
// ==========================================

export async function withRateLimit<T>(
  endpoint: string,
  requestFn: () => Promise<T>,
  options: {
    dedupeKey?: string
    skipRateLimit?: boolean
    signal?: AbortSignal
  } = {}
): Promise<T> {
  const config = getOrCreateConfig(endpoint)
  
  // Check rate limit
  if (!options.skipRateLimit && !config.bucket.consume()) {
    const retryAfter = config.bucket.getTimeUntilNextRefill()
    throw new RateLimitError(
      `คำขอมากเกินไป กรุณารอ ${Math.ceil(retryAfter / 1000)} วินาที`,
      retryAfter,
      10,
      0
    )
  }
  
  // Check deduplication
  const dedupeKey = options.dedupeKey || endpoint
  
  // Execute with deduplication
  return config.deduplicator.execute(dedupeKey, async () => {
    // Check abort signal
    if (options.signal?.aborted) {
      throw new Error('Request aborted')
    }
    
    const result = await requestFn()
    return result
  })
}

// ==========================================
// Response Handler for 429 Errors
// ==========================================

export function handleRateLimitResponse(
  response: Response,
  endpoint: string
): never {
  const retryAfter = response.headers.get('Retry-After')
  const retryMs = retryAfter ? parseInt(retryAfter, 10) * 1000 : 60000
  
  // Update bucket to respect server rate limit
  const config = getOrCreateConfig(endpoint)
  // Force bucket empty to prevent more requests
  // (TokenBucket doesn't support this directly, would need implementation)
  
  const { error } = useToast()
  error(
    `คำขอมากเกินไป กรุณารอ ${Math.ceil(retryMs / 1000)} วินาที`,
    { title: 'ถูกจำกัดการใช้งานชั่วคราว' }
  )
  
  throw new RateLimitError('Rate limited by server', retryMs)
}

// ==========================================
// Cleanup Utilities
// ==========================================

export function clearRateLimit(endpoint?: string): void {
  if (endpoint) {
    endpointConfigs.delete(endpoint)
  } else {
    endpointConfigs.clear()
  }
}

export function getRateLimitStatus(): Array<{
  endpoint: string
  tokens: number
  maxTokens: number
  timeUntilReset: number
}> {
  return Array.from(endpointConfigs.entries()).map(([endpoint, config]) => ({
    endpoint,
    ...config.bucket.state,
  }))
}
```

### Step 4.2: Update Supabase Service to Use Rate Limiting

```typescript
// src/services/supabaseService.ts (updated with rate limiting)

import { withRateLimit } from './apiRateLimiter'

export const requisitionService = {
  async create(pcuId: number, periodId: number, requesterId: string) {
    return withRateLimit(
      'requisitions:create',
      async () => {
        const { data, error } = await supabase
          .from('requisitions_drugcupsabot')
          .insert({ pcu_id: pcuId, period_id: periodId, requester_id: requesterId, status: 'draft' })
          .select('id')
          .single()
        
        if (error) throw error
        return { data, error: null }
      },
      {
        dedupeKey: `create:${pcuId}:${periodId}`,
      }
    )
  },
  
  async updateStatus(id: number, status: RequisitionStatus) {
    return withRateLimit(
      `requisitions:update:${id}`,
      async () => {
        const { error } = await supabase
          .from('requisitions_drugcupsabot')
          .update({ status, ...(status === 'submitted' ? { submitted_at: new Date().toISOString() } : {}) })
          .eq('id', id)
        
        if (error) throw error
        return { error: null }
      },
      {
        dedupeKey: `update:${id}`,
      }
    )
  },
}

export const itemService = {
  async update(item: Partial<Item> & { id: number }) {
    return withRateLimit(
      `items:update:${item.id}`,
      async () => {
        const { error } = await supabase
          .from('items_drugcupsabot')
          .update(item)
          .eq('id', item.id)
        
        if (error) throw error
        return { error: null }
      },
      {
        dedupeKey: `update:${item.id}`,
      }
    )
  },
}
```

---

## Phase 5: Global Request Interceptor

### Step 5.1: Create Axios/Fetch Interceptor (if needed)

Create `src/services/httpInterceptor.ts`:

```typescript
/**
 * HTTP Interceptor for global rate limiting
 * Works with fetch API (Supabase uses fetch internally)
 */

import { RateLimitError } from '@/utils/rateLimit'

const originalFetch = window.fetch

// Request tracking
const pendingRequests = new Map<string, AbortController>()

window.fetch = async function(
  input: RequestInfo | URL,
  init?: RequestInit
): Promise<Response> {
  const url = typeof input === 'string' ? input : input.toString()
  const method = init?.method || 'GET'
  
  // Create unique key for this request
  const requestKey = `${method}:${url}:${JSON.stringify(init?.body)}`
  
  // Cancel previous identical request (automatic deduplication)
  if (pendingRequests.has(requestKey) && method !== 'GET') {
    pendingRequests.get(requestKey)!.abort()
  }
  
  // Create new abort controller
  const controller = new AbortController()
  const signal = controller.signal
  
  // Merge with existing signal if provided
  if (init?.signal) {
    init.signal.addEventListener('abort', () => controller.abort())
  }
  
  pendingRequests.set(requestKey, controller)
  
  try {
    const response = await originalFetch(input, {
      ...init,
      signal,
    })
    
    // Handle 429 Too Many Requests
    if (response.status === 429) {
      const retryAfter = response.headers.get('Retry-After')
      throw new RateLimitError(
        'Rate limited',
        retryAfter ? parseInt(retryAfter, 10) * 1000 : 60000
      )
    }
    
    return response
  } finally {
    pendingRequests.delete(requestKey)
  }
}
```

---

## 🚨 Critical Implementation Rules

### DO:
- ✅ Always use SmartButton for action buttons
- ✅ Use useFormSubmit for all form submissions
- ✅ Debounce search inputs with SearchInput component
- ✅ Use deduplication for identical API calls
- ✅ Show visual feedback during throttling
- ✅ Respect server 429 responses
- ✅ Clean up abort controllers on unmount

### DON'T:
- ❌ Allow unlimited button clicks
- ❌ Submit forms without throttling
- ❌ Search on every keystroke without debounce
- ❌ Allow duplicate API requests
- ❌ Ignore rate limit headers from server
- ❌ Block UI without loading indicators

---

## 📋 Migration Checklist Per Component

```
For each component with user actions:
[ ] Replace <button> with <SmartButton>
[ ] Add throttleMs prop (start with 1000ms)
[ ] Use useFormSubmit for form submissions
[ ] Add loading states
[ ] Test rapid clicking (should be blocked)
[ ] Test error recovery

For each search/filter:
[ ] Replace input with <SearchInput>
[ ] Set appropriate debounceMs (300-500ms)
[ ] Add minChars if needed
[ ] Test typing speed (should not flash)

For API calls:
[ ] Wrap with withRateLimit()
[ ] Add dedupeKey for mutating operations
[ ] Handle RateLimitError in catch blocks
```

---

## 🎯 Success Criteria

Rate limiting implementation is complete when:

1. No button can be double-clicked accidentally
2. Forms throttle submissions (2+ second delay)
3. Search debounces at 300ms minimum
4. API calls deduplicate automatically
5. Visual feedback shows during all rate-limited states
6. Server 429 errors handled gracefully
7. No "stuck" loading states
8. Users can cancel long-running operations

---

## 🆘 Troubleshooting

### Issue: "Button still allows rapid clicks"
**Solution**: Check that SmartButton is used and throttleMs is set. Verify isLoading state is properly wired.

### Issue: "Search feels laggy"
**Solution**: Reduce debounceMs to 200ms or use leading: true option for immediate first search.

### Issue: "Duplicate requests still happening"
**Solution**: Ensure dedupeKey is unique and stable. Check that RequestDeduplicator is shared instance.

### Issue: "Rate limit error not showing retry time"
**Solution**: Verify RateLimitError is properly caught and message includes formatted time.

---

**Remember**: Rate limiting is about user experience as much as system protection. Clear feedback prevents frustration while protecting resources.