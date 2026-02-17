# Zod Input Validation Implementation Guide

This document provides a comprehensive, step-by-step guide for implementing Zod input validation across the Drug Requisition System without breaking existing functionality. Follow these instructions carefully to maintain production stability.

---

## 🎯 Implementation Strategy: "Progressive Validation"

We will implement validation incrementally, form by form, ensuring the application remains functional at every step. Never implement multiple critical forms simultaneously.

---

## Phase 0: Foundation Setup (Do First)

### Step 0.1: Install Zod Dependencies

```bash
# Core Zod library
npm install zod

# Optional: Zod-form-adapter for form libraries (if needed later)
# npm install @hookform/resolvers zod react-hook-form  # For React, Vue equivalent below

# Vue integration (if using VeeValidate later)
# npm install @vee-validate/zod vee-validate
```

### Step 0.2: Create src/schemas/index.ts (Core Schema Definitions)

This is the MOST IMPORTANT file. All validation schemas are centralized here.

```typescript
/**
 * Zod Validation Schemas for Drug Requisition System
 * All input validation rules are defined here for single source of truth
 */

import { z } from 'zod'

// ==========================================
// Custom Error Messages (Thai)
// ==========================================

const errorMessages = {
  required: (field: string) => `${field} จำเป็นต้องกรอก`,
  email: 'รูปแบบอีเมลไม่ถูกต้อง',
  minLength: (field: string, min: number) => `${field} ต้องมีอย่างน้อย ${min} ตัวอักษร`,
  maxLength: (field: string, max: number) => `${field} ต้องไม่เกิน ${max} ตัวอักษร`,
  positiveNumber: (field: string) => `${field} ต้องเป็นตัวเลขที่ไม่ติดลบ`,
  integer: (field: string) => `${field} ต้องเป็นจำนวนเต็ม`,
  minValue: (field: string, min: number) => `${field} ต้องไม่น้อยกว่า ${min}`,
  maxValue: (field: string, max: number) => `${field} ต้องไม่เกิน ${max}`,
  selectOption: (field: string) => `กรุณาเลือก${field}`,
}

// ==========================================
// Reusable Schema Parts
// ==========================================

/**
 * Email schema with strict validation
 */
export const emailSchema = z
  .string()
  .min(1, errorMessages.required('อีเมล'))
  .email(errorMessages.email)
  .max(255, errorMessages.maxLength('อีเมล', 255))
  .trim()
  .toLowerCase()

/**
 * Password schema with security requirements
 */
export const passwordSchema = z
  .string()
  .min(1, errorMessages.required('รหัสผ่าน'))
  .min(6, errorMessages.minLength('รหัสผ่าน', 6))
  .max(100, errorMessages.maxLength('รหัสผ่าน', 100))
  .regex(/[a-zA-Z]/, 'รหัสผ่านต้องมีตัวอักษรภาษาอังกฤษอย่างน้อย 1 ตัว')
  .regex(/[0-9]/, 'รหัสผ่านต้องมีตัวเลขอย่างน้อย 1 ตัว')

/**
 * Positive integer schema for quantities
 */
export const quantitySchema = (fieldName: string) =>
  z
    .union([z.string(), z.number(), z.null()])
    .transform((val) => (val === '' || val === null ? null : Number(val)))
    .refine((val) => val === null || (!isNaN(val) && val >= 0), {
      message: errorMessages.positiveNumber(fieldName),
    })
    .refine((val) => val === null || Number.isInteger(val), {
      message: errorMessages.integer(fieldName),
    })
    .refine((val) => val === null || val <= 999999, {
      message: errorMessages.maxValue(fieldName, 999999),
    })
    .nullable()
    .optional()

/**
 * Price schema with decimal support
 */
export const priceSchema = z
  .union([z.string(), z.number()])
  .transform((val) => (val === '' ? 0 : Number(val)))
  .refine((val) => !isNaN(val) && val >= 0, {
    message: errorMessages.positiveNumber('ราคา'),
  })
  .refine((val) => val <= 999999999.99, {
    message: errorMessages.maxValue('ราคา', 999999999.99),
  })
  .default(0)

/**
 * ID schema (positive integer)
 */
export const idSchema = z.number().int().positive()

/**
 * String with max length
 */
export const stringWithMax = (field: string, max: number) =>
  z
    .string()
    .max(max, errorMessages.maxLength(field, max))
    .trim()
    .optional()
    .nullable()

// ==========================================
// Authentication Schemas
// ==========================================

/**
 * Login form validation schema
 */
export const loginSchema = z.object({
  email: emailSchema,
  password: z
    .string()
    .min(1, errorMessages.required('รหัสผ่าน'))
    .min(6, errorMessages.minLength('รหัสผ่าน', 6)),
})

export type LoginInput = z.infer<typeof loginSchema>

/**
 * Registration form validation schema
 */
export const registerSchema = z
  .object({
    username: z
      .string()
      .min(1, errorMessages.required('ชื่อ-นามสกุล'))
      .min(2, errorMessages.minLength('ชื่อ-นามสกุล', 2))
      .max(100, errorMessages.maxLength('ชื่อ-นามสกุล', 100))
      .trim(),
    email: emailSchema,
    password: passwordSchema,
    confirmPassword: z.string().min(1, errorMessages.required('ยืนยันรหัสผ่าน')),
    pcu_id: z.union([
      z.number().int().positive(errorMessages.selectOption('รพ.สต.')),
      z.literal('').transform(() => {
        throw new Error(errorMessages.selectOption('รพ.สต.'))
      }),
    ]),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: 'รหัสผ่านไม่ตรงกัน',
    path: ['confirmPassword'],
  })

export type RegisterInput = z.infer<typeof registerSchema>

// ==========================================
// Requisition Schemas
// ==========================================

/**
 * Single requisition item validation
 */
export const requisitionItemSchema = z.object({
  item_id: idSchema,
  quantity: quantitySchema('จำนวนเบิก'),
  on_hand_quantity: quantitySchema('ยอดคงเหลือ'),
  price_at_request: priceSchema,
})

export type RequisitionItemInput = z.infer<typeof requisitionItemSchema>

/**
 * Complete requisition form validation
 */
export const requisitionFormSchema = z.object({
  period_id: idSchema,
  pcu_id: idSchema,
  items: z
    .array(
      z.object({
        item_id: idSchema,
        quantity: z.number().int().min(0).nullable(),
        on_hand_quantity: z.number().int().min(0).nullable(),
      })
    )
    .refine(
      (items) => items.some((item) => item.quantity && item.quantity > 0),
      {
        message: 'ต้องมีอย่างน้อย 1 รายการที่มีจำนวนเบิกมากกว่า 0',
      }
    ),
})

export type RequisitionFormInput = z.infer<typeof requisitionFormSchema>

/**
 * Requisition item from form data (dynamic keys)
 */
export const requisitionItemFormSchema = z.record(
  z.string().or(z.number()),
  z.object({
    quantity: z.union([z.number(), z.null()]).optional(),
    onHand: z.union([z.number(), z.null()]).optional(),
  })
)

// ==========================================
// Item Management Schemas
// ==========================================

/**
 * Item creation/update validation
 */
export const itemSchema = z.object({
  name: z
    .string()
    .min(1, errorMessages.required('ชื่อรายการ'))
    .min(2, errorMessages.minLength('ชื่อรายการ', 2))
    .max(255, errorMessages.maxLength('ชื่อรายการ', 255))
    .trim(),
  price_per_unit: priceSchema,
  unit_pack: z
    .string()
    .max(50, errorMessages.maxLength('หน่วยนับ', 50))
    .trim()
    .default(''),
  category: z
    .string()
    .min(1, errorMessages.selectOption('หมวดหมู่'))
    .max(100, errorMessages.maxLength('หมวดหมู่', 100)),
  is_active: z.boolean().default(true),
  is_available: z.boolean().default(true),
  notes: z
    .string()
    .max(500, errorMessages.maxLength('หมายเหตุ', 500))
    .trim()
    .nullable()
    .optional()
    .transform((val) => (val === '' ? null : val)),
})

export type ItemInput = z.infer<typeof itemSchema>

// ==========================================
// PCU Personnel Schemas
// ==========================================

/**
 * PCU personnel information validation
 */
export const pcuPersonnelSchema = z.object({
  pcu_id: idSchema,
  requester_name: z
    .string()
    .max(100, errorMessages.maxLength('ชื่อผู้เบิก', 100))
    .trim()
    .nullable()
    .optional()
    .transform((val) => (val === '' ? null : val)),
  requester_position: z
    .string()
    .max(100, errorMessages.maxLength('ตำแหน่งผู้เบิก', 100))
    .trim()
    .nullable()
    .optional()
    .transform((val) => (val === '' ? null : val)),
  receiver_name: z
    .string()
    .max(100, errorMessages.maxLength('ชื่อผู้รับของ', 100))
    .trim()
    .nullable()
    .optional()
    .transform((val) => (val === '' ? null : val)),
  receiver_position: z
    .string()
    .max(100, errorMessages.maxLength('ตำแหน่งผู้รับของ', 100))
    .trim()
    .nullable()
    .optional()
    .transform((val) => (val === '' ? null : val)),
})

export type PCUPersonnelInput = z.infer<typeof pcuPersonnelSchema>

// ==========================================
// Admin Approval Schemas
// ==========================================

/**
 * Approved quantity update validation
 */
export const approvedQuantitySchema = z.object({
  id: idSchema,
  approved_quantity: z
    .number()
    .int()
    .min(0, errorMessages.positiveNumber('จำนวนอนุมัติ'))
    .max(999999, errorMessages.maxValue('จำนวนอนุมัติ', 999999)),
})

export type ApprovedQuantityInput = z.infer<typeof approvedQuantitySchema>

/**
 * User status update validation
 */
export const userStatusSchema = z.object({
  userId: z.string().uuid(),
  status: z.enum(['approved', 'rejected'], {
    errorMap: () => ({ message: 'สถานะไม่ถูกต้อง' }),
  }),
})

export type UserStatusInput = z.infer<typeof userStatusSchema>

// ==========================================
// Report/Query Schemas
// ==========================================

/**
 * Period ID parameter validation
 */
export const periodIdParamSchema = z.object({
  periodId: z
    .string()
    .transform((val) => parseInt(val, 10))
    .refine((val) => !isNaN(val) && val > 0, {
      message: 'รหัสรอบเบิกไม่ถูกต้อง',
    }),
})

export type PeriodIdParam = z.infer<typeof periodIdParamSchema>

/**
 * Requisition ID parameter validation
 */
export const requisitionIdParamSchema = z.object({
  requisitionId: z
    .string()
    .transform((val) => parseInt(val, 10))
    .refine((val) => !isNaN(val) && val > 0, {
      message: 'รหัสใบเบิกไม่ถูกต้อง',
    }),
})

export type RequisitionIdParam = z.infer<typeof requisitionIdParamSchema>

// ==========================================
// Utility Functions for Validation
// ==========================================

/**
 * Safely parse data with Zod schema
 * Returns { success: true, data: T } or { success: false, errors: string[] }
 */
export const safeParse = <T>(schema: z.ZodType<T>, data: unknown) => {
  const result = schema.safeParse(data)
  
  if (result.success) {
    return { success: true as const, data: result.data }
  }
  
  const errors = result.error.errors.map((err) => err.message)
  return { success: false as const, errors }
}

/**
 * Parse form data with detailed field errors
 * Returns object with field-level error messages
 */
export const parseForm = <T>(schema: z.ZodType<T>, data: unknown) => {
  const result = schema.safeParse(data)
  
  if (result.success) {
    return {
      success: true as const,
      data: result.data,
      fieldErrors: {} as Record<string, string>,
    }
  }
  
  const fieldErrors: Record<string, string> = {}
  result.error.errors.forEach((err) => {
    const path = err.path.join('.')
    fieldErrors[path] = err.message
  })
  
  return { success: false as const, fieldErrors }
}

/**
 * Async validation wrapper for API calls
 */
export const validateAsync = async <T>(
  schema: z.ZodType<T>,
  data: unknown,
  fetchFn: (validatedData: T) => Promise<unknown>
) => {
  const parseResult = safeParse(schema, data)
  
  if (!parseResult.success) {
    return { success: false as const, errors: parseResult.errors }
  }
  
  try {
    const result = await fetchFn(parseResult.data)
    return { success: true as const, data: result }
  } catch (error) {
    return {
      success: false as const,
      errors: [error instanceof Error ? error.message : 'Unknown error'],
    }
  }
}
```

