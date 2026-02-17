 # Toast Notification Implementation Guide

This document provides a comprehensive, step-by-step guide for replacing all `alert()` calls with a professional toast notification system using `vue-toastification` without breaking existing functionality. Follow these instructions carefully to maintain production stability.

---

## 🎯 Implementation Strategy: "Silent Replacement"

We will replace alerts incrementally, ensuring the application remains functional at every step. The toast system will enhance UX while maintaining all existing logic flows.

---

## Phase 0: Foundation Setup (Do First)

### Step 0.1: Install vue-toastification

```bash
# Core library with Vue 3 support
npm install vue-toastification

# Note: This is the Vue 3 compatible version (next branch)
# For stable Vue 3, use: npm install vue-toastification@next
```

### Step 0.2: Create Toast Configuration

Create `src/plugins/toast.ts`:

```typescript
/**
 * Vue Toastification Configuration
 * Centralized toast setup with custom styling matching design system
 */

import type { PluginOptions, ToastOptionsAndRequiredContent } from 'vue-toastification'
import Toast, { POSITION, TYPE } from 'vue-toastification'
import type { App } from 'vue'

// Import styles
import 'vue-toastification/dist/index.css'

// ==========================================
// Custom Toast Styles (CSS Variables Integration)
// ==========================================

const customToastStyles = `
/* Base toast container */
.Vue-Toastification__container {
  z-index: 9999;
  font-family: 'Poppins', 'Noto Sans Thai', sans-serif;
}

/* Toast notification base */
.Vue-Toastification__toast {
  border-radius: 0.5rem;
  box-shadow: 0 4px 12px rgba(0, 90, 156, 0.15);
  padding: 1rem 1.25rem;
  min-height: auto;
}

/* Success toast - matches --success-color */
.Vue-Toastification__toast--success {
  background-color: #F7CFD8;
  color: #1B4332;
  border-left: 4px solid #2E7D32;
}

.Vue-Toastification__toast--success .Vue-Toastification__icon {
  color: #2E7D32;
}

/* Error toast - matches --danger-color */
.Vue-Toastification__toast--error {
  background-color: #FED7D7;
  color: #742A2A;
  border-left: 4px solid #C53030;
}

.Vue-Toastification__toast--error .Vue-Toastification__icon {
  color: #C53030;
}

/* Warning toast - matches --warning-color */
.Vue-Toastification__toast--warning {
  background-color: #FEEBC8;
  color: #7C2D12;
  border-left: 4px solid #B45309;
}

.Vue-Toastification__toast--warning .Vue-Toastification__icon {
  color: #B45309;
}

/* Info toast - matches --info-color */
.Vue-Toastification__toast--info {
  background-color: #DBEAFE;
  color: #1E3A8A;
  border-left: 4px solid #2563EB;
}

.Vue-Toastification__toast--info .Vue-Toastification__icon {
  color: #2563EB;
}

/* Close button */
.Vue-Toastification__close-button {
  color: currentColor;
  opacity: 0.6;
  transition: opacity 0.2s;
}

.Vue-Toastification__close-button:hover {
  opacity: 1;
}

/* Progress bar */
.Vue-Toastification__progress-bar {
  height: 3px;
  opacity: 0.3;
}

/* Toast content */
.Vue-Toastification__toast-body {
  font-size: 0.95rem;
  line-height: 1.5;
  font-weight: 500;
}

