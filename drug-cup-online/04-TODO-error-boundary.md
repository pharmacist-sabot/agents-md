 # Error Boundary Implementation Guide

This document provides a comprehensive, step-by-step guide for implementing Error Boundaries in the Drug Requisition System using Vue 3's native error handling capabilities. This ensures the application remains functional even when unexpected errors occur, following production-grade standards used by top-tier tech companies.

---

## 🎯 Implementation Strategy: "Defense in Depth"

We will implement multiple layers of error protection:
1. **Component-Level Error Boundaries** - Catch errors in specific sections
2. **View-Level Error Boundaries** - Protect entire pages
3. **Global Error Handler** - Catch everything else
4. **Async Error Handling** - Handle Promise rejections

---

## Phase 0: Foundation Setup (Do First)

### Step 0.1: Create Core Error Boundary Component

Create `src/components/ErrorBoundary.vue`:

```vue
<!-- src/components/ErrorBoundary.vue -->
<template>
  <div v-if="hasError" class="error-boundary">
    <div class="error-content" :class="[`severity-${props.severity}`]">
      <div class="error-icon">
        <i class="fas fa-exclamation-triangle"></i>
      </div>
      
      <h2 class="error-title">{{ title }}</h2>
      
      <p class="error-message">{{ displayMessage }}</p>
      
      <div v-if="props.showDetails && errorDetails" class="error-details">
        <details>
          <summary>รายละเอียดทางเทคนิค (สำหรับผู้ดูแลระบบ)</summary>
          <pre class="error-stack">{{ errorDetails }}</pre>
        </details>
      </div>

      <div class="error-actions">
        <button 
          v-if="props.recoverable" 
          @click="recover" 
          class="btn btn-primary"
        >
          <i class="fas fa-redo"></i>
          {{ props.recoverButtonText }}
        </button>
        
        <button 
          @click="reload" 
          class="btn btn-secondary"
          :class="{ 'btn-full': !props.recoverable }"
        >
          <i class="fas fa-sync-alt"></i>
          รีเฟรชหน้าเว็บ
        </button>
        
        <router-link 
          v-if="props.showHomeButton" 
          to="/" 
          class="btn btn-secondary"
        >
          <i class="fas fa-home"></i>
          กลับหน้าหลัก
        </router-link>
      </div>

      <p v-if="props.reportable" class="error-report">
        <small>
          หากปัญหายังคงอยู่ กรุณาติดต่อทีมผู้ดูแลระบบ 
          <br>
          รหัสข้อผิดพลาด: <code>{{ errorId }}</code>
        </small>
      </p>
    </div>
  </div>
  
  <slot v-else></slot>
</template>

<script setup lang="ts">
import { ref, onErrorCaptured, computed, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { useToast } from '@/composables/useToast'

// ==========================================
// Types
// ==========================================

type ErrorSeverity = 'low' | 'medium' | 'high' | 'critical'

interface Props {
  // Display options
  title?: string
  message?: string
  severity?: ErrorSeverity
  showDetails?: boolean
  showHomeButton?: boolean
  
  // Recovery options
  recoverable?: boolean
  recoverButtonText?: string
  onRecover?: () => void | Promise<void>
  
  // Reporting
  reportable?: boolean
  fallbackRoute?: string
  
  // Development
  debug?: boolean
}

// ==========================================
// Props with Defaults
// ==========================================

const props = withDefaults(defineProps<Props>(), {
  title: 'เกิดข้อผิดพลาด',
  message: 'ระบบไม่สามารถดำเนินการได้ กรุณาลองใหม่อีกครั้ง',
  severity: 'medium',
  showDetails: import.meta.env.DEV, // Auto-show in dev mode
  showHomeButton: true,
  recoverable: true,
  recoverButtonText: 'ลองใหม่',
  reportable: true,
  fallbackRoute: '/',
  debug: false,
})

// ==========================================
// State
// ==========================================

const hasError = ref(false)
const error = ref<Error | null>(null)
const errorInfo = ref<string>('')
const errorId = ref(generateErrorId())
const recoveryAttempts = ref(0)
const maxRecoveryAttempts = 3

const route = useRoute()
const router = useRouter()
const { error: showErrorToast } = useToast()

// ==========================================
// Computed
// ==========================================

const displayMessage = computed(() => {
  if (props.message && props.message !== 'เกิดข้อผิดพลาด') {
    return props.message
  }
  
  // Context-aware messages based on route
  const routeMessages: Record<string, string> = {
    '/pcu/requisition': 'ไม่สามารถโหลดแบบฟอร์มเบิกยาได้',
    '/admin/dashboard': 'ไม่สามารถโหลดข้อมูลแดชบอร์ดได้',
    '/admin/item-management': 'ไม่สามารถโหลดรายการยาได้',
  }
  
  const matchedRoute = Object.keys(routeMessages).find(r => 
    route.path.startsWith(r)
  )
  
  return matchedRoute 
    ? routeMessages[matchedRoute] 
    : error.value?.message || props.message
})

const errorDetails = computed(() => {
  if (!error.value) return ''
  
  let details = `Error: ${error.value.name}\n`
  details += `Message: ${error.value.message}\n`
  details += `Component: ${errorInfo.value}\n`
  details += `Route: ${route.fullPath}\n`
  details += `Time: ${new Date().toISOString()}\n`
  
  if (props.debug && error.value.stack) {
    details += `\nStack Trace:\n${error.value.stack}`
  }
  
  return details
})

// ==========================================
// Error Capture (Vue 3 Native)
// ==========================================

onErrorCaptured((err: unknown, instance, info) => {
  // Don't capture if already handling
  if (hasError.value) return false
  
  // Filter out non-error objects
  if (!(err instanceof Error)) {
    console.warn('Non-error thrown:', err)
    return true // Propagate to parent
  }
  
  // Log for monitoring
  console.error('Error Boundary Caught:', {
    error: err,
    component: instance?.$options?.name || 'Anonymous',
    info,
    route: route.fullPath,
    errorId: errorId.value,
  })
  
  // Set error state
  error.value = err
  errorInfo.value = info
  hasError.value = true
  recoveryAttempts.value++
  
  // Report to monitoring service (if configured)
  reportError(err, info)
  
  // Show toast for non-critical errors
  if (props.severity === 'low') {
    showErrorToast('เกิดข้อผิดพลาดเล็กน้อย กำลังพยายามกู้คืน...', {
      title: 'ข้อผิดพลาด',
    })
    // Auto-recover for low severity
    setTimeout(recover, 1000)
    return false // Don't show boundary UI
  }
  
  // Prevent error propagation for handled severities
  return false
})

// ==========================================
// Actions
// ==========================================

async function recover(): Promise<void> {
  if (recoveryAttempts.value >= maxRecoveryAttempts) {
    // Force reload if too many attempts
    reload()
    return
  }
  
  try {
    // Call custom recovery if provided
    if (props.onRecover) {
      await props.onRecover()
    }
    
    // Reset error state
    hasError.value = false
    error.value = null
    errorInfo.value = ''
    errorId.value = generateErrorId()
    
  } catch (recoveryError) {
    console.error('Recovery failed:', recoveryError)
    error.value = recoveryError instanceof Error 
      ? recoveryError 
      : new Error('Recovery failed')
  }
}

function reload(): void {
  window.location.reload()
}

function reportError(err: Error, info: string): void {
  // Send to error tracking service (Sentry, LogRocket, etc.)
  // Example: Sentry.captureException(err, { extra: { info, errorId: errorId.value } })
  
  // For now, just log to console in production build
  if (import.meta.env.PROD) {
    // TODO: Implement actual error reporting service
    fetch('/api/log-error', {
      method: 'POST',
      body: JSON.stringify({
        errorId: errorId.value,
        message: err.message,
        stack: err.stack,
        component: info,
        route: route.fullPath,
        userAgent: navigator.userAgent,
        timestamp: new Date().toISOString(),
      }),
    }).catch(console.error)
  }
}

function generateErrorId(): string {
  return `ERR-${Date.now()}-${Math.random().toString(36).substr(2, 9).toUpperCase()}`
}

// ==========================================
// Watchers
// ==========================================

// Reset error when route changes (fresh start)
watch(() => route.path, () => {
  if (hasError.value && props.recoverable) {
    recover()
  }
})
</script>

<style scoped>
.error-boundary {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 2rem;
  background: linear-gradient(135deg, #f5f1ff 0%, #e6f7ff 100%);
}

.error-content {
  max-width: 600px;
  width: 100%;
  background: white;
  border-radius: 1rem;
  padding: 2.5rem;
  text-align: center;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.1);
  border-top: 4px solid var(--warning-color);
}

.error-content.severity-low {
  border-top-color: var(--info-color);
}

.error-content.severity-medium {
  border-top-color: var(--warning-color);
}

.error-content.severity-high {
  border-top-color: var(--danger-color);
}

.error-content.severity-critical {
  border-top-color: #7c2d12;
  background: #fef2f2;
}

.error-icon {
  font-size: 4rem;
  color: var(--warning-color);
  margin-bottom: 1.5rem;
}

.severity-high .error-icon,
.severity-critical .error-icon {
  color: var(--danger-color);
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.7; }
}

.error-title {
  font-size: 1.75rem;
  color: var(--text-color);
  margin-bottom: 1rem;
}

.error-message {
  color: var(--text-muted);
  font-size: 1.1rem;
  line-height: 1.6;
  margin-bottom: 1.5rem;
}

.error-details {
  margin: 1.5rem 0;
  text-align: left;
}

.error-details summary {
  cursor: pointer;
  color: var(--primary-color);
  font-weight: 500;
  padding: 0.5rem;
  border-radius: 0.25rem;
  transition: background 0.2s;
}

.error-details summary:hover {
  background: var(--bg-color);
}

.error-stack {
  background: #1a1a2e;
  color: #eee;
  padding: 1rem;
  border-radius: 0.5rem;
  font-size: 0.8rem;
  overflow-x: auto;
  white-space: pre-wrap;
  word-break: break-word;
  margin-top: 0.5rem;
}

.error-actions {
  display: flex;
  gap: 1rem;
  justify-content: center;
  flex-wrap: wrap;
  margin: 1.5rem 0;
}

.btn-full {
  flex: 1;
}

.error-report {
  margin-top: 1.5rem;
  padding-top: 1.5rem;
  border-top: 1px solid var(--border-color);
  color: var(--text-muted);
}

.error-report code {
  background: var(--bg-color);
  padding: 0.25rem 0.5rem;
  border-radius: 0.25rem;
  font-family: monospace;
  font-size: 0.9em;
}

@media (max-width: 480px) {
  .error-boundary {
    padding: 1rem;
  }
  
  .error-content {
    padding: 1.5rem;
  }
  
  .error-icon {
    font-size: 3rem;
  }
  
  .error-title {
    font-size: 1.4rem;
  }
  
  .error-actions {
    flex-direction: column;
  }
  
  .error-actions .btn {
    width: 100%;
  }
}
</style>
```