---

## Phase 1: Composable for Validation

### Step 1.1: Create src/composables/useValidation.ts

```typescript
/**
 * Vue composable for Zod validation
 * Provides reactive validation state and helper functions
 */

import { ref, computed } from 'vue'
import type { ZodType } from 'zod'
import { parseForm, safeParse } from '@/schemas'

interface ValidationState<T> {
  isValid: boolean
  errors: string[]
  fieldErrors: Record<string, string>
  data: T | null
}

export function useValidation<T>(schema: ZodType<T>) {
  const state = ref<ValidationState<T>>({
    isValid: false,
    errors: [],
    fieldErrors: {},
    data: null,
  })

  const hasErrors = computed(() => state.value.errors.length > 0)
  const firstError = computed(() => state.value.errors[0] || null)

  /**
   * Validate data against schema
   */
  const validate = (data: unknown): data is T => {
    const result = parseForm(schema, data)
    
    if (result.success) {
      state.value = {
        isValid: true,
        errors: [],
        fieldErrors: {},
        data: result.data,
      }
      return true
    }
    
    state.value = {
      isValid: false,
      errors: Object.values(result.fieldErrors),
      fieldErrors: result.fieldErrors,
      data: null,
    }
    return false
  }

  /**
   * Validate without updating state (for checking only)
   */
  const check = (data: unknown): boolean => {
    const result = safeParse(schema, data)
    return result.success
  }

  /**
   * Get error for specific field
   */
  const getFieldError = (fieldPath: string): string | undefined => {
    return state.value.fieldErrors[fieldPath]
  }

  /**
   * Clear all validation state
   */
  const clear = () => {
    state.value = {
      isValid: false,
      errors: [],
      fieldErrors: {},
      data: null,
    }
  }

  /**
   * Set external errors (e.g., from API)
   */
  const setExternalErrors = (errors: string[]) => {
    state.value.errors = errors
    state.value.isValid = false
  }

  return {
    // State (readonly)
    isValid: computed(() => state.value.isValid),
    errors: computed(() => state.value.errors),
    fieldErrors: computed(() => state.value.fieldErrors),
    data: computed(() => state.value.data),
    hasErrors,
    firstError,
    
    // Methods
    validate,
    check,
    getFieldError,
    clear,
    setExternalErrors,
  }
}

/**
 * Composable for form validation with real-time checking
 */
export function useFormValidation<T extends Record<string, unknown>>(
  schema: ZodType<T>,
  initialData: T
) {
  const formData = ref<T>({ ...initialData })
  const touchedFields = ref<Set<string>>(new Set())
  const dirtyFields = ref<Set<string>>(new Set())

  const { validate, getFieldError, clear, ...validationState } = useValidation(schema)

  /**
   * Mark field as touched (for showing errors after blur)
   */
  const touch = (fieldPath: string) => {
    touchedFields.value.add(fieldPath)
  }

  /**
   * Check if field has been touched
   */
  const isTouched = (fieldPath: string): boolean => {
    return touchedFields.value.has(fieldPath)
  }

  /**
   * Check if field is dirty (value changed from initial)
   */
  const isDirty = (fieldPath: string): boolean => {
    return dirtyFields.value.has(fieldPath)
  }

  /**
   * Update field value with tracking
   */
  const updateField = <K extends keyof T>(field: K, value: T[K]) => {
    const initialValue = initialData[field]
    formData.value[field] = value
    
    dirtyFields.value.add(String(field))
    
    // Auto-validate on change if field has been touched
    if (touchedFields.value.has(String(field))) {
      validate(formData.value)
    }
  }

  /**
   * Get error message for field (only if touched)
   */
  const getVisibleError = (fieldPath: string): string | undefined => {
    if (!touchedFields.value.has(fieldPath)) return undefined
    return getFieldError(fieldPath)
  }

  /**
   * Validate entire form
   */
  const validateForm = (): boolean => {
    // Mark all fields as touched
    Object.keys(formData.value).forEach((key) => touchedFields.value.add(key))
    return validate(formData.value)
  }

  /**
   * Reset form to initial state
   */
  const reset = () => {
    formData.value = { ...initialData }
    touchedFields.value.clear()
    dirtyFields.value.clear()
    clear()
  }

  /**
   * Submit handler wrapper
   */
  const handleSubmit = async <R>(
    submitFn: (data: T) => Promise<R>
  ): Promise<{ success: true; result: R } | { success: false; errors: string[] }> => {
    const isValid = validateForm()
    
    if (!isValid) {
      return { success: false, errors: validationState.errors.value }
    }
    
    try {
      const result = await submitFn(formData.value)
      return { success: true, result }
    } catch (error) {
      const message = error instanceof Error ? error.message : 'Submission failed'
      return { success: false, errors: [message] }
    }
  }

  return {
    // Form state
    formData,
    isDirty,
    isTouched,
    
    // Validation state (from useValidation)
    ...validationState,
    
    // Methods
    touch,
    updateField,
    getVisibleError,
    validateForm,
    reset,
    handleSubmit,
  }
}
```