/* Icon sizing */
.Vue-Toastification__icon {
  font-size: 1.25rem;
  margin-right: 0.75rem;
}
`

// Inject custom styles
const injectStyles = () => {
  const styleId = 'toast-custom-styles'
  if (document.getElementById(styleId)) return
  
  const style = document.createElement('style')
  style.id = styleId
  style.innerHTML = customToastStyles
  document.head.appendChild(style)
}

// ==========================================
// Toast Options Configuration
// ==========================================

export const toastOptions: PluginOptions = {
  position: POSITION.TOP_RIGHT,
  timeout: 5000,
  closeOnClick: true,
  pauseOnFocusLoss: true,
  pauseOnHover: true,
  draggable: true,
  draggablePercent: 0.6,
  showCloseButtonOnHover: false,
  hideProgressBar: false,
  closeButton: 'button',
  icon: true,
  rtl: false,
  transition: 'Vue-Toastification__bounce',
  maxToasts: 5,
  newestOnTop: true,
  filterBeforeCreate: (toast, toasts) => {
    // Prevent duplicate toasts
    if (toasts.length > 0) {
      const existing = toasts.find((t) => t.content === toast.content && t.type === toast.type)
      if (existing) return false
    }
    return toast
  },
}

// ==========================================
// Toast Types & Icons Mapping
// ==========================================

export const toastIcons = {
  success: 'fa-check-circle',
  error: 'fa-exclamation-circle',
  warning: 'fa-exclamation-triangle',
  info: 'fa-info-circle',
}

// ==========================================
// Plugin Installation
// ==========================================

export const installToast = (app: App) => {
  injectStyles()
  app.use(Toast, toastOptions)
}

export default Toast
```

### Step 0.3: Create Composable for Toast Operations

Create `src/composables/useToast.ts`:

```typescript
/**
 * Toast notification composable
 * Provides type-safe, consistent toast notifications across the app
 */

import { useToast as useVueToastification, type ToastOptions } from 'vue-toastification'
import type { ToastContent, ToastID } from 'vue-toastification/dist/types/types'

// Toast types matching our design system
export type ToastType = 'success' | 'error' | 'warning' | 'info'

// Toast options with our defaults
interface CustomToastOptions extends ToastOptions {
  title?: string
  persistent?: boolean // If true, no auto-dismiss
}

export function useToast() {
  const toast = useVueToastification()

  // ==========================================
  // Core Toast Methods
  // ==========================================

  /**
   * Show success toast
   */
  const success = (message: string, options: CustomToastOptions = {}): ToastID => {
    return toast.success(formatMessage(message, options.title), {
      ...getDefaultOptions(options),
      timeout: options.persistent ? false : 5000,
    })
  }

  /**
   * Show error toast
   */
  const error = (message: string, options: CustomToastOptions = {}): ToastID => {
    return toast.error(formatMessage(message, options.title), {
      ...getDefaultOptions(options),
      timeout: options.persistent ? false : 8000, // Errors stay longer
    })
  }

  /**
   * Show warning toast
   */
  const warning = (message: string, options: CustomToastOptions = {}): ToastID => {
    return toast.warning(formatMessage(message, options.title), {
      ...getDefaultOptions(options),
      timeout: options.persistent ? false : 6000,
    })
  }

  /**
   * Show info toast
   */
  const info = (message: string, options: CustomToastOptions = {}): ToastID => {
    return toast.info(formatMessage(message, options.title), {
      ...getDefaultOptions(options),
      timeout: options.persistent ? false : 4000,
    })
  }

  // ==========================================
  // Specialized Toast Methods
  // ==========================================

  /**
   * Show loading toast (persistent, manual dismiss required)
   */
  const loading = (message: string = 'กำลังดำเนินการ...'): ToastID => {
    return toast.info(message, {
      timeout: false,
      closeButton: false,
      icon: {
        name: 'fa-spinner fa-spin',
      },
    })
  }

  /**
   * Dismiss a specific toast
   */
  const dismiss = (toastId: ToastID): void => {
    toast.dismiss(toastId)
  }

  /**
   * Dismiss all toasts
   */
  const clearAll = (): void => {
    toast.clear()
  }

  /**
   * Update existing toast (e.g., loading -> success)
   */
  const update = (
    toastId: ToastID,
    type: ToastType,
    message: string,
    options: CustomToastOptions = {}
  ): void => {
    toast.update(toastId, {
      content: formatMessage(message, options.title),
      options: {
        type: toast[type], // Convert our type to vue-toastification type
        timeout: options.persistent ? false : type === 'error' ? 8000 : 5000,
        closeButton: true,
        icon: true,
      },
    })
  }

  /**
   * Promise-based toast (auto handles loading/success/error)
   */
  const promise = async <T>(
    promiseFn: () => Promise<T>,
    messages: {
      loading: string
      success: string | ((data: T) => string)
      error: string | ((error: Error) => string)
    },
    options: { successTitle?: string; errorTitle?: string } = {}
  ): Promise<T> => {
    const loadingToast = loading(messages.loading)

    try {
      const result = await promiseFn()
      const successMessage =
        typeof messages.success === 'function'
          ? messages.success(result)
          : messages.success

      update(loadingToast, 'success', successMessage, { title: options.successTitle })
      return result
    } catch (err) {
      const errorMessage =
        typeof messages.error === 'function'
          ? messages.error(err instanceof Error ? err : new Error(String(err)))
          : messages.error

      update(loadingToast, 'error', errorMessage, { title: options.errorTitle })
      throw err
    }
  }

  // ==========================================
  // Helper Functions
  // ==========================================

  const formatMessage = (message: string, title?: string): string => {
    if (!title) return message
    return `<div class="toast-title">${title}</div><div class="toast-message">${message}</div>`
  }

  const getDefaultOptions = (options: CustomToastOptions): ToastOptions => {
    return {
      position: 'top-right',
      ...options,
    }
  }

  return {
    // Core methods
    success,
    error,
    warning,
    info,
    
    // Advanced methods
    loading,
    dismiss,
    clearAll,
    update,
    promise,
    
    // Raw toast instance for advanced use
    raw: toast,
  }
}
```