### Step 0.2: Create Async Error Boundary

Create `src/components/AsyncErrorBoundary.vue` for async operations:

```vue
<!-- src/components/AsyncErrorBoundary.vue -->
<template>
  <div v-if="loading" class="async-loading">
    <div class="spinner">
      <i class="fas fa-circle-notch fa-spin"></i>
    </div>
    <p>{{ loadingText }}</p>
  </div>
  
  <ErrorBoundary
    v-else-if="error"
    :severity="errorSeverity"
    :message="errorMessage"
    :recoverable="true"
    :on-recover="retry"
    recover-button-text="ลองโหลดใหม่"
  >
    <template #default>
      <!-- This won't render due to error -->
    </template>
  </ErrorBoundary>
  
  <slot v-else :data="data" :refresh="refresh"></slot>
</template>

<script setup lang="ts">
import { ref, onMounted, watch } from 'vue'
import ErrorBoundary from './ErrorBoundary.vue'

// ==========================================
// Types
// ==========================================

interface Props {
  // Async function to execute
  fetchFn: () => Promise<any>
  
  // Options
  immediate?: boolean
  loadingText?: string
  errorSeverity?: 'low' | 'medium' | 'high'
  retryCount?: number
  
  // Watch dependencies
  watch?: any[]
}

const props = withDefaults(defineProps<Props>(), {
  immediate: true,
  loadingText: 'กำลังโหลดข้อมูล...',
  errorSeverity: 'medium',
  retryCount: 3,
})

// ==========================================
// State
// ==========================================

const loading = ref(false)
const error = ref<Error | null>(null)
const errorMessage = ref('')
const data = ref<any>(null)
const retryAttempt = ref(0)

// ==========================================
// Methods
// ==========================================

async function execute(): Promise<void> {
  if (loading.value) return
  
  loading.value = true
  error.value = null
  
  try {
    const result = await props.fetchFn()
    data.value = result
    retryAttempt.value = 0
  } catch (err) {
    retryAttempt.value++
    
    error.value = err instanceof Error ? err : new Error(String(err))
    
    // Custom error messages based on error type
    if (err instanceof TypeError && err.message.includes('fetch')) {
      errorMessage.value = 'ไม่สามารถเชื่อมต่อกับเซิร์ฟเวอร์ได้ กรุณาตรวจสอบการเชื่อมต่ออินเทอร์เน็ต'
    } else if (err instanceof Error && err.message.includes('timeout')) {
      errorMessage.value = 'การโหลดข้อมูลใช้เวลานานเกินไป กรุณาลองใหม่อีกครั้ง'
    } else {
      errorMessage.value = 'ไม่สามารถโหลดข้อมูลได้'
    }
    
    // Auto-retry for transient errors
    if (retryAttempt.value < props.retryCount && isRetryableError(err)) {
      setTimeout(() => {
        if (!loading.value) execute()
      }, 1000 * retryAttempt.value) // Exponential backoff
    }
  } finally {
    loading.value = false
  }
}

function isRetryableError(err: unknown): boolean {
  if (!(err instanceof Error)) return false
  
  const retryableMessages = [
    'network',
    'timeout',
    'fetch',
    'temporary',
    '503',
    '502',
  ]
  
  return retryableMessages.some(msg => 
    err.message.toLowerCase().includes(msg)
  )
}

function retry(): void {
  execute()
}

function refresh(): void {
  data.value = null
  execute()
}

// ==========================================
// Lifecycle
// ==========================================

onMounted(() => {
  if (props.immediate) {
    execute()
  }
})

// Watch for changes and re-fetch
if (props.watch) {
  watch(props.watch, () => {
    execute()
  }, { deep: true })
}

defineExpose({
  refresh,
  retry,
  execute,
})
</script>

<style scoped>
.async-loading {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 300px;
  color: var(--text-muted);
}

.spinner {
  font-size: 2rem;
  margin-bottom: 1rem;
  color: var(--primary-color);
}

.async-loading p {
  font-size: 1rem;
}
</style>
```