---

## Phase 2: Component Integration (Progressive)

### Step 2.1: Login.vue with Zod Validation

```vue
<!-- src/views/Login.vue -->
<template>
  <div class="login-wrapper">
    <div class="card login-container">
      <div class="login-header">
        <i class="fas fa-hospital-user"></i>
        <h1>เข้าสู่ระบบ</h1>
        <p>ระบบเบิกยาออนไลน์ โรงพยาบาลสระโบสถ์</p>
      </div>
      
      <form @submit.prevent="onSubmit">
        <div class="form-group" :class="{ 'has-error': emailError }">
          <label for="email">อีเมล <span class="required">*</span></label>
          <input
            type="email"
            id="email"
            v-model="formData.email"
            @blur="touch('email')"
            :class="{ 'input-error': emailError }"
            placeholder="name@example.com"
            autocomplete="email"
          />
          <span v-if="emailError" class="error-text">{{ emailError }}</span>
        </div>
        
        <div class="form-group" :class="{ 'has-error': passwordError }">
          <label for="password">รหัสผ่าน <span class="required">*</span></label>
          <input
            type="password"
            id="password"
            v-model="formData.password"
            @blur="touch('password')"
            :class="{ 'input-error': passwordError }"
            placeholder="********"
            autocomplete="current-password"
          />
          <span v-if="passwordError" class="error-text">{{ passwordError }}</span>
        </div>
        
        <!-- Global error display -->
        <div v-if="globalError" class="alert alert-error">
          <i class="fas fa-exclamation-circle"></i>
          {{ globalError }}
        </div>
        
        <button
          type="submit"
          class="btn btn-primary"
          :disabled="isSubmitting || !isDirty"
        >
          <i class="fas fa-sign-in-alt"></i>
          <span>{{ isSubmitting ? 'กำลังเข้าสู่ระบบ...' : 'เข้าสู่ระบบ' }}</span>
        </button>
        
        <div class="register-link">
          ยังไม่มีบัญชี?
          <router-link to="/register">ลงทะเบียนที่นี่</router-link>
        </div>
      </form>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { useRouter } from 'vue-router'
import { useAuth } from '@/composables/useAuth'
import { useFormValidation } from '@/composables/useValidation'
import { loginSchema } from '@/schemas'
import type { LoginInput } from '@/schemas'

const router = useRouter()
const { login } = useAuth()

const initialData: LoginInput = {
  email: '',
  password: '',
}

const {
  formData,
  updateField,
  touch,
  getVisibleError,
  validateForm,
  handleSubmit,
  isValid,
  errors,
  reset,
} = useFormValidation(loginSchema, initialData)

const isSubmitting = ref(false)
const globalError = ref('')

// Computed error messages for template
const emailError = computed(() => getVisibleError('email'))
const passwordError = computed(() => getVisibleError('password'))
const isDirty = computed(() => formData.email !== '' || formData.password !== '')

const onSubmit = async () => {
  globalError.value = ''
  isSubmitting.value = true
  
  const result = await handleSubmit(async (data) => {
    await login(data.email, data.password)
    router.push('/')
  })
  
  if (!result.success) {
    // Check if it's a validation error or API error
    if (!isValid.value) {
      // Validation errors are already displayed per-field
    } else {
      // API error
      globalError.value = result.errors[0] || 'เข้าสู่ระบบไม่สำเร็จ'
    }
  }
  
  isSubmitting.value = false
}
</script>

<style scoped>
/* Add validation styles */
.has-error label {
  color: var(--danger-color);
}

.input-error {
  border-color: var(--danger-color) !important;
  background-color: #fff5f5;
}

.error-text {
  display: block;
  color: var(--danger-color);
  font-size: 0.875rem;
  margin-top: 0.25rem;
}

.alert {
  padding: 0.75rem 1rem;
  border-radius: var(--border-radius);
  margin-bottom: 1rem;
}

.alert-error {
  background-color: #fff5f5;
  color: var(--danger-color);
  border: 1px solid var(--danger-color);
}

.alert i {
  margin-right: 0.5rem;
}

.required {
  color: var(--danger-color);
}
</style>
```