### Step 0.4: Add Toast Styles to Global CSS

Add to `src/assets/main.css`:

```css
/* ==========================================
   Toast Notification Enhancements
   ========================================== */

.toast-title {
  font-weight: 700;
  font-size: 1rem;
  margin-bottom: 0.25rem;
  display: block;
}

.toast-message {
  font-weight: 400;
  font-size: 0.9rem;
  opacity: 0.9;
}

/* Mobile optimizations */
@media (max-width: 600px) {
  .Vue-Toastification__container {
    width: 100%;
    padding: 0 1rem;
  }
  
  .Vue-Toastification__toast {
    border-radius: 0.5rem;
    margin-bottom: 0.5rem;
  }
}

/* Reduced motion preference */
@media (prefers-reduced-motion: reduce) {
  .Vue-Toastification__bounce-enter-active,
  .Vue-Toastification__bounce-leave-active {
    transition: none;
  }
}
```

### Step 0.5: Register Plugin in Main Entry

Update `src/main.js` → `src/main.ts` (or keep .js):

```typescript
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'
import router from './router'
import { installToast } from './plugins/toast'
import './assets/main.css'

const app = createApp(App)

app.use(createPinia())
app.use(router)
installToast(app) // Add toast plugin

app.mount('#app')
```

---

## Phase 1: Alert Audit & Mapping

### Step 1.1: Create Alert Inventory

Create `src/migration/alerts-map.ts` (temporary tracking file):

```typescript
/**
 * Alert Migration Inventory
 * Track all alert() calls that need replacement
 * 
 * Status: 
 * - pending: Not yet migrated
 * - migrated: Replaced with toast
 * - verified: Tested and working
 */

export const alertInventory = [
  // Auth
  { file: 'views/Login.vue', line: 45, type: 'error', message: 'เข้าสู่ระบบไม่สำเร็จ', status: 'pending' },
  { file: 'views/Register.vue', line: 89, type: 'success', message: 'ลงทะเบียนสำเร็จ', status: 'pending' },
  { file: 'views/Register.vue', line: 95, type: 'error', message: 'อีเมลนี้มีการลงทะเบียนแล้ว', status: 'pending' },
  
  // Admin
  { file: 'views/admin/AdminRequisitionDetail.vue', line: 156, type: 'success', message: 'บันทึกและอนุมัติใบเบิก', status: 'pending' },
  { file: 'views/admin/AdminRequisitionDetail.vue', line: 166, type: 'error', message: 'เกิดข้อผิดพลาด', status: 'pending' },
  { file: 'views/admin/AdminRequisitionDetail.vue', line: 189, type: 'success', message: 'ยืนยันการจ่ายเวชภัณฑ์', status: 'pending' },
  { file: 'views/admin/ItemManagement.vue', line: 134, type: 'success', message: 'อัปเดตรายการ', status: 'pending' },
  { file: 'views/admin/ItemManagement.vue', line: 178, type: 'success', message: 'เพิ่มรายการ', status: 'pending' },
  { file: 'views/admin/UserManagement.vue', line: 78, type: 'success', message: 'อัปเดตสถานะผู้ใช้', status: 'pending' },
  { file: 'views/admin/SystemSettings.vue', line: 95, type: 'success', message: 'บันทึกข้อมูล', status: 'pending' },
  
  // PCU
  { file: 'views/pcu/RequisitionForm.vue', line: 245, type: 'success', message: 'บันทึกใบเบิก', status: 'pending' },
  { file: 'views/pcu/RequisitionForm.vue', line: 252, type: 'error', message: 'เกิดข้อผิดพลาดในการบันทึก', status: 'pending' },
] as const

// Migration priority: High traffic -> Low traffic
export const migrationPriority = [
  'views/Login.vue',
  'views/Register.vue',
  'views/pcu/RequisitionForm.vue',
  'views/admin/AdminRequisitionDetail.vue',
  'views/admin/ItemManagement.vue',
  'views/admin/UserManagement.vue',
  'views/admin/SystemSettings.vue',
]
```