---

## Phase 1: Global Error Handling Layer

### Step 1.1: Create Global Error Plugin

Create `src/plugins/globalErrorHandler.ts`:

```typescript
/**
 * Global Error Handler Plugin
 * Catches all unhandled errors across the application
 */

import type { App } from 'vue'
import { useToast } from '@/composables/useToast'

export interface ErrorHandlerOptions {
  // Report to external service (Sentry, etc.)
  reportErrors?: boolean
  
  // Show toast for non-fatal errors
  showToastNotifications?: boolean
  
  // Log to console in production
  logInProduction?: boolean
  
  // Custom error transform
  transformError?: (error: Error) => Error
}

export const defaultOptions: ErrorHandlerOptions = {
  reportErrors: import.meta.env.PROD,
  showToastNotifications: true,
  logInProduction: false,
}

export function installGlobalErrorHandler(
  app: App,
  options: ErrorHandlerOptions = {}
): void {
  const opts = { ...defaultOptions, ...options }
  
  // ==========================================
  // Vue App Error Handler
  // ==========================================
  
  app.config.errorHandler = (err, instance, info) => {
    const error = normalizeError(err)
    
    // Log error
    logError('Vue Error', error, {
      component: instance?.$options?.name,
      info,
      timestamp: new Date().toISOString(),
    })
    
    // Show toast if not already caught by boundary
    if (opts.showToastNotifications && !isHandledByBoundary(instance)) {
      showErrorNotification(error, 'Vue Component Error')
    }
    
    // Report to external service
    if (opts.reportErrors) {
      reportToService(error, { type: 'vue', info })
    }
  }
  
  // ==========================================
  // Global Promise Rejection Handler
  // ==========================================
  
  window.addEventListener('unhandledrejection', (event) => {
    const error = normalizeError(event.reason)
    
    logError('Unhandled Promise Rejection', error, {
      stack: error.stack,
    })
    
    showErrorNotification(error, 'Async Operation Failed')
    
    if (opts.reportErrors) {
      reportToService(error, { type: 'promise' })
    }
    
    // Prevent default browser handling
    event.preventDefault()
  })
  
  // ==========================================
  // Global Error Handler
  // ==========================================
  
  window.addEventListener('error', (event) => {
    const error = new Error(event.message)
    error.stack = `${event.filename}:${event.lineno}:${event.colno}`
    
    logError('Global Error', error, {
      filename: event.filename,
      lineno: event.lineno,
      colno: event.colno,
    })
    
    // Only show toast for non-script errors
    if (event.error && opts.showToastNotifications) {
      showErrorNotification(error, 'Runtime Error')
    }
    
    if (opts.reportErrors) {
      reportToService(error, { 
        type: 'global',
        filename: event.filename,
        lineno: event.lineno,
      })
    }
  })
  
  // ==========================================
  // Network Error Handling
  // ==========================================
  
  window.addEventListener('offline', () => {
    const { warning } = useToast()
    warning('คุณออฟไลน์อยู่ บางฟีเจอร์อาจไม่พร้อมใช้งาน', {
      title: 'ไม่มีการเชื่อมต่อ',
      persistent: true,
    })
  })
  
  window.addEventListener('online', () => {
    const { success } = useToast()
    success('เชื่อมต่ออินเทอร์เน็ตสำเร็จ', {
      title: 'ออนไลน์แล้ว',
    })
  })
  
  // ==========================================
  // Supabase Realtime Error Handling
  // ==========================================
  
  // TODO: Add Supabase channel error handling when implementing realtime
}

// ==========================================
// Helper Functions
// ==========================================

function normalizeError(err: unknown): Error {
  if (err instanceof Error) return err
  if (typeof err === 'string') return new Error(err)
  return new Error('Unknown error occurred')
}

function logError(type: string, error: Error, meta: Record<string, any>): void {
  console.group(`[${type}] ${error.message}`)
  console.error('Error:', error)
  console.log('Metadata:', meta)
  console.groupEnd()
}

function isHandledByBoundary(instance: any): boolean {
  // Check if component is inside ErrorBoundary
  let parent = instance?.$parent
  while (parent) {
    if (parent.$options?.name === 'ErrorBoundary') return true
    parent = parent.$parent
  }
  return false
}

function showErrorNotification(error: Error, context: string): void {
  // Use setTimeout to avoid "setup called without active instance" error
  setTimeout(() => {
    const { error: showError } = useToast()
    showError(`${context}: ${error.message}`, {
      title: 'ข้อผิดพลาด',
      timeout: 8000,
    })
  }, 0)
}

function reportToService(error: Error, context: Record<string, any>): void {
  // Placeholder for Sentry/LogRocket integration
  // Example:
  // Sentry.captureException(error, { extra: context })
  
  // For now, send to custom endpoint
  if (import.meta.env.PROD) {
    fetch('/api/errors', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        message: error.message,
        stack: error.stack,
        context,
        url: window.location.href,
        userAgent: navigator.userAgent,
        timestamp: new Date().toISOString(),
      }),
      // Don't wait for response, fire and forget
      keepalive: true,
    }).catch(() => {
      // Silent fail - don't cause infinite error loop
    })
  }
}
```