### Step 2.2: Register.vue with Zod Validation

```vue
<!-- src/views/Register.vue -->
<template>
  <div class="register-wrapper">
    <div class="card register-container">
      <!-- Header unchanged -->
      
      <form @submit.prevent="onSubmit">
        <!-- Username -->
        <div class="form-group" :class="{ 'has-error': getVisibleError('username') }">
          <label for="username">
            ชื่อ-นามสกุล <span class="required">*</span>
          </label>
          <input
            type="text"
            id="username"
            :value="formData.username"
            @input="updateField('username', ($event.target as HTMLInputElement).value)"
            @blur="touch('username')"
            :class="{ 'input-error': getVisibleError('username') }"
            placeholder="เช่น สมชาย ใจดี"
            :disabled="isSubmitting"
          />
          <span v-if="getVisibleError('username')" class="error-text">
            {{ getVisibleError('username') }}
          </span>
        </div>

        <!-- Email -->
        <div class="form-group" :class="{ 'has-error': getVisibleError('email') }">
          <label for="email">อีเมล <span class="required">*</span></label>
          <input
            type="email"
            id="email"
            :value="formData.email"
            @input="updateField('email', ($event.target as HTMLInputElement).value)"
            @blur="touch('email')"
            :class="{ 'input-error': getVisibleError('email') }"
            placeholder="name@example.com"
            :disabled="isSubmitting"
          />
          <span v-if="getVisibleError('email')" class="error-text">
            {{ getVisibleError('email') }}
          </span>
        </div>

        <!-- Password -->
        <div class="form-group" :class="{ 'has-error': getVisibleError('password') }">
          <label for="password">
            รหัสผ่าน <span class="required">*</span>
            <small class="hint">(อย่างน้อย 6 ตัว มีตัวอักษรและตัวเลข)</small>
          </label>
          <input
            type="password"
            id="password"
            :value="formData.password"
            @input="updateField('password', ($event.target as HTMLInputElement).value)"
            @blur="touch('password')"
            :class="{ 'input-error': getVisibleError('password') }"
            placeholder="********"
            :disabled="isSubmitting"
          />
          <span v-if="getVisibleError('password')" class="error-text">
            {{ getVisibleError('password') }}
          </span>
        </div>

        <!-- Confirm Password -->
        <div class="form-group" :class="{ 'has-error': getVisibleError('confirmPassword') }">
          <label for="confirmPassword">
            ยืนยันรหัสผ่าน <span class="required">*</span>
          </label>
          <input
            type="password"
            id="confirmPassword"
            :value="formData.confirmPassword"
            @input="updateField('confirmPassword', ($event.target as HTMLInputElement).value)"
            @blur="touch('confirmPassword')"
            :class="{ 'input-error': getVisibleError('confirmPassword') }"
            placeholder="********"
            :disabled="isSubmitting"
          />
          <span v-if="getVisibleError('confirmPassword')" class="error-text">
            {{ getVisibleError('confirmPassword') }}
          </span>
        </div>

        <!-- PCU Select -->
        <div class="form-group" :class="{ 'has-error': getVisibleError('pcu_id') }">
          <label for="pcu">
            หน่วยบริการ (รพ.สต.) <span class="required">*</span>
          </label>
          <select
            id="pcu"
            :value="formData.pcu_id"
            @change="updateField('pcu_id', Number(($event.target as HTMLSelectElement).value))"
            @blur="touch('pcu_id')"
            :class="{ 'input-error': getVisibleError('pcu_id') }"
            :disabled="isPcuLoading || isSubmitting"
          >
            <option v-if="isPcuLoading" disabled value="">กำลังโหลดรายชื่อ...</option>
            <option v-else disabled value="">-- กรุณาเลือกรพ.สต. --</option>
            <option v-for="pcu in pcuList" :key="pcu.id" :value="pcu.id">
              {{ pcu.name }}
            </option>
          </select>
          <span v-if="getVisibleError('pcu_id')" class="error-text">
            {{ getVisibleError('pcu_id') }}
          </span>
        </div>

        <!-- Global error -->
        <div v-if="globalError" class="alert alert-error">
          <i class="fas fa-exclamation-circle"></i>
          {{ globalError }}
        </div>

        <button type="submit" class="btn btn-primary" :disabled="isSubmitting">
          <i v-if="isSubmitting" class="fas fa-spinner fa-spin"></i>
          <i v-else class="fas fa-check-circle"></i>
          <span>{{ isSubmitting ? 'กำลังลงทะเบียน...' : 'ลงทะเบียน' }}</span>
        </button>
      </form>

      <div class="login-link">
        มีบัญชีอยู่แล้ว?
        <router-link to="/login">เข้าสู่ระบบที่นี่</router-link>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { supabase } from '@/supabaseClient'
import { useFormValidation } from '@/composables/useValidation'
import { registerSchema } from '@/schemas'
import type { RegisterInput } from '@/schemas'
import type { PCU } from '@/types'

const router = useRouter()
const pcuList = ref<PCU[]>([])
const isPcuLoading = ref(true)
const globalError = ref('')

const initialData: RegisterInput = {
  username: '',
  email: '',
  password: '',
  confirmPassword: '',
  pcu_id: '' as unknown as number, // Will be transformed by schema
}

const {
  formData,
  updateField,
  touch,
  getVisibleError,
  validateForm,
  handleSubmit,
  isValid,
  reset,
} = useFormValidation(registerSchema, initialData)

const isSubmitting = ref(false)

const fetchPcuList = async () => {
  try {
    isPcuLoading.value = true
    const { data, error } = await supabase
      .from('pcus_drugcupsabot')
      .select('id, name')
      .order('name')
    
    if (error) throw error
    pcuList.value = data || []
  } catch (err) {
    globalError.value = 'ไม่สามารถโหลดรายชื่อ รพ.สต. ได้'
    console.error(err)
  } finally {
    isPcuLoading.value = false
  }
}

onMounted(fetchPcuList)

const onSubmit = async () => {
  globalError.value = ''
  isSubmitting.value = true

  const result = await handleSubmit(async (data) => {
    const APP_BASE_URL = import.meta.env.VITE_APP_BASE_URL || window.location.origin
    const EMAIL_REDIRECT_URL = `${APP_BASE_URL}/waiting-for-approval`

    const { error } = await supabase.auth.signUp({
      email: data.email,
      password: data.password,
      options: {
        data: {
          username: data.username.trim(),
          pcu_id: data.pcu_id,
          email: data.email.trim(),
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

    alert('การลงทะเบียนสำเร็จ! กรุณารอการอนุมัติจากผู้ดูแลระบบ')
    router.push('/login')
  })

  if (!result.success) {
    if (isValid.value) {
      // API error
      globalError.value = result.errors[0] || 'ลงทะเบียนไม่สำเร็จ'
    }
    // Validation errors shown per-field
  }

  isSubmitting.value = false
}
</script>

<style scoped>
.hint {
  display: block;
  font-size: 0.8rem;
  color: var(--text-muted);
  font-weight: normal;
}
</style>
```

