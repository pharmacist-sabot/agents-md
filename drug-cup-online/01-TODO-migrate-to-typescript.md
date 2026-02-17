# TypeScript Migration Guide

This document provides a comprehensive, step-by-step guide for migrating the Drug Requisition System from JavaScript to TypeScript without breaking existing functionality. Follow these instructions carefully to maintain production stability.

---

## 🎯 Migration Strategy: "Strangler Fig Pattern"

We will migrate incrementally, file by file, ensuring the application remains functional at every step. Never migrate multiple critical files simultaneously.

---

## Phase 0: Foundation Setup (Do First)

### Step 0.1: Install TypeScript Dependencies

```bash
# Core TypeScript
npm install -D typescript vue-tsc @types/node

# Type definitions for existing dependencies
npm install -D @supabase/supabase-js

# Vite TypeScript support
npm install -D vite-plugin-checker
```

### Step 0.2: Create tsconfig.json

Create `tsconfig.json` in project root:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "module": "ESNext",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "preserve",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "paths": {
      "@/*": ["./src/*"]
    },
    "types": ["vite/client", "node"]
  },
  "include": ["src/**/*.ts", "src/**/*.tsx", "src/**/*.vue"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

Create `tsconfig.node.json`:

```json
{
  "compilerOptions": {
    "composite": true,
    "skipLibCheck": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowSyntheticDefaultImports": true,
    "strict": true
  },
  "include": ["vite.config.ts"]
}
```

### Step 0.3: Rename vite.config.js to vite.config.ts

```typescript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import path from 'path'
import { fileURLToPath } from 'url'

const __filename = fileURLToPath(import.meta.url)
const __dirname = path.dirname(__filename)

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [
    vue(),
    // Add type checking in development
    // checker({ vueTsc: true }) // Uncomment after initial migration
  ],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
})
```

### Step 0.4: Update package.json Scripts

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc --noEmit && vite build",
    "preview": "vite preview",
    "type-check": "vue-tsc --noEmit --composite false",
    "lint": "eslint . --ext .vue,.ts,.tsx --fix"
  }
}
```

---

## Phase 1: Type Definitions (Core Foundation)

### Step 1.1: Create src/types/index.ts

This is the MOST IMPORTANT file. Define all domain types here first.

```typescript
/**
 * Core Domain Types for Drug Requisition System
 * These types represent the database schema and business entities
 */

// ==========================================
// Enums / Union Types
// ==========================================

export type UserRole = 'admin' | 'pcu'
export type RequisitionStatus = 'draft' | 'submitted' | 'approved' | 'fulfilled' | 'rejected'
export type UserStatus = 'pending' | 'approved' | 'rejected'

// ==========================================
// Database Entities
// ==========================================

export interface PCU {
  id: number
  name: string
  created_at?: string
}

export interface PCUPersonnel {
  id?: number
  pcu_id: number
  requester_name: string
  requester_position: string
  receiver_name: string
  receiver_position: string
}

export interface UserProfile {
  id: string
  username: string
  email: string
  role: UserRole
  status: UserStatus
  pcu_id: number | null
  pcus_drugcupsabot?: PCU
  created_at?: string
}

export interface Item {
  id: number
  name: string
  price_per_unit: number
  unit_pack: string
  category: string
  category_order: number | null
  item_order: number | null
  is_active: boolean
  is_available: boolean
  notes: string | null
  created_at?: string
}

export interface RequisitionPeriod {
  id: number
  name: string
  start_date: string
  end_date: string
  status: 'open' | 'closed'
  created_at?: string
}

export interface RequisitionItem {
  id?: number
  requisition_id?: number
  item_id: number
  quantity: number
  approved_quantity: number | null
  on_hand_quantity: number | null
  price_at_request: number
  items_drugcupsabot?: Item
}

export interface Requisition {
  id: number
  pcu_id: number
  period_id: number
  requester_id: string
  status: RequisitionStatus
  submitted_at: string | null
  created_at: string
  pcus_drugcupsabot?: PCU
  requisition_periods_drugcupsabot?: RequisitionPeriod
  requisition_items_drugcupsabot?: RequisitionItem[]
}

// ==========================================
// API Response Types
// ==========================================