### Step 1.2: Create Error Utilities

Create `src/utils/errorHandling.ts`:

```typescript
/**
 * Error handling utilities for consistent error management
 */

import type { RouteLocationRaw } from 'vue-router'

// ==========================================
// Error Classes
// ==========================================

export class AppError extends Error {
  constructor(
    message: string,
    public code: string,
    public severity: 'low' | 'medium' | 'high' | 'critical' = 'medium',
    public recoverable: boolean = true,
    public redirectTo?: RouteLocationRaw
  ) {
    super(message)
    this.name = 'AppError'
  }
}

export class NetworkError extends AppError {
  constructor(message: string = 'การเชื่อมต่อล้มเหลว') {
    super(message, 'NETWORK_ERROR', 'high', true)
    this.name = 'NetworkError'
  }
}

export class AuthError extends AppError {
  constructor(message: string = 'การตรวจสอบสิทธิ์ล้มเหลว') {
    super(message, 'AUTH_ERROR', 'critical', false, '/login')
    this.name = 'AuthError'
  }
}

export class ValidationError extends AppError {
  constructor(
    message: string = 'ข้อมูลไม่ถูกต้อง',
    public fields: Record<string, string[]> = {}
  ) {
    super(message, 'VALIDATION_ERROR', 'low', true)
    this.name = 'ValidationError'
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string = 'ข้อมูล') {
    super(`${resource}ไม่พบ`, 'NOT_FOUND', 'medium', true)
    this.name = 'NotFoundError'
  }
}

// ==========================================
// Error Guards
// ==========================================

export function isAppError(error: unknown): error is AppError {
  return error instanceof AppError
}

export function isNetworkError(error: unknown): error is NetworkError {
  return error instanceof NetworkError
}

export function isAuthError(error: unknown): error is AuthError {
  return error instanceof AuthError
}

// ==========================================
// Error Transformers
// ==========================================

export function transformSupabaseError(error: any): AppError {
  // Handle specific Supabase error codes
  const code = error?.code || 'UNKNOWN'
  const message = error?.message || 'เกิดข้อผิดพลาดที่ไม่คาดคิด'
  
  switch (code) {
    case 'PGRST116': // Row not found
      return new NotFoundError('รายการที่ร้องขอ')
    
    case '23505': // Unique violation
      return new ValidationError('ข้อมูลซ้ำซ้อน', {
        general: ['มีข้อมูลนี้อยู่ในระบบแล้ว']
      })
    
    case '42501': // RLS violation
      return new AuthError('คุณไม่มีสิทธิ์เข้าถึงข้อมูลนี้')
    
    case 'PGRST301': // JWT expired
      return new AuthError('เซสชันหมดอายุ กรุณาเข้าสู่ระบบใหม่')
    
    case 'NETWORK_ERROR':
    case 'ETIMEDOUT':
      return new NetworkError()
    
    default:
      return new AppError(message, code)
  }
}

// ==========================================
// Safe Execution Utilities
// ==========================================

export async function safeAsync<T>(
  promise: Promise<T>,
  errorHandler?: (error: Error) => void
): Promise<[T | null, Error | null]> {
  try {
    const result = await promise
    return [result, null]
  } catch (err) {
    const error = err instanceof Error ? err : new Error(String(err))
    errorHandler?.(error)
    return [null, error]
  }
}

export function safeSync<T>(fn: () => T, fallback: T): T {
  try {
    return fn()
  } catch {
    return fallback
  }
}

// ==========================================
// Retry Logic
// ==========================================

interface RetryOptions {
  attempts?: number
  delay?: number
  backoff?: number
  retryable?: (error: Error) => boolean
}

export async function withRetry<T>(
  fn: () => Promise<T>,
  options: RetryOptions = {}
): Promise<T> {
  const {
    attempts = 3,
    delay = 1000,
    backoff = 2,
    retryable = () => true,
  } = options
  
  let lastError: Error
  
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn()
    } catch (err) {
      lastError = err instanceof Error ? err : new Error(String(err))
      
      if (!retryable(lastError) || i === attempts - 1) {
        throw lastError
      }
      
      await sleep(delay * Math.pow(backoff, i))
    }
  }
  
  throw lastError!
}

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms))
}
```