---

## Phase 3: Requisition Form Validation (Complex)

### Step 3.1: Create src/composables/useRequisitionForm.ts

```typescript
/**
 * Specialized composable for requisition form with item-level validation
 */

import { ref, computed } from 'vue'
import type { Item, RequisitionFormData } from '@/types'
import { quantitySchema } from '@/schemas'
import { safeParse } from '@/schemas'

interface ItemValidationState {
  quantity: string | null
  onHand: string | null
}

export function useRequisitionForm(items: Item[]) {
  const formData = ref<RequisitionFormData>({})
  const touchedItems = ref<Set<number>>(new Set())
  const itemErrors = ref<Map<number, ItemValidationState>>(new Map())

  // Initialize form data
  const initialize = () => {
    const initial: RequisitionFormData = {}
    items.forEach((item) => {
      initial[item.id] = {
        quantity: null,
        onHand: null,
      }
    })
    formData.value = initial
    itemErrors.value.clear()
    touchedItems.value.clear()
  }

  // Validate single item
  const validateItem = (itemId: number): boolean => {
    const item = items.find((i) => i.id === itemId)
    if (!item) return false

    const data = formData.value[itemId]
    const errors: ItemValidationState = {
      quantity: null,
      onHand: null,
    }

    // Only validate if item is available
    if (!item.is_available) return true

    // Validate quantity
    if (data?.quantity !== null && data?.quantity !== undefined) {
      const result = safeParse(quantitySchema('จำนวนเบิก'), data.quantity)
      if (!result.success) {
        errors.quantity = result.errors[0]
      }
    }

    // Validate on-hand quantity
    if (data?.onHand !== null && data?.onHand !== undefined) {
      const result = safeParse(quantitySchema('ยอดคงเหลือ'), data.onHand)
      if (!result.success) {
        errors.onHand = result.errors[0]
      }
    }

    itemErrors.value.set(itemId, errors)
    return !errors.quantity && !errors.onHand
  }

  // Touch item (show errors)
  const touchItem = (itemId: number) => {
    touchedItems.value.add(itemId)
    validateItem(itemId)
  }

  // Update item quantity
  const updateQuantity = (itemId: number, value: number | null) => {
    if (!formData.value[itemId]) {
      formData.value[itemId] = { quantity: null, onHand: null }
    }
    formData.value[itemId].quantity = value
    
    if (touchedItems.value.has(itemId)) {
      validateItem(itemId)
    }
  }

  // Update item on-hand
  const updateOnHand = (itemId: number, value: number | null) => {
    if (!formData.value[itemId]) {
      formData.value[itemId] = { quantity: null, onHand: null }
    }
    formData.value[itemId].onHand = value
    
    if (touchedItems.value.has(itemId)) {
      validateItem(itemId)
    }
  }

  // Get visible error for item field
  const getItemError = (itemId: number, field: keyof ItemValidationState): string | null => {
    if (!touchedItems.value.has(itemId)) return null
    return itemErrors.value.get(itemId)?.[field] || null
  }

  // Validate all items
  const validateAll = (): boolean => {
    let isValid = true
    
    items.forEach((item) => {
      if (item.is_available) {
        touchedItems.value.add(item.id)
        const itemValid = validateItem(item.id)
        if (!itemValid) isValid = false
      }
    })

    // Check at least one item has quantity > 0
    const hasItems = Object.values(formData.value).some(
      (d) => d.quantity && d.quantity > 0
    )
    
    if (!hasItems) {
      isValid = false
    }

    return isValid
  }

  // Get items for submission (filter out empty and unavailable)
  const getSubmitData = () => {
    return Object.entries(formData.value)
      .filter(([itemId, data]) => {
        const item = items.find((i) => i.id === Number(itemId))
        return item?.is_available && data.quantity && data.quantity > 0
      })
      .map(([itemId, data]) => ({
        item_id: Number(itemId),
        quantity: data.quantity!,
        on_hand_quantity: data.onHand,
      }))
  }

  // Calculate totals
  const totalValue = computed(() => {
    return items.reduce((sum, item) => {
      if (!item.is_available) return sum
      const quantity = formData.value[item.id]?.quantity || 0
      return sum + quantity * item.price_per_unit
    }, 0)
  })

  const itemCount = computed(() => {
    return Object.values(formData.value).filter((d) => d.quantity && d.quantity > 0).length
  })

  return {
    formData,
    initialize,
    updateQuantity,
    updateOnHand,
    touchItem,
    getItemError,
    validateAll,
    getSubmitData,
    totalValue,
    itemCount,
    hasItems: computed(() => itemCount.value > 0),
  }
}
```