export interface SupabaseResponse<T> {
  data: T | null
  error: Error | null
}

// ==========================================
// Form Data Types
// ==========================================

export interface RequisitionFormData {
  [itemId: number]: {
    quantity: number | null
    onHand: number | null
  }
}

export interface LoginForm {
  email: string
  password: string
}

export interface RegisterForm {
  username: string
  email: string
  password: string
  pcu_id: number | ''
}

// ==========================================
// Component Props Types
// ==========================================

export interface SidebarProps {
  isOpen: boolean
}

export interface RequisitionDetailProps {
  requisitionId: string
}

export interface RequisitionFormProps {
  periodId: string
  requisitionId: string
}

// ==========================================
// Utility Types
// ==========================================

export type Nullable<T> = T | null
export type Optional<T> = T | undefined
```

---

## Phase 2: Utility Layer (Safest to Migrate)

### Step 2.1: Create src/utils/formatters.ts

```typescript
/**
 * Formatting utilities with strict type safety
 */

export const formatCurrency = (value: number | null | undefined): string => {
  if (value === null || value === undefined || isNaN(value)) {
    return '0.00'
  }
  return Number(value).toLocaleString('th-TH', {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  })
}

export const formatDate = (
  dateString: string | Date | null | undefined,
  options?: Intl.DateTimeFormatOptions
): string => {
  if (!dateString) return 'N/A'
  
  const defaultOptions: Intl.DateTimeFormatOptions = {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  }
  
  const date = typeof dateString === 'string' ? new Date(dateString) : dateString
  
  if (isNaN(date.getTime())) return 'Invalid Date'
  
  return date.toLocaleDateString('th-TH', options ?? defaultOptions)
}

export const formatDateTime = (
  dateString: string | Date | null | undefined
): string => {
  return formatDate(dateString, {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
  })
}

export const calculateItemValue = (
  quantity: number | null | undefined,
  price: number | null | undefined
): number => {
  const safeQuantity = quantity ?? 0
  const safePrice = price ?? 0
  return Number(safeQuantity) * Number(safePrice)
}

export const calculateGrandTotal = (
  items: Array<{ quantity?: number; approved_quantity?: number | null; price_at_request?: number }>
): number => {
  return items.reduce((sum, item) => {
    const quantity = item.approved_quantity ?? item.quantity ?? 0
    const price = item.price_at_request ?? 0
    return sum + (Number(quantity) * Number(price))
  }, 0)
}
```

### Step 2.2: Create src/utils/validation.ts

```typescript
/**
 * Input validation utilities using Zod-style validation
 * (Can upgrade to Zod later if needed)
 */

export interface ValidationResult<T> {
  success: boolean
  data?: T
  errors?: string[]
}

export const validateEmail = (email: string): boolean => {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  return emailRegex.test(email)
}

export const validatePositiveNumber = (
  value: unknown,
  fieldName: string
): ValidationResult<number> => {
  if (value === null || value === undefined || value === '') {
    return { success: true, data: 0 }
  }
  
  const num = Number(value)
  
  if (isNaN(num)) {
    return {
      success: false,
      errors: [`${fieldName} ต้องเป็นตัวเลข`]
    }
  }
  
  if (num < 0) {
    return {
      success: false,
      errors: [`${fieldName} ต้องไม่ติดลบ`]
    }
  }
  
  return { success: true, data: num }
}

export const validateRequired = <T>(
  value: T | null | undefined,
  fieldName: string
): ValidationResult<T> => {
  if (value === null || value === undefined || value === '') {
    return {
      success: false,
      errors: [`${fieldName} จำเป็นต้องกรอก`]
    }
  }
  return { success: true, data: value }
}
```

---

## Phase 3: API/Service Layer

### Step 3.1: Create src/services/supabaseService.ts

```typescript
/**
 * Typed Supabase service layer
 * All database operations go through here
 */

import { supabase } from '@/supabaseClient'
import type {
  UserProfile,
  PCU,
  Item,
  Requisition,
  RequisitionPeriod,
  RequisitionItem,
  PCUPersonnel,
  SupabaseResponse,
} from '@/types'

// ==========================================
// Error Handling Wrapper
// ==========================================