---

## Phase 2: Component-by-Component Migration

### Step 2.1: Login.vue Migration

**Before:**
```javascript
// Old code
alert("เข้าสู่ระบบไม่สำเร็จ: " + error.message)
```

**After:**

```vue
<!-- src/views/Login.vue -->
<template>
  <!-- Template unchanged -->
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { useRouter } from 'vue-router'
import { useAuth } from '@/composables/useAuth'
import { useToast } from '@/composables/useToast'
import { useFormValidation } from '@/composables/useValidation'
import { loginSchema } from '@/schemas'
import type { LoginInput } from '@/schemas'

const router = useRouter()
const { login } = useAuth()
const { success, error: showError } = useToast() // Destructure toast methods

const initialData: LoginInput = { email: '', password: '' }

const {
  formData,
  updateField,
  touch,
  getVisibleError,
  validateForm,
  handleSubmit,
} = useFormValidation(loginSchema, initialData)

const isSubmitting = ref(false)

const onSubmit = async () => {
  const result = await handleSubmit(async (data) => {
    await login(data.email, data.password)
    
    // SUCCESS: Replace alert with toast
    success('เข้าสู่ระบบสำเร็จ', {
      title: 'ยินดีต้อนรับ',
      timeout: 3000,
    })
    
    router.push('/')
  })

  if (!result.success) {
    // ERROR: Replace alert with toast
    showError(result.errors[0] || 'เข้าสู่ระบบไม่สำเร็จ', {
      title: 'เกิดข้อผิดพลาด',
    })
  }
}
</script>
```

### Step 2.2: Register.vue Migration

```vue
<!-- src/views/Register.vue -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { supabase } from '@/supabaseClient'
import { useToast } from '@/composables/useToast'
import { useFormValidation } from '@/composables/useValidation'
import { registerSchema } from '@/schemas'
import type { RegisterInput, PCU } from '@/types'

const router = useRouter()
const { success, error: showError, promise } = useToast() // Add promise helper

// ... setup code ...

const onSubmit = async () => {
  // Use promise-based toast for async operations
  try {
    await promise(
      async () => {
        const APP_BASE_URL = import.meta.env.VITE_APP_BASE_URL || window.location.origin
        const EMAIL_REDIRECT_URL = `${APP_BASE_URL}/waiting-for-approval`

        const { data, error } = await supabase.auth.signUp({
          email: formData.email,
          password: formData.password,
          options: {
            data: {
              username: formData.username.trim(),
              pcu_id: formData.pcu_id,
              email: formData.email.trim(),
            },
            emailRedirectTo: EMAIL_REDIRECT_URL,
          },
        })

        if (error) {
          if (error.message.includes('User already registered')) {
            throw new Error('อีเมลนี้มีการลงทะเบียนแล้ว')
          }
          throw error
        }

        return data
      },
      {
        loading: 'กำลังลงทะเบียน...',
        success: 'การลงทะเบียนสำเร็จ! กรุณารอการอนุมัติจากผู้ดูแลระบบ',
        error: (err) => err.message || 'ลงทะเบียนไม่สำเร็จ',
      },
      {
        successTitle: 'สำเร็จ',
        errorTitle: 'ลงทะเบียนไม่สำเร็จ',
      }
    )

    router.push('/login')
  } catch {
    // Error already shown by promise helper
  }
}
</script>
```