---

## Phase 2: Integration with Existing Architecture

### Step 2.1: Wrap App.vue with Global Error Boundary

Update `src/App.vue`:

```vue
<!-- src/App.vue -->
<template>
  <ErrorBoundary
    severity="critical"
    :recoverable="true"
    :show-details="isDev"
    title="ระบบเกิดข้อผิดพลาดร้ายแรง"
    message="ไม่สามารถโหลดแอปพลิเคชันได้ กรุณารีเฟรชหน้าเว็บหรือติดต่อผู้ดูแลระบบ"
    @recover="handleAppRecovery"
  >
    <div v-if="!auth.isLoggedIn || isPublicPage">
      <router-view />
    </div>
    <AdminLayout v-else-if="auth.isAdmin" />
    <PcuLayout v-else />
  </ErrorBoundary>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import { useRoute } from 'vue-router'
import { useAuthStore } from '@/store/auth'
import ErrorBoundary from '@/components/ErrorBoundary.vue'
import AdminLayout from '@/layouts/AdminLayout.vue'
import PcuLayout from '@/layouts/PcuLayout.vue'

const auth = useAuthStore()
const route = useRoute()

const isPublicPage = computed(() => {
  const publicPages = ['Login', 'Home', 'PrintRequisition', 'PrintRequisitionSummary']
  return publicPages.includes(route.name as string)
})

const isDev = computed(() => import.meta.env.DEV)

const handleAppRecovery = async () => {
  // Clear auth state and reload
  await auth.logout()
  window.location.href = '/login'
}
</script>
```