const handleError = <T>(error: unknown): SupabaseResponse<T> => {
  console.error('Supabase Error:', error)
  return {
    data: null,
    error: error instanceof Error ? error : new Error(String(error))
  }
}

// ==========================================
// Auth Services
// ==========================================

export const authService = {
  async getSession() {
    try {
      const { data, error } = await supabase.auth.getSession()
      if (error) throw error
      return { data, error: null }
    } catch (err) {
      return handleError(err)
    }
  },

  async signIn(email: string, password: string) {
    try {
      const { data, error } = await supabase.auth.signInWithPassword({
        email,
        password,
      })
      if (error) throw error
      return { data, error: null }
    } catch (err) {
      return handleError(err)
    }
  },

  async signOut() {
    try {
      const { error } = await supabase.auth.signOut()
      if (error) throw error
      return { error: null }
    } catch (err) {
      return handleError(err)
    }
  }
}

// ==========================================
// Profile Services
// ==========================================

export const profileService = {
  async getProfile(userId: string): Promise<SupabaseResponse<UserProfile>> {
    try {
      const { data, error } = await supabase
        .from('profiles_drugcupsabot')
        .select('*, pcus_drugcupsabot(name), status')
        .eq('id', userId)
        .single()
      
      if (error) throw error
      return { data: data as UserProfile, error: null }
    } catch (err) {
      return handleError<UserProfile>(err)
    }
  },

  async getPendingUsers(): Promise<SupabaseResponse<UserProfile[]>> {
    try {
      const { data, error } = await supabase
        .from('profiles_drugcupsabot')
        .select('id, username, email, created_at, pcus_drugcupsabot(name)')
        .eq('status', 'pending')
      
      if (error) throw error
      return { data: data as UserProfile[], error: null }
    } catch (err) {
      return handleError<UserProfile[]>(err)
    }
  },

  async updateStatus(userId: string, status: 'approved' | 'rejected') {
    try {
      const { error } = await supabase
        .from('profiles_drugcupsabot')
        .update({ status })
        .eq('id', userId)
      
      if (error) throw error
      return { error: null }
    } catch (err) {
      return handleError(err)
    }
  }
}

// ==========================================
// PCU Services
// ==========================================

export const pcuService = {
  async getAll(): Promise<SupabaseResponse<PCU[]>> {
    try {
      const { data, error } = await supabase
        .from('pcus_drugcupsabot')
        .select('id, name')
        .order('name')
      
      if (error) throw error
      return { data: data as PCU[], error: null }
    } catch (err) {
      return handleError<PCU[]>(err)
    }
  },

  async getPersonnel(pcuId: number): Promise<SupabaseResponse<PCUPersonnel>> {
    try {
      const { data, error } = await supabase
        .from('pcu_personnel_drugcupsabot')
        .select('*')
        .eq('pcu_id', pcuId)
        .single()
      
      if (error && error.code !== 'PGRST116') throw error
      return { data: data as PCUPersonnel, error: null }
    } catch (err) {
      return handleError<PCUPersonnel>(err)
    }
  },

  async savePersonnel(personnel: PCUPersonnel) {
    try {
      const { error } = await supabase
        .from('pcu_personnel_drugcupsabot')
        .upsert(personnel, { onConflict: 'pcu_id' })
      
      if (error) throw error
      return { error: null }
    } catch (err) {
      return handleError(err)
    }
  }
}

// ==========================================
// Item Services
// ==========================================

export const itemService = {
  async getAll(): Promise<SupabaseResponse<Item[]>> {
    try {
      const { data, error } = await supabase
        .from('items_drugcupsabot')
        .select('*')
        .order('category_order', { ascending: true, nullsFirst: false })
        .order('item_order', { ascending: true, nullsFirst: false })
        .order('name', { ascending: true })
      
      if (error) throw error
      return { data: data as Item[], error: null }
    } catch (err) {
      return handleError<Item[]>(err)
    }
  },

  async getActive(): Promise<SupabaseResponse<Item[]>> {
    try {
      const { data, error } = await supabase
        .from('items_drugcupsabot')
        .select('*')
        .eq('is_active', true)
        .order('category_order', { ascending: true })
        .order('item_order', { ascending: true })
      
      if (error) throw error
      return { data: data as Item[], error: null }
    } catch (err) {
      return handleError<Item[]>(err)
    }
  },

  async update(item: Partial<Item> & { id: number }) {
    try {
      const { id, created_at, category, category_order, item_order, ...updateData } = item
      
      const { error } = await supabase
        .from('items_drugcupsabot')
        .update(updateData)
        .eq('id', id)
      
      if (error) throw error
      return { error: null }
    } catch (err) {
      return handleError(err)
    }
  },

  async create(item: Omit<Item, 'id' | 'created_at'>) {
    try {
      const { data, error } = await supabase
        .from('items_drugcupsabot')
        .insert(item)
        .select()
      
      if (error) throw error
      return { data: data as Item[], error: null }
    } catch (err) {
      return handleError(err)
    }
  }
}