### Step 2.3: AdminRequisitionDetail.vue Migration

```vue
<!-- src/views/admin/AdminRequisitionDetail.vue -->
<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import { useToast } from '@/composables/useToast'
import { requisitionService, rpcService } from '@/services/supabaseService'
import type { Requisition, RequisitionItem } from '@/types'

const props = defineProps<{ requisitionId: string }>()

const { success, error: showError, warning, loading, update, dismiss } = useToast()

// ... existing setup ...

const saveAndApprove = async () => {
  const confirmed = confirm(
    'ยืนยันการบันทึกและอนุมัติใบเบิกนี้? การกระทำนี้จะอัปเดตจำนวนที่อนุมัติทั้งหมด'
  )
  if (!confirmed) return

  const loadingToast = loading('กำลังบันทึกและอนุมัติ...')

  try {
    const itemsToUpdate = editableItems.value.map((item) => ({
      id: item.id,
      approved_quantity: item.approved_quantity || 0,
    }))

    const { error: rpcError } = await rpcService.updateApprovedQuantities(itemsToUpdate)
    if (rpcError) throw rpcError

    const { error: reqError } = await requisitionService.updateStatus(
      parseInt(props.requisitionId),
      'approved'
    )
    if (reqError) throw reqError

    // Update loading toast to success
    update(loadingToast, 'success', 'บันทึกและอนุมัติใบเบิกเรียบร้อยแล้ว', {
      title: 'สำเร็จ',
    })

    await fetchRequisition()
  } catch (err) {
    update(
      loadingToast,
      'error',
      err instanceof Error ? err.message : 'เกิดข้อผิดพลาดในการบันทึก',
      { title: 'ผิดพลาด' }
    )
  }
}

const fulfillRequisition = async () => {
  const confirmed = confirm('ยืนยันการจ่ายเวชภัณฑ์ตามจำนวนที่อนุมัติแล้วใช่หรือไม่?')
  if (!confirmed) return

  const loadingToast = loading('กำลังยืนยันการจ่าย...')

  try {
    const { error } = await requisitionService.updateStatus(
      parseInt(props.requisitionId),
      'fulfilled'
    )
    if (error) throw error

    update(loadingToast, 'success', 'ยืนยันการจ่ายเวชภัณฑ์เรียบร้อยแล้ว', {
      title: 'สำเร็จ',
    })

    await fetchRequisition()
  } catch (err) {
    update(
      loadingToast,
      'error',
      err instanceof Error ? err.message : 'เกิดข้อผิดพลาด',
      { title: 'ผิดพลาด' }
    )
  }
}
</script>
```

### Step 2.4: ItemManagement.vue Migration

```vue
<!-- src/views/admin/ItemManagement.vue -->
<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import { useToast } from '@/composables/useToast'
import { itemService } from '@/services/supabaseService'
import { itemSchema, safeParse } from '@/schemas'
import type { Item } from '@/types'

const { success, error: showError, warning } = useToast()

// ... setup ...

const updateItem = async (item: Partial<Item> & { id: number }) => {
  // Validate before update
  const partialSchema = itemSchema.pick({ 
    name: true, 
    price_per_unit: true, 
    unit_pack: true,
    notes: true 
  })
  
  const validation = safeParse(partialSchema, {
    name: item.name,
    price_per_unit: item.price_per_unit,
    unit_pack: item.unit_pack,
    notes: item.notes,
  })

  if (!validation.success) {
    showError(validation.errors[0], { title: 'ข้อมูลไม่ถูกต้อง' })
    return
  }

  try {
    const { error } = await itemService.update(item)
    if (error) throw error
    
    success(`อัปเดตรายการ "${item.name}" เรียบร้อยแล้ว`, {
      title: 'บันทึกสำเร็จ',
      timeout: 3000,
    })
  } catch (err) {
    showError(
      err instanceof Error ? err.message : 'เกิดข้อผิดพลาดในการอัปเดต',
      { title: 'บันทึกไม่สำเร็จ' }
    )
    // Revert changes or refetch
    await fetchItems()
  }
}

const handleAddItem = async () => {
  // Validation...
  
  try {
    const { data, error } = await itemService.create(newItem.value)
    if (error) throw error

    success(`เพิ่มรายการ "${data?.[0]?.name}" สำเร็จ`, {
      title: 'สำเร็จ',
    })
    
    showAddItemModal.value = false
    await fetchItems()
  } catch (err) {
    showError(
      err instanceof Error ? err.message : 'เกิดข้อผิดพลาดในการเพิ่มรายการ',
      { title: 'เพิ่มรายการไม่สำเร็จ' }
    )
  }
}
</script>
```