### Step 2.2: Wrap Individual Views

Create `src/components/ViewErrorBoundary.vue` (simplified for views):

```vue
<!-- src/components/ViewErrorBoundary.vue -->
<template>
  <ErrorBoundary
    :severity="props.severity"
    :recoverable="true"
    :show-home-button="true"
    :title="props.title"
    @recover="emit('recover')"
  >
    <slot></slot>
  </ErrorBoundary>
</template>

<script setup lang="ts">
import ErrorBoundary from './ErrorBoundary.vue'

interface Props {
  title?: string
  severity?: 'low' | 'medium' | 'high' | 'critical'
}

const props = withDefaults(defineProps<Props>(), {
  title: 'ไม่สามารถโหลดหน้านี้ได้',
  severity: 'medium',
})

const emit = defineEmits<{
  recover: []
}>()
</script>
```

Update `src/views/pcu/PcuDashboard.vue`:

```vue
<!-- src/views/pcu/PcuDashboard.vue -->
<template>
  <ViewErrorBoundary
    title="ไม่สามารถโหลดแดชบอร์ด รพ.สต. ได้"
    @recover="refreshData"
  >
    <div class="container">
      <!-- existing template -->
    </div>
  </ViewErrorBoundary>
</template>

<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import { useRouter } from 'vue-router'
import ViewErrorBoundary from '@/components/ViewErrorBoundary.vue'
import { useAuth } from '@/composables/useAuth'
import { useToast } from '@/composables/useToast'
import { periodService, requisitionService } from '@/services/supabaseService'
import { safeAsync, withRetry, NetworkError } from '@/utils/errorHandling'
import type { RequisitionPeriod, Requisition } from '@/types'

// ... setup ...

const refreshData = async () => {
  // Clear and reload all data
  openPeriods.value = []
  existingRequisitions.value = []
  await loadData()
}

const loadData = async () => {
  // Use safeAsync for error handling
  const [periodsData, periodsErr] = await safeAsync(
    withRetry(() => periodService.getOpen(), {
      attempts: 3,
      retryable: (e) => e instanceof NetworkError,
    })
  )
  
  if (periodsErr) {
    // ErrorBoundary will catch and display
    throw periodsErr
  }
  
  openPeriods.value = periodsData || []
  
  // ... rest of loading logic ...
}
</script>
```

### Step 2.3: Wrap Data Tables and Complex Components

Create `src/components/DataErrorBoundary.vue` for data components:

```vue
<!-- src/components/DataErrorBoundary.vue -->
<template>
  <div v-if="error" class="data-error">
    <div class="error-icon">
      <i class="fas fa-database"></i>
    </div>
    <p class="error-text">{{ displayMessage }}</p>
    <button @click="retry" class="btn btn-sm btn-primary">
      <i class="fas fa-redo"></i>
      ลองใหม่
    </button>
  </div>
  
  <slot v-else :error="setError" :clear="clearError"></slot>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'

interface Props {
  fallbackMessage?: string
}

const props = withDefaults(defineProps<Props>(), {
  fallbackMessage: 'ไม่สามารถโหลดข้อมูลได้',
})

const error = ref<Error | null>(null)

const displayMessage = computed(() => {
  if (!error.value) return ''
  
  // User-friendly messages
  if (error.value.message.includes('network')) {
    return 'การเชื่อมต่อล้มเหลว กรุณาตรวจสอบอินเทอร์เน็ต'
  }
  
  return error.value.message || props.fallbackMessage
})

const setError = (err: Error) => {
  error.value = err
}

const clearError = () => {
  error.value = null
}

const retry = () => {
  clearError()
  // Parent component should handle retry via slot props
}

defineExpose({
  setError,
  clearError,
})
</script>

<style scoped>
.data-error {
  text-align: center;
  padding: 2rem;
  color: var(--text-muted);
}

.error-icon {
  font-size: 2rem;
  color: var(--warning-color);
  margin-bottom: 0.5rem;
}

.error-text {
  margin-bottom: 1rem;
}
</style>
```

Use in `src/views/admin/AdminDashboard.vue`:

```vue
<!-- src/views/admin/AdminDashboard.vue -->
<template>
  <div class="container">
    <h2>Admin Dashboard - จัดการใบเบิก</h2>

    <DataErrorBoundary ref="errorBoundary" fallback-message="ไม่สามารถโหลดข้อมูลใบเบิกได้">
      <template #default="{ setError }">
        <AsyncErrorBoundary 
          :fetch-fn="fetchRequisitions"
          v-slot="{ data: requisitions }"
        >
          <!-- Render requisitions -->
        </AsyncErrorBoundary>
      </template>
    </DataErrorBoundary>
  </div>
</template>
```

---

## Phase 3: Service Layer Integration

### Step 3.1: Update Supabase Service with Error Handling

Create `src/services/baseService.ts`:

```typescript
/**
 * Base service with error handling and retry logic
 */

import { withRetry, transformSupabaseError, safeAsync } from '@/utils/errorHandling'
import { useToast } from '@/composables/useToast'

interface ServiceOptions {
  retry?: boolean
  retryAttempts?: number
  showToast?: boolean
}

const defaultOptions: ServiceOptions = {
  retry: true,
  retryAttempts: 3,
  showToast: false, // Don't show toast by default, let boundary handle
}

export async function executeQuery<T>(
  queryFn: () => Promise<{ data: T | null; error: any }>,
  options: ServiceOptions = {}
): Promise<{ data: T | null; error: Error | null }> {
  const opts = { ...defaultOptions, ...options }
  
  const execute = async () => {
    const result = await queryFn()
    
    if (result.error) {
      throw transformSupabaseError(result.error)
    }
    
    return result.data
  }
  
  try {
    const data = opts.retry
      ? await withRetry(execute, { attempts: opts.retryAttempts })
      : await execute()
    
    return { data, error: null }
  } catch (err) {
    const error = err instanceof Error ? err : new Error(String(err))
    
    // Log for monitoring
    console.error('Service Error:', error)
    
    return { data: null, error }
  }
}

export async function executeMutation<T>(
  mutationFn: () => Promise<{ data: T | null; error: any }>,
  successMessage?: string,
  options: ServiceOptions = {}
): Promise<{ data: T | null; error: Error | null }> {
  const result = await executeQuery(mutationFn, options)
  
  if (result.data && successMessage && options.showToast !== false) {
    const { success } = useToast()
    success(successMessage, { title: 'สำเร็จ' })
  }
  
  return result
}
```

Update services to use base service:

```typescript
// src/services/supabaseService.ts (updated)
import { executeQuery, executeMutation } from './baseService'

export const requisitionService = {
  async getById(id: number) {
    return executeQuery(() => 
      supabase.from('requisitions_drugcupsabot').select(`...`).eq('id', id).single()
    )
  },
  
  async updateStatus(id: number, status: RequisitionStatus) {
    return executeMutation(
      () => supabase.from('requisitions_drugcupsabot').update({ status }).eq('id', id),
      'อัปเดตสถานะสำเร็จ',
      { showToast: true }
    )
  },
}
```

---

## Phase 4: Router Integration

### Step 4.1: Add Error Handling to Router

Update `src/router/index.ts`:

```typescript
import { createRouter, createWebHistory } from 'vue-router'
import { useToast } from '@/composables/useToast'
import { AuthError, NotFoundError } from '@/utils/errorHandling'

const router = createRouter({
  history: createWebHistory(),
  routes,
})

// Error handling navigation guards
router.onError((error, to) => {
  console.error('Router Error:', error, to)
  
  const { error: showError } = useToast()
  
  if (error instanceof AuthError) {
    showError('เซสชันหมดอายุ กรุณาเข้าสู่ระบบใหม่', { title: 'หมดอายุ' })
    router.push('/login')
    return
  }
  
  if (error instanceof NotFoundError) {
    showError('หน้าที่คุณกำลังค้นหาไม่พบ', { title: 'ไม่พบ' })
    router.push('/')
    return
  }
  
  showError('ไม่สามารถโหลดหน้านี้ได้', { title: 'ข้อผิดพลาด' })
})

// Route-level error boundary simulation
router.beforeEach(async (to, from, next) => {
  try {
    // Your existing auth logic...
    next()
  } catch (err) {
    next(false) // Cancel navigation
    throw err // Let error boundary catch
  }
})
```

---

## Phase 5: Testing Error Boundaries

### Step 5.1: Create Error Test Component

Create `src/components/ErrorTestButton.vue` (dev only):

```vue
<!-- src/components/ErrorTestButton.vue -->
<template>
  <button 
    v-if="isDev" 
    @click="triggerError" 
    class="error-test-btn"
    title="Test Error Boundary (Dev Only)"
  >
    <i class="fas fa-bug"></i>
    Test Error
  </button>
</template>

<script setup lang="ts">
const isDev = import.meta.env.DEV

const triggerError = () => {
  // Throw different error types
  const errors = [
    () => { throw new Error('Test: Generic Error') },
    () => { throw new TypeError('Test: Type Error') },
    () => { throw new RangeError('Test: Range Error') },
    () => { throw new Error('Test: Async Error') },
    () => { 
      setTimeout(() => { throw new Error('Test: Async Timeout Error') }, 100) 
    },
  ]
  
  const randomError = errors[Math.floor(Math.random() * errors.length)]
  randomError()
}
</script>

<style scoped>
.error-test-btn {
  position: fixed;
  bottom: 20px;
  right: 20px;
  z-index: 9998;
  background: #6b7280;
  color: white;
  padding: 0.5rem 1rem;
  border-radius: 0.5rem;
  font-size: 0.8rem;
  opacity: 0.7;
  transition: opacity 0.2s;
}

.error-test-btn:hover {
  opacity: 1;
}
</style>
```

Add to `App.vue` for development testing.

---

## 🚨 Critical Implementation Rules

### DO:
- ✅ Always use ErrorBoundary at App level
- ✅ Wrap each major view with ViewErrorBoundary
- ✅ Use specific error classes (NetworkError, AuthError, etc.)
- ✅ Provide recovery mechanisms where possible
- ✅ Log errors for monitoring
- ✅ Test error scenarios in development
- ✅ Handle async errors with AsyncErrorBoundary

### DON'T:
- ❌ Swallow errors silently
- ❌ Show technical details to users in production
- ❌ Allow infinite error loops
- ❌ Forget to clean up resources on error
- ❌ Use generic Error class for everything

---

## 📋 Implementation Checklist

```
Global Setup:
[ ] Create ErrorBoundary.vue component
[ ] Create AsyncErrorBoundary.vue component
[ ] Create globalErrorHandler.ts plugin
[ ] Create errorHandling.ts utilities
[ ] Install plugin in main.ts

Integration:
[ ] Wrap App.vue with ErrorBoundary
[ ] Add ViewErrorBoundary to each view
[ ] Update services to use error handling
[ ] Add router error handling
[ ] Test all error scenarios

Monitoring:
[ ] Add error logging endpoint
[ ] Configure external error service (Sentry)
[ ] Set up error alerting
[ ] Document error codes
```

---

## 🎯 Success Criteria

Error Boundary implementation is complete when:

1. All unexpected errors show user-friendly UI
2. No white screens or infinite loading states
3. Users can recover from errors without refresh
4. Errors are logged with sufficient context
5. Technical details hidden from users in production
6. All async operations have error handling
7. Tested with ErrorTestButton component

---

## 🆘 Troubleshooting

### Issue: "ErrorBoundary not catching error"
**Solution**: Ensure error is thrown during render, not in async callback. Use `onErrorCaptured` for async.

### Issue: "Infinite error loop"
**Solution**: Check recovery logic doesn't re-trigger same error. Add recovery attempt counter.

### Issue: "Error in ErrorBoundary itself"
**Solution**: Add fallback UI directly in template, outside ErrorBoundary component.

---

**Remember**: Error boundaries are your last line of defense. They should never be needed, but when they are, they save the user experience.