// ==========================================
// Requisition Services
// ==========================================

export const requisitionService = {
  async getById(id: number): Promise<SupabaseResponse<Requisition>> {
    try {
      const { data, error } = await supabase
        .from('requisitions_drugcupsabot')
        .select(`
          *,
          pcus_drugcupsabot(name),
          requisition_periods_drugcupsabot(name),
          requisition_items_drugcupsabot(
            *,
            items_drugcupsabot(name, unit_pack)
          )
        `)
        .eq('id', id)
        .single()
      
      if (error) throw error
      return { data: data as Requisition, error: null }
    } catch (err) {
      return handleError<Requisition>(err)
    }
  },

  async getByPCU(pcuId: number): Promise<SupabaseResponse<Requisition[]>> {
    try {
      const { data, error } = await supabase
        .from('requisitions_drugcupsabot')
        .select(`
          id, period_id, status, created_at, submitted_at,
          requisition_periods_drugcupsabot(name)
        `)
        .eq('pcu_id', pcuId)
      
      if (error) throw error
      return { data: data as Requisition[], error: null }
    } catch (err) {
      return handleError<Requisition[]>(err)
    }
  },

  async getByStatus(statuses: string[]): Promise<SupabaseResponse<Requisition[]>> {
    try {
      const { data, error } = await supabase
        .from('requisitions_drugcupsabot')
        .select(`
          id, submitted_at, status,
          pcus_drugcupsabot(name),
          requisition_periods_drugcupsabot(name)
        `)
        .in('status', statuses)
        .order('submitted_at', { ascending: true })
      
      if (error) throw error
      return { data: data as Requisition[], error: null }
    } catch (err) {
      return handleError<Requisition[]>(err)
    }
  },

  async getByPeriod(periodId: number, statuses: string[]): Promise<SupabaseResponse<Requisition[]>> {
    try {
      const { data, error } = await supabase
        .from('requisitions_drugcupsabot')
        .select(`
          id, status,
          pcus_drugcupsabot(name),
          requisition_periods_drugcupsabot(name),
          requisition_items_drugcupsabot(
            quantity, approved_quantity, price_at_request,
            items_drugcupsabot(name, unit_pack)
          )
        `)
        .eq('period_id', periodId)
        .in('status', statuses)
      
      if (error) throw error
      return { data: data as Requisition[], error: null }
    } catch (err) {
      return handleError<Requisition[]>(err)
    }
  },

  async create(pcuId: number, periodId: number, requesterId: string): Promise<SupabaseResponse<{ id: number }>> {
    try {
      const { data, error } = await supabase
        .from('requisitions_drugcupsabot')
        .insert({
          pcu_id: pcuId,
          period_id: periodId,
          requester_id: requesterId,
          status: 'draft'
        })
        .select('id')
        .single()
      
      if (error) throw error
      return { data, error: null }
    } catch (err) {
      return handleError<{ id: number }>(err)
    }
  },

  async updateStatus(id: number, status: RequisitionStatus, submittedAt?: string | null) {
    try {
      const updateData: { status: RequisitionStatus; submitted_at?: string | null } = { status }
      if (submittedAt !== undefined) {
        updateData.submitted_at = submittedAt
      }
      
      const { error } = await supabase
        .from('requisitions_drugcupsabot')
        .update(updateData)
        .eq('id', id)
      
      if (error) throw error
      return { error: null }
    } catch (err) {
      return handleError(err)
    }
  }
}