### Step 2.5: UserManagement.vue Migration

```vue
<!-- src/views/admin/UserManagement.vue -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useToast } from '@/composables/useToast'
import { profileService } from '@/services/supabaseService'
import type { UserProfile } from '@/types'

const { success, error: showError, warning } = useToast()

// ... setup ...

const updateUserStatus = async (userId: string, newStatus: 'approved' | 'rejected') => {
  const actionText = newStatus === 'approved' ? 'อนุมัติ' : 'ปฏิเสธ'
  
  // Use warning toast for confirmation instead of native confirm
  // Or keep native confirm for critical actions, your choice
  
  if (!confirm(`คุณต้องการ${actionText}ผู้ใช้งานนี้ใช่หรือไม่?`)) return

  try {
    const { error } = await profileService.updateStatus(userId, newStatus)
    if (error) throw error

    success(`อัปเดตสถานะผู้ใช้เป็น"${actionText}"เรียบร้อยแล้ว`, {
      title: 'สำเร็จ',
    })

    await fetchPendingUsers()
  } catch (err) {
    showError(
      err instanceof Error ? err.message : 'เกิดข้อผิดพลาดในการอัปเดตสถานะ',
      { title: 'ผิดพลาด' }
    )
  }
}
</script>
```

### Step 2.6: SystemSettings.vue Migration

```vue
<!-- src/views/admin/SystemSettings.vue -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useToast } from '@/composables/useToast'
import { pcuService } from '@/services/supabaseService'
import { pcuPersonnelSchema, safeParse } from '@/schemas'
import type { PCU, PCUPersonnel } from '@/types'

const { success, error: showError } = useToast()

// ... setup ...

const savePcuPersonnel = async (pcu: PCUPersonnel) => {
  // Validate
  const validation = safeParse(pcuPersonnelSchema, pcu)
  if (!validation.success) {
    showError(validation.errors[0], { title: 'ข้อมูลไม่ถูกต้อง' })
    return
  }

  try {
    const { error } = await pcuService.savePersonnel(pcu)
    if (error) throw error

    success(`บันทึกข้อมูลของ ${pcu.pcu_name} เรียบร้อยแล้ว`, {
      title: 'บันทึกสำเร็จ',
    })
    
    await fetchSettings()
  } catch (err) {
    showError(
      err instanceof Error ? err.message : 'เกิดข้อผิดพลาดในการบันทึก',
      { title: 'บันทึกไม่สำเร็จ' }
    )
  }
}
</script>
```

### Step 2.7: RequisitionForm.vue Migration (PCU)

```vue
<!-- src/views/pcu/RequisitionForm.vue -->
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { useAuth } from '@/composables/useAuth'
import { useToast } from '@/composables/useToast'
import { useRequisitionForm } from '@/composables/useRequisitionForm'
import { requisitionService, requisitionItemService, itemService } from '@/services/supabaseService'
import { formatCurrency } from '@/utils/formatters'
import type { Item, RequisitionPeriod } from '@/types'

const { success, error: showError, warning } = useToast()

// ... setup ...

const saveRequisition = async (status: 'draft' | 'submitted') => {
  if (!validateForm()) {
    showError('กรุณาตรวจสอบข้อมูลให้ถูกต้อง', { title: 'ข้อมูลไม่ครบถ้วน' })
    return false
  }

  if (status === 'submitted') {
    // Use warning toast for non-blocking confirmation
    // Or keep confirm() for blocking action
    const confirmed = confirm('คุณต้องการยืนยันการส่งใบเบิกนี้ใช่หรือไม่? เมื่อส่งแล้วจะไม่สามารถแก้ไขได้อีก')
    if (!confirmed) return false
  }

  isSubmitting.value = true

  try {
    // ... save logic ...

    const message = status === 'submitted' 
      ? 'ส่งใบเบิกเรียบร้อยแล้ว' 
      : 'บันทึกฉบับร่างเรียบร้อยแล้ว'
    
    success(message, {
      title: 'สำเร็จ',
      timeout: status === 'submitted' ? 5000 : 3000,
    })

    router.push('/pcu/dashboard')
    return true
  } catch (err) {
    showError(
      err instanceof Error ? err.message : 'เกิดข้อผิดพลาดในการบันทึก',
      { 
        title: 'บันทึกไม่สำเร็จ',
        persistent: true, // Keep error visible
      }
    )
    return false
  } finally {
    isSubmitting.value = false
  }
}
</script>
```