### Step 3.2: RequisitionForm.vue with Validation

```vue
<!-- src/views/pcu/RequisitionForm.vue -->
<template>
  <div class="container">
    <h2>
      <i class="fas fa-file-signature"></i>
      {{ isEditing ? 'แก้ไขใบเบิก' : 'สร้างใบเบิกใหม่' }}
    </h2>
    <p class="text-muted">{{ periodInfo?.name }}</p>

    <!-- Validation summary -->
    <div v-if="showValidationSummary && !isValid" class="alert alert-warning">
      <i class="fas fa-exclamation-triangle"></i>
      กรุณาตรวจสอบข้อมูล:
      <ul>
        <li v-if="!hasItems">ต้องมีอย่างน้อย 1 รายการที่มีจำนวนเบิกมากกว่า 0</li>
        <li v-for="error in validationErrors" :key="error">{{ error }}</li>
      </ul>
    </div>

    <div class="toolbar card">
      <div class="form-group">
        <label for="search"><i class="fas fa-search"></i> ค้นหารายการ</label>
        <input
          type="text"
          id="search"
          v-model="searchTerm"
          placeholder="พิมพ์ชื่อยาเพื่อค้นหา..."
          class="search-input"
        />
      </div>
      <div class="summary-badge">
        <span class="badge">
          <i class="fas fa-shopping-cart"></i>
          {{ itemCount }} รายการ
        </span>
        <span class="badge badge-primary">
          รวม: {{ formatCurrency(totalValue) }} บาท
        </span>
      </div>
    </div>

    <div v-if="loading" class="loading">กำลังโหลดรายการ...</div>
    <div v-if="error" class="error">{{ error }}</div>

    <form @submit.prevent="submitForm" v-if="!loading && !error">
      <div class="table-wrapper card">
        <table>
          <thead>
            <tr>
              <th>ลำดับ</th>
              <th>ประเภท</th>
              <th>รายการเวชภัณฑ์</th>
              <th>หน่วยนับ</th>
              <th class="text-right">ราคา</th>
              <th class="text-center">คงเหลือ</th>
              <th class="text-center">จำนวนเบิก</th>
              <th class="text-right">มูลค่า</th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="item in filteredItems"
              :key="item.id"
              :class="{ 
                'locked-item': !item.is_available,
                'has-error': getItemError(item.id, 'quantity') || getItemError(item.id, 'onHand')
              }"
            >
              <td class="text-center">{{ item.item_order }}</td>
              <td>{{ item.category }}</td>
              <td>
                {{ item.name }}
                <span v-if="!item.is_available && item.notes" class="item-note">
                  ({{ item.notes }})
                </span>
                <span v-if="getItemError(item.id, 'quantity')" class="field-error">
                  {{ getItemError(item.id, 'quantity') }}
                </span>
              </td>
              <td class="text-center">{{ item.unit_pack }}</td>
              <td class="text-right">
                {{ formatCurrency(item.price_per_unit) }}
              </td>
              <td class="text-center">
                <input
                  type="number"
                  min="0"
                  class="quantity-input"
                  :class="{ 'input-error': getItemError(item.id, 'onHand') }"
                  :value="formData[item.id]?.onHand"
                  @input="updateOnHand(item.id, parseNumber($event.target.value))"
                  @blur="touchItem(item.id)"
                  :disabled="!item.is_available"
                />
              </td>
              <td class="text-center">
                <input
                  type="number"
                  min="0"
                  class="quantity-input"
                  :class="{ 'input-error': getItemError(item.id, 'quantity') }"
                  :value="formData[item.id]?.quantity"
                  @input="updateQuantity(item.id, parseNumber($event.target.value))"
                  @blur="touchItem(item.id)"
                  :disabled="!item.is_available"
                />
              </td>
              <td class="text-right">
                {{ formatCurrency(calculateItemValue(item)) }}
              </td>
            </tr>
          </tbody>
          <tfoot>
            <tr>
              <td colspan="7" class="text-right bold">รวมมูลค่าทั้งหมด</td>
              <td class="text-right bold">
                {{ formatCurrency(totalValue) }}
              </td>
            </tr>
          </tfoot>
        </table>
      </div>

      <div class="form-actions">
        <button
          type="button"
          @click="saveDraft"
          class="btn btn-secondary"
          :disabled="isSubmitting || !hasItems"
        >
          <i class="fas fa-save"></i>
          {{ isSubmitting ? 'กำลังบันทึก...' : 'บันทึกฉบับร่าง' }}
        </button>
        <button
          type="submit"
          class="btn btn-success"
          :disabled="isSubmitting || !hasItems"
        >
          <i class="fas fa-paper-plane"></i>
          {{ isSubmitting ? 'กำลังส่ง...' : 'ส่งใบเบิก' }}
        </button>
      </div>
    </form>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { useAuth } from '@/composables/useAuth'
import { useRequisitionForm } from '@/composables/useRequisitionForm'
import { itemService, requisitionService, requisitionItemService } from '@/services/supabaseService'
import { formatCurrency } from '@/utils/formatters'
import type { Item, RequisitionPeriod } from '@/types'

const props = defineProps<{
  periodId: string
  requisitionId: string
}>()

const router = useRouter()
const auth = useAuth()

const loading = ref(true)
const isSubmitting = ref(false)
const error = ref('')
const showValidationSummary = ref(false)
const validationErrors = ref<string[]>([])

const items = ref<Item[]>([])
const periodInfo = ref<RequisitionPeriod | null>(null)
const searchTerm = ref('')

const isEditing = computed(() => props.requisitionId && props.requisitionId !== 'new')

const {
  formData,
  initialize,
  updateQuantity,
  updateOnHand,
  touchItem,
  getItemError,
  validateAll,
  getSubmitData,
  totalValue,
  itemCount,
  hasItems,
} = useRequisitionForm(items.value)

const isValid = computed(() => hasItems.value && validationErrors.value.length === 0)

const filteredItems = computed(() => {
  if (!searchTerm.value) return items.value
  return items.value.filter((item) =>
    item.name.toLowerCase().includes(searchTerm.value.toLowerCase())
  )
})

const parseNumber = (value: string): number | null => {
  if (value === '' || value === null) return null
  const num = parseInt(value, 10)
  return isNaN(num) ? null : num
}

const calculateItemValue = (item: Item): number => {
  if (!item.is_available) return 0
  const quantity = formData.value[item.id]?.quantity || 0
  return quantity * item.price_per_unit
}

onMounted(async () => {
  try {
    // Load period info
    const { data: periodData } = await itemService.getActive() // Replace with period service
    // ... load period info

    // Load items
    const { data: itemsData, error: itemsError } = await itemService.getActive()
    if (itemsError) throw itemsError
    items.value = itemsData || []

    // Initialize form
    initialize()

    // Load existing data if editing
    if (isEditing.value) {
      const reqId = parseInt(props.requisitionId, 10)
      const { data: existingData, error: draftError } = await requisitionItemService.getByRequisition(reqId)
      
      if (draftError) throw draftError
      
      if (existingData) {
        existingData.forEach((item) => {
          updateQuantity(item.item_id, item.quantity)
          updateOnHand(item.item_id, item.on_hand_quantity)
        })
      }
    }
  } catch (err) {
    error.value = 'ไม่สามารถโหลดข้อมูลได้'
    console.error(err)
  } finally {
    loading.value = false
  }
})

const validateForm = (): boolean => {
  showValidationSummary.value = true
  validationErrors.value = []
  
  const itemsValid = validateAll()
  
  if (!hasItems.value) {
    validationErrors.value.push('กรุณาระบุจำนวนเบิกอย่างน้อย 1 รายการ')
  }
  
  return itemsValid && hasItems.value
}

const saveRequisition = async (status: 'draft' | 'submitted') => {
  if (!validateForm()) return false
  
  if (status === 'submitted') {
    const confirmed = confirm(
      'คุณต้องการยืนยันการส่งใบเบิกนี้ใช่หรือไม่? เมื่อส่งแล้วจะไม่สามารถแก้ไขได้อีก'
    )
    if (!confirmed) return false
  }

  isSubmitting.value = true
  
  try {
    const pcuId = auth.userPcuId.value
    if (!pcuId) throw new Error('ไม่พบข้อมูลรพ.สต.')
    
    const periodIdNum = parseInt(props.periodId, 10)
    const submitItems = getSubmitData()

    let reqId = isEditing.value ? parseInt(props.requisitionId, 10) : null

    // Create new if needed
    if (!reqId) {
      const { data, error: createError } = await requisitionService.create(
        pcuId,
        periodIdNum,
        auth.userProfile.value?.id || ''
      )
      if (createError) throw createError
      reqId = data!.id
    }

    // Delete existing items
    await requisitionItemService.deleteByRequisition(reqId)

    // Insert new items
    const itemsToInsert = submitItems.map((item) => ({
      requisition_id: reqId!,
      item_id: item.item_id,
      quantity: item.quantity,
      on_hand_quantity: item.on_hand_quantity,
      price_at_request: items.value.find((i) => i.id === item.item_id)?.price_per_unit || 0,
    }))

    if (itemsToInsert.length > 0) {
      await requisitionItemService.createBatch(itemsToInsert)
    }

    // Update status
    await requisitionService.updateStatus(
      reqId,
      status,
      status === 'submitted' ? new Date().toISOString() : null
    )

    alert(`ใบเบิกถูก "${status === 'submitted' ? 'ส่ง' : 'บันทึก'}" เรียบร้อยแล้ว`)
    router.push('/pcu/dashboard')
    return true
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'เกิดข้อผิดพลาด'
    return false
  } finally {
    isSubmitting.value = false
  }
}

const saveDraft = () => saveRequisition('draft')
const submitForm = () => saveRequisition('submitted')
</script>

<style scoped>
.summary-badge {
  display: flex;
  gap: 1rem;
  align-items: center;
}

.badge {
  padding: 0.5rem 1rem;
  background-color: var(--bg-color);
  border-radius: var(--border-radius);
  font-weight: 500;
}

.badge-primary {
  background-color: var(--primary-color);
  color: white;
}

.field-error {
  display: block;
  color: var(--danger-color);
  font-size: 0.75rem;
  margin-top: 0.25rem;
}

tr.has-error td {
  background-color: #fff5f5;
}

.alert-warning {
  background-color: #fffbeb;
  border: 1px solid #f59e0b;
  color: #92400e;
  padding: 1rem;
  border-radius: var(--border-radius);
  margin-bottom: 1rem;
}

.alert-warning ul {
  margin: 0.5rem 0 0 1.5rem;
}
</style>
```