// ==========================================
// Requisition Item Services
// ==========================================

export const requisitionItemService = {
  async getByRequisition(requisitionId: number): Promise<SupabaseResponse<RequisitionItem[]>> {
    try {
      const { data, error } = await supabase
        .from('requisition_items_drugcupsabot')
        .select('item_id, quantity, on_hand_quantity')
        .eq('requisition_id', requisitionId)
      
      if (error) throw error
      return { data: data as RequisitionItem[], error: null }
    } catch (err) {
      return handleError<RequisitionItem[]>(err)
    }
  },

  async deleteByRequisition(requisitionId: number) {
    try {
      const { error } = await supabase
        .from('requisition_items_drugcupsabot')
        .delete()
        .eq('requisition_id', requisitionId)
      
      if (error) throw error
      return { error: null }
    } catch (err) {
      return handleError(err)
    }
  },

  async createBatch(items: Omit<RequisitionItem, 'id'>[]) {
    try {
      const { error } = await supabase
        .from('requisition_items_drugcupsabot')
        .insert(items)
      
      if (error) throw error
      return { error: null }
    } catch (err) {
      return handleError(err)
    }
  }
}

// ==========================================
// Period Services
// ==========================================

export const periodService = {
  async getAll(): Promise<SupabaseResponse<RequisitionPeriod[]>> {
    try {
      const { data, error } = await supabase
        .from('requisition_periods_drugcupsabot')
        .select('id, name, start_date, end_date, status')
        .order('start_date', { ascending: false })
      
      if (error) throw error
      return { data: data as RequisitionPeriod[], error: null }
    } catch (err) {
      return handleError<RequisitionPeriod[]>(err)
    }
  },

  async getOpen(): Promise<SupabaseResponse<RequisitionPeriod[]>> {
    try {
      const { data, error } = await supabase
        .from('requisition_periods_drugcupsabot')
        .select('*')
        .eq('status', 'open')
        .order('start_date', { ascending: false })
      
      if (error) throw error
      return { data: data as RequisitionPeriod[], error: null }
    } catch (err) {
      return handleError<RequisitionPeriod[]>(err)
    }
  }
}

// ==========================================
// RPC Functions
// ==========================================

export const rpcService = {
  async getAccountingSummary(periodId: number): Promise<SupabaseResponse<Array<{ pcu_id: number; pcu_name: string; total_value: number }>>> {
    try {
      const { data, error } = await supabase.rpc('get_accounting_summary_by_period', {
        period_id_param: periodId
      })
      
      if (error) throw error
      return { data, error: null }
    } catch (err) {
      return handleError(err)
    }
  },

  async updateApprovedQuantities(items: Array<{ id: number; approved_quantity: number }>) {
    try {
      const { error } = await supabase.rpc('update_approved_quantities', {
        items_data: items
      })
      
      if (error) throw error
      return { error: null }
    } catch (err) {
      return handleError(err)
    }
  }
}
```

---

## Phase 4: Composables (Vue 3 Best Practice)

### Step 4.1: Create src/composables/useAuth.ts

```typescript
/**
 * Authentication composable with full TypeScript support
 * Replaces src/store/auth.js
 */

import { ref, computed } from 'vue'
import { authService, profileService } from '@/services/supabaseService'
import type { UserProfile, User } from '@/types'

const user = ref<User | null>(null)
const profile = ref<UserProfile | null>(null)
const isLoading = ref(false)
const error = ref<Error | null>(null)