---

## Phase 3: Global Error Handling

### Step 3.1: Create Global Error Handler

Create `src/plugins/errorHandler.ts`:

```typescript
/**
 * Global error handling with toast notifications
 */

import type { App } from 'vue'
import { useToast } from '@/composables/useToast'

export const installErrorHandler = (app: App) => {
  // Vue error handler
  app.config.errorHandler = (err, instance, info) => {
    console.error('Vue Error:', err, info)
    
    // Use setTimeout to avoid "setup() called without active component" error
    setTimeout(() => {
      const { error } = useToast()
      error('เกิดข้อผิดพลาดที่ไม่คาดคิด กรุณารีเฟรชหน้าเว็บ', {
        title: 'ข้อผิดพลาดระบบ',
        persistent: true,
      })
    }, 0)
  }

  // Global unhandled promise rejection
  window.addEventListener('unhandledrejection', (event) => {
    console.error('Unhandled Promise Rejection:', event.reason)
    
    setTimeout(() => {
      const { error } = useToast()
      error('การเชื่อมต่อล้มเหลว กรุณาลองใหม่อีกครั้ง', {
        title: 'ข้อผิดพลาดเครือข่าย',
      })
    }, 0)
  })

  // Network error handling (offline detection)
  window.addEventListener('offline', () => {
    const { warning } = useToast()
    warning('คุณออฟไลน์อยู่ กรุณาเชื่อมต่ออินเทอร์เน็ต', {
      title: 'ไม่มีการเชื่อมต่อ',
      persistent: true,
    })
  })

  window.addEventListener('online', () => {
    const { success } = useToast()
    success('เชื่อมต่ออินเทอร์เน็ตสำเร็จ', {
      title: 'ออนไลน์แล้ว',
      timeout: 3000,
    })
  })
}
```

Update `src/main.ts`:

```typescript
import { installErrorHandler } from './plugins/errorHandler'

// ... other plugins ...

installErrorHandler(app)
```

---

## Phase 4: Special Cases

### Step 4.1: Print View Error Handling

For print views that can't show toasts (no UI), keep error display in DOM:

```vue
<!-- src/views/admin/PrintableView.vue -->
<template>
  <div v-if="error" class="print-error">
    <h1>เกิดข้อผิดพลาด</h1>
    <p>{{ error }}</p>
    <button @click="retry">ลองใหม่</button>
  </div>
  <!-- ... normal template ... -->
</template>

<script setup>
// Don't use toast here - print views have no container
// Keep native error display or use console
</script>

<style>
.print-error {
  padding: 2rem;
  text-align: center;
  color: var(--danger-color);
}
@media print {
  .print-error {
    display: block !important;
  }
}
</style>
```

### Step 4.2: Confirm Dialog Replacement (Optional)

If you want to replace `confirm()` with a custom modal:

Create `src/components/ConfirmModal.vue`:

```vue
<template>
  <Teleport to="body">
    <div v-if="isOpen" class="modal-overlay" @click="onCancel">
      <div class="modal-content" @click.stop>
        <div class="modal-header">
          <i :class="['fas', iconClass]"></i>
          <h3>{{ title }}</h3>
        </div>
        <p class="modal-message">{{ message }}</p>
        <div class="modal-actions">
          <button @click="onCancel" class="btn btn-secondary">
            {{ cancelText }}
          </button>
          <button @click="onConfirm" class="btn" :class="confirmButtonClass">
            {{ confirmText }}
          </button>
        </div>
      </div>
    </div>
  </Teleport>
</template>

<script setup>
import { computed } from 'vue'

const props = defineProps({
  isOpen: Boolean,
  title: { type: String, default: 'ยืนยันการดำเนินการ' },
  message: { type: String, required: true },
  type: { type: String, default: 'warning' }, // warning, danger, info
  confirmText: { type: String, default: 'ยืนยัน' },
  cancelText: { type: String, default: 'ยกเลิก' },
})

const emit = defineEmits(['confirm', 'cancel'])

const iconClass = computed(() => ({
  warning: 'fa-exclamation-triangle',
  danger: 'fa-exclamation-circle',
  info: 'fa-info-circle',
}[props.type]))

const confirmButtonClass = computed(() => ({
  warning: 'btn-warning',
  danger: 'btn-danger',
  info: 'btn-primary',
}[props.type]))

const onConfirm = () => emit('confirm')
const onCancel = () => emit('cancel')
</script>
```

Then use with composable:

```typescript
// src/composables/useConfirm.ts
import { ref } from 'vue'

const isOpen = ref(false)
const resolveRef = ref<(value: boolean) => void>()

export const useConfirm = () => {
  const confirm = (options: {
    title?: string
    message: string
    type?: 'warning' | 'danger' | 'info'
  }): Promise<boolean> => {
    // Set options and open modal
    isOpen.value = true
    return new Promise((resolve) => {
      resolveRef.value = resolve
    })
  }

  const onConfirm = () => {
    isOpen.value = false
    resolveRef.value?.(true)
  }

  const onCancel = () => {
    isOpen.value = false
    resolveRef.value?.(false)
  }

  return {
    isOpen,
    confirm,
    onConfirm,
    onCancel,
  }
}
```

---

## 🚨 Critical Implementation Rules

### DO:
- ✅ Always import `useToast` at component level, not global
- ✅ Use `setTimeout 0` when calling toast outside Vue component context
- ✅ Provide both Thai and English messages (i18n ready)
- ✅ Use appropriate toast types (success/error/warning/info)
- ✅ Set appropriate timeouts (errors longer, success shorter)
- ✅ Dismiss loading toasts on success/error
- ✅ Test toast visibility on mobile devices

### DON'T:
- ❌ Use `alert()` anywhere after migration
- ❌ Show toast before router navigation (may not be seen)
- ❌ Use toast for validation errors (use inline instead)
- ❌ Create multiple toast instances
- ❌ Forget to handle errors in catch blocks

---

## 📋 Migration Checklist Per Component

```
For each component:
[ ] Import useToast composable
[ ] Replace alert('success') with toast.success()
[ ] Replace alert('error') with toast.error()
[ ] Replace alert('confirm') with confirm() or custom modal
[ ] Test success path
[ ] Test error path
[ ] Verify toast appears correctly
[ ] Verify toast styling matches design
[ ] Mark as migrated in inventory
```

---

## 🎯 Success Criteria

Toast implementation is complete when:

1. Zero `alert()` calls remain in codebase (grep -r "alert(" src/)
2. All success actions show confirmation toast
3. All errors show error toast with clear message
4. Loading states use persistent loading toast
5. Mobile toast visibility tested
6. No console errors from toast plugin
7. Accessibility: toasts announced to screen readers

---

## 🆘 Troubleshooting

### Issue: "Toast not showing"
**Solution**: Ensure `installToast(app)` is called before app.mount(), and component is inside Vue app.

### Issue: "Toast style not applied"
**Solution**: Check that custom CSS is injected and not overridden by scoped styles.

### Issue: "Multiple toasts stacking"
**Solution**: Use `filterBeforeCreate` in options to prevent duplicates, or call `clearAll()` before important toasts.

### Issue: "Toast behind modal"
**Solution**: Check z-index. Toast container should be 9999, modals lower.

---

**Remember**: Toast notifications should enhance UX, not annoy users. Keep messages concise, actionable, and culturally appropriate for Thai healthcare workers.