---

## Phase 4: Admin Forms Validation

### Step 4.1: ItemManagement.vue Validation

```vue
<!-- Add to ItemManagement.vue template -->
<td>
  <input
    type="text"
    :value="item.name"
    @blur="validateAndUpdate(item, 'name', $event.target.value)"
    :class="{ 'input-error': getFieldError(item.id, 'name') }"
    class="form-control-table"
  />
  <span v-if="getFieldError(item.id, 'name')" class="field-error">
    {{ getFieldError(item.id, 'name') }}
  </span>
</td>
<td>
  <input
    type="number"
    step="0.01"
    :value="item.price_per_unit"
    @blur="validateAndUpdate(item, 'price_per_unit', $event.target.value)"
    :class="{ 'input-error': getFieldError(item.id, 'price_per_unit') }"
    class="form-control-table text-right"
  />
</td>
```

```typescript
// Add to ItemManagement.vue script
import { itemSchema, safeParse } from '@/schemas'
import type { Item } from '@/types'

const fieldErrors = ref<Map<number, Record<string, string>>>(new Map())

const getFieldError = (itemId: number, field: string): string | undefined => {
  return fieldErrors.value.get(itemId)?.[field]
}

const validateAndUpdate = async (item: Item, field: keyof Item, value: unknown) => {
  // Validate single field
  const partialSchema = itemSchema.pick({ [field]: true as never })
  const result = safeParse(partialSchema, { [field]: value })
  
  if (!result.success) {
    const currentErrors = fieldErrors.value.get(item.id) || {}
    fieldErrors.value.set(item.id, {
      ...currentErrors,
      [field]: result.errors[0]
    })
    return
  }
  
  // Clear error
  const currentErrors = fieldErrors.value.get(item.id) || {}
  delete currentErrors[field]
  fieldErrors.value.set(item.id, currentErrors)
  
  // Update item
  await updateItem({ ...item, [field]: value })
}
```