export function useAuth() {
  const isLoggedIn = computed(() => !!user.value)
  const isAdmin = computed(() => profile.value?.role === 'admin')
  const userPcuName = computed(() => profile.value?.pcus_drugcupsabot?.name ?? 'N/A')
  const userPcuId = computed(() => profile.value?.pcu_id ?? null)
  const userProfile = computed(() => profile.value)

  const fetchSession = async (): Promise<boolean> => {
    isLoading.value = true
    error.value = null
    
    try {
      const { data } = await authService.getSession()
      
      if (data?.session) {
        user.value = data.session.user as User
        await fetchProfile(user.value.id)
        return true
      }
      return false
    } catch (err) {
      error.value = err instanceof Error ? err : new Error('Session fetch failed')
      return false
    } finally {
      isLoading.value = false
    }
  }

  const fetchProfile = async (userId: string): Promise<void> => {
    const { data, error: profileError } = await profileService.getProfile(userId)
    
    if (profileError) {
      console.error('Error fetching profile:', profileError)
      return
    }
    
    profile.value = data
  }

  const login = async (email: string, password: string): Promise<void> => {
    isLoading.value = true
    error.value = null
    
    try {
      const { data, error: loginError } = await authService.signIn(email, password)
      
      if (loginError) throw loginError
      
      await fetchProfile(data!.user.id)
      
      if (profile.value?.status !== 'approved') {
        const status = profile.value?.status
        await logout()
        
        if (status === 'pending') {
          throw new Error('บัญชีของคุณกำลังรอการอนุมัติจากผู้ดูแลระบบ')
        } else if (status === 'rejected') {
          throw new Error('คำขอลงทะเบียนของคุณถูกปฏิเสธโดยผู้ดูแลระบบ')
        } else {
          throw new Error('บัญชีของคุณยังไม่ถูกเปิดใช้งาน กรุณาติดต่อผู้ดูแล')
        }
      }
      
      user.value = data!.user as User
    } catch (err) {
      error.value = err instanceof Error ? err : new Error('Login failed')
      throw error.value
    } finally {
      isLoading.value = false
    }
  }

  const logout = async (): Promise<void> => {
    try {
      await authService.signOut()
    } catch (e) {
      console.error('Logout error:', e)
    } finally {
      user.value = null
      profile.value = null
    }
  }

  return {
    // State
    user,
    profile,
    isLoading,
    error,
    
    // Computed
    isLoggedIn,
    isAdmin,
    userPcuName,
    userPcuId,
    userProfile,
    
    // Methods
    fetchSession,
    login,
    logout
  }
}
```

### Step 4.2: Create src/composables/useRequisition.ts

```typescript
/**
 * Requisition business logic composable
 */

import { ref, computed } from 'vue'
import {
  requisitionService,
  requisitionItemService,
  itemService
} from '@/services/supabaseService'
import { calculateGrandTotal } from '@/utils/formatters'
import type {
  Requisition,
  RequisitionItem,
  Item,
  RequisitionFormData,
  RequisitionStatus
} from '@/types'