---

## Phase 5: API Input Validation (Defense in Depth)

### Step 5.1: Create src/middleware/validationMiddleware.ts

```typescript
/**
 * Server-side validation middleware for API calls
 * Validates inputs before sending to Supabase
 */

import type { ZodType } from 'zod'
import { safeParse } from '@/schemas'

/**
 * Wrapper for Supabase RPC calls with validation
 */
export const validateAndCall = async <TInput, TOutput>(
  schema: ZodType<TInput>,
  data: unknown,
  callFn: (validated: TInput) => Promise<TOutput>
): Promise<{ success: true; data: TOutput } | { success: false; errors: string[] }> => {
  const validation = safeParse(schema, data)
  
  if (!validation.success) {
    return { success: false, errors: validation.errors }
  }
  
  try {
    const result = await callFn(validation.data)
    return { success: true, data: result }
  } catch (error) {
    return {
      success: false,
      errors: [error instanceof Error ? error.message : 'API call failed']
    }
  }
}

/**
 * Validate route parameters
 */
export const validateParams = <T>(schema: ZodType<T>, params: Record<string, string>) => {
  const result = safeParse(schema, params)
  
  if (!result.success) {
    throw new Error(`Invalid parameters: ${result.errors.join(', ')}`)
  }
  
  return result.data
}
```

---

## 🚨 Critical Implementation Rules

### DO:
- ✅ Validate on blur for immediate feedback
- ✅ Validate on submit for completeness
- ✅ Show field-level errors next to inputs
- ✅ Show global errors for API failures
- ✅ Disable submit while validating
- ✅ Sanitize inputs (trim, lowercase for email)
- ✅ Use `transform` in Zod for type coercion

### DON'T:
- ❌ Show all errors immediately on page load
- ❌ Allow submission with validation errors
- ❌ Trust client-side validation alone (always validate server-side)
- ❌ Use `any` type for form data
- ❌ Mix validation logic with business logic

---

## 📋 Validation Checklist Per Form

```
For each form:
[ ] Define Zod schema in src/schemas/index.ts
[ ] Create form composable if complex
[ ] Add real-time validation (on blur)
[ ] Add submit validation
[ ] Display field-level errors
[ ] Display global errors
[ ] Disable submit on invalid
[ ] Test edge cases (empty, negative, overflow)
[ ] Test API error handling
```

---

## 🎯 Success Criteria

Validation implementation is complete when:

1. All forms use Zod schemas
2. No `alert()` for validation errors (use inline display)
3. Real-time feedback on user input
4. Submit blocked until valid
5. Type-safe form data throughout
6. Server-side validation as backup
7. All edge cases handled gracefully

---

## 🆘 Troubleshooting

### Issue: "ZodError: Invalid type"
**Solution**: Use `.transform()` for type coercion or `.union()` for multiple types.

### Issue: "Validation too strict"
**Solution**: Use `.optional()` and `.nullable()` appropriately, with `.default()` for fallbacks.

### Issue: "Errors not showing"
**Solution**: Ensure `touch()` is called on blur and errors are read from `getVisibleError()`.

---

**Remember**: Validation is about user experience, not just security. Help users succeed with clear, immediate feedback.