export function useRequisition() {
  const requisition = ref<Requisition | null>(null)
  const items = ref<Item[]>([])
  const formData = ref<RequisitionFormData>({})
  const isLoading = ref(false)
  const error = ref<Error | null>(null)

  const grandTotal = computed(() => {
    if (!requisition.value?.requisition_items_drugcupsabot) return 0
    return calculateGrandTotal(requisition.value.requisition_items_drugcupsabot)
  })

  const initializeFormData = (availableItems: Item[], existingItems: RequisitionItem[] = []) => {
    const data: RequisitionFormData = {}
    
    availableItems.forEach(item => {
      const existing = existingItems.find(ei => ei.item_id === item.id)
      data[item.id] = {
        quantity: existing?.quantity ?? null,
        onHand: existing?.on_hand_quantity ?? null
      }
    })
    
    formData.value = data
  }

  const fetchRequisition = async (id: number): Promise<boolean> => {
    isLoading.value = true
    error.value = null
    
    try {
      const { data, error: fetchError } = await requisitionService.getById(id)
      
      if (fetchError) throw fetchError
      if (!data) throw new Error('Requisition not found')
      
      requisition.value = data
      return true
    } catch (err) {
      error.value = err instanceof Error ? err : new Error('Failed to fetch requisition')
      return false
    } finally {
      isLoading.value = false
    }
  }

  const fetchItems = async (): Promise<boolean> => {
    isLoading.value = true
    
    try {
      const { data, error: fetchError } = await itemService.getActive()
      
      if (fetchError) throw fetchError
      
      items.value = data ?? []
      return true
    } catch (err) {
      error.value = err instanceof Error ? err : new Error('Failed to fetch items')
      return false
    } finally {
      isLoading.value = false
    }
  }

  const saveRequisition = async (
    pcuId: number,
    periodId: number,
    requesterId: string,
    status: RequisitionStatus,
    existingId?: number
  ): Promise<number | null> => {
    isLoading.value = true
    error.value = null
    
    try {
      let requisitionId = existingId

      // Create new if not editing
      if (!existingId) {
        const { data, error: createError } = await requisitionService.create(
          pcuId,
          periodId,
          requesterId
        )
        
        if (createError) throw createError
        requisitionId = data!.id
      }

      // Clear existing items
      await requisitionItemService.deleteByRequisition(requisitionId!)

      // Prepare items to insert
      const itemsToInsert = Object.entries(formData.value)
        .filter(([itemId, entry]) => {
          const itemDetails = items.value.find(i => i.id === Number(itemId))
          return entry.quantity && entry.quantity > 0 && itemDetails?.is_available
        })
        .map(([itemId, entry]) => {
          const itemDetails = items.value.find(i => i.id === Number(itemId))!
          return {
            requisition_id: requisitionId!,
            item_id: Number(itemId),
            quantity: entry.quantity!,
            on_hand_quantity: entry.onHand,
            price_at_request: itemDetails.price_per_unit
          }
        })

      // Insert new items
      if (itemsToInsert.length > 0) {
        const { error: insertError } = await requisitionItemService.createBatch(itemsToInsert)
        if (insertError) throw insertError
      }

      // Update status
      await requisitionService.updateStatus(
        requisitionId!,
        status,
        status === 'submitted' ? new Date().toISOString() : null
      )

      return requisitionId
    } catch (err) {
      error.value = err instanceof Error ? err : new Error('Failed to save requisition')
      return null
    } finally {
      isLoading.value = false
    }
  }

  return {
    // State
    requisition,
    items,
    formData,
    isLoading,
    error,
    
    // Computed
    grandTotal,
    
    // Methods
    initializeFormData,
    fetchRequisition,
    fetchItems,
    saveRequisition
  }
}
```

---

## Phase 5: Component Migration Strategy

### Migration Order (Safest → Riskiest):

1. ✅ **Sidebar.vue** (Pure presentational)
2. ✅ **Navbar.vue** (Simple logic)
3. ✅ **Login.vue** (Form validation test case)
4. ⚠️ **Home.vue** (Uses auth store)
5. ⚠️ **PcuDashboard.vue** (Complex data fetching)
6. 🔴 **RequisitionForm.vue** (Most complex - do last)

### Step 5.1: Example - Sidebar.vue → Sidebar.vue (TypeScript)

Rename `Sidebar.vue` → `Sidebar.vue` (keep name, change content):

```vue
<!-- src/components/Sidebar.vue -->
<template>
  <aside class="sidebar no-print" :class="{ 'is-open': isOpen }">
    <!-- Template unchanged -->
  </aside>
</template>

<script setup lang="ts">
import type { SidebarProps } from '@/types'

const props = defineProps<SidebarProps>()

// Component logic unchanged
</script>

<style scoped>
/* Styles unchanged */
</style>
```

### Step 5.2: Example - Login.vue Migration

```vue
<!-- src/views/Login.vue -->
<template>
  <!-- Template mostly unchanged, only add type-safe bindings -->
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { useRouter } from 'vue-router'
import { useAuth } from '@/composables/useAuth'
import type { LoginForm } from '@/types'

const router = useRouter()
const { login, isLoading, error: authError } = useAuth()

const form = ref<LoginForm>({
  email: '',
  password: ''
})

const errorMessage = ref<string>('')

const handleLogin = async (): Promise<void> => {
  errorMessage.value = ''
  
  try {
    await login(form.value.email, form.value.password)
    router.push('/')
  } catch (error) {
    errorMessage.value = error instanceof Error ? error.message : 'Login failed'
  }
}
</script>
```

---

## Phase 6: Router Migration

### Step 6.1: Create src/router/index.ts

```typescript
import { createRouter, createWebHistory } from 'vue-router'
import { useAuth } from '@/composables/useAuth'
import type { RouteRecordRaw, NavigationGuardNext, RouteLocationNormalized } from 'vue-router'

// Lazy load all routes for better performance
const Login = () => import('@/views/Login.vue')
const Home = () => import('@/views/Home.vue')
const Register = () => import('@/views/Register.vue')
const WaitingForApproval = () => import('@/views/WaitingForApproval.vue')

// PCU Views
const PcuDashboard = () => import('@/views/pcu/PcuDashboard.vue')
const RequisitionForm = () => import('@/views/pcu/RequisitionForm.vue')
const RequisitionDetail = () => import('@/views/pcu/RequisitionDetail.vue')

// Admin Views
const AdminDashboard = () => import('@/views/admin/AdminDashboard.vue')
const AdminRequisitionDetail = () => import('@/views/admin/AdminRequisitionDetail.vue')
// ... other admin views

interface RouteMeta {
  requiresAuth?: boolean
  requiresAdmin?: boolean
  requiresPcu?: boolean
}

const routes: Array<RouteRecordRaw & { meta?: RouteMeta }> = [
  {
    path: '/login',
    name: 'Login',
    component: Login,
  },
  // ... other routes with proper typing
]

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
})

// Navigation guard with proper typing
router.beforeEach(async (
  to: RouteLocationNormalized,
  from: RouteLocationNormalized,
  next: NavigationGuardNext
) => {
  const { fetchSession, isLoggedIn, isAdmin } = useAuth()

  if (to.name === 'WaitingForApproval') {
    return next()
  }

  if (!isLoggedIn.value) {
    await fetchSession()
  }

  const requiresAuth = to.matched.some((record) => record.meta?.requiresAuth)
  const requiresAdmin = to.matched.some((record) => record.meta?.requiresAdmin)
  const requiresPcu = to.matched.some((record) => record.meta?.requiresPcu)

  if (requiresAuth && !isLoggedIn.value) {
    return next({ name: 'Login', query: { redirect: to.fullPath } })
  }

  if (to.name === 'Login' && isLoggedIn.value) {
    return next({ name: 'Home' })
  }

  if (requiresAdmin && !isAdmin.value) {
    console.warn('Access denied: Admin route')
    return next({ name: 'Home' })
  }

  if (requiresPcu && isAdmin.value) {
    console.warn('Access denied: PCU route')
    return next({ name: 'Home' })
  }

  next()
})

export default router
```

---

## Phase 7: Environment Type Safety

### Step 7.1: Create src/env.d.ts

```typescript
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_SUPABASE_URL: string
  readonly VITE_SUPABASE_ANON_KEY: string
  readonly VITE_APP_BASE_URL?: string
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

---

## 🚨 Critical Migration Rules

### DO:
- ✅ Migrate one file at a time
- ✅ Run `npm run type-check` after each file
- ✅ Test functionality in browser before moving to next file
- ✅ Keep `.js` files working alongside `.ts` files during migration
- ✅ Use `any` temporarily if stuck, then refactor later
- ✅ Add `// @ts-expect-error` with explanation for known issues

### DON'T:
- ❌ Rename all files at once
- ❌ Change logic while migrating (only add types)
- ❌ Delete `.js` files until `.ts` replacement is fully tested
- ❌ Skip testing between migrations
- ❌ Use `any` without TODO comment

---

## 📋 Migration Checklist Per File

```
For each file:
[ ] Rename .js/.vue to .ts/.vue (with lang="ts")
[ ] Add type imports
[ ] Add prop/interface definitions
[ ] Add return types to functions
[ ] Replace any with proper types
[ ] Run type-check
[ ] Test in browser
[ ] Commit with message: "migrate: [filename] to TypeScript"
```

---

## 🎯 Success Criteria

The migration is complete when:

1. `npm run type-check` passes with zero errors
2. All `.js` files in `src/` are converted to `.ts`
3. All `.vue` files have `lang="ts"`
4. No `any` types remain (or have documented exceptions)
5. All tests pass (when implemented)
6. Application functions identically to before migration

---

## 🆘 Troubleshooting

### Issue: "Cannot find module '@/types'"
**Solution**: Ensure `tsconfig.json` has correct paths configuration and restart VS Code.

### Issue: "Property does not exist on type"
**Solution**: Check that database types match your Supabase schema exactly.

### Issue: Vue SFC compiler errors
**Solution**: Ensure `vue-tsc` version matches Vue version (3.5.x).

---

**Remember**: This is a marathon, not a sprint. Prioritize correctness over speed. When in doubt, keep the JavaScript version working and add types gradually.