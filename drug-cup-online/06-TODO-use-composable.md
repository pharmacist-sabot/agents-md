# Composables Architecture Guide

This document provides a comprehensive, step-by-step guide for creating production-grade Vue 3 composables (`useRequisition()`, `useItems()`, `useAuth()`) that centralize business logic, eliminate code duplication, and follow industry best practices used by top-tier tech companies.

---

## 🎯 Implementation Strategy: "Extract, Test, Replace"

We will follow a three-phase approach:
1. **Extract**: Create composables alongside existing code
2. **Test**: Verify composables work independently
3. **Replace**: Gradually migrate components to use composables

This ensures zero downtime and easy rollback.

---

## Phase 0: Foundation Architecture

### Step 0.1: Create Composables Directory Structure

```
src/
├── composables/
│   ├── index.ts                 # Public API exports
│   ├── useAuth/
│   │   ├── index.ts             # Main composable
│   │   ├── types.ts             # Auth-specific types
│   │   └── utils.ts             # Auth helpers
│   ├── useRequisition/
│   │   ├── index.ts
│   │   ├── types.ts
│   │   ├── calculations.ts      # Price/quantity math
│   │   └── validation.ts        # Form validation
│   ├── useItems/
│   │   ├── index.ts
│   │   ├── types.ts
│   │   └── filters.ts           # Search/filter logic
│   └── shared/
│       ├── useAsyncState.ts     # Generic async handling
│       ├── usePagination.ts     # List pagination
│       └── useOptimisticUpdate.ts # Optimistic UI
```

### Step 0.2: Create Shared Foundation Composables

Create `src/composables/shared/useAsyncState.ts`:

```typescript
/**
 * Generic async state management
 * Foundation for all data-fetching composables
 */

import { ref, computed, type Ref, type ComputedRef } from 'vue'
import { useLoadingState } from '@/composables/useLoadingState'

export interface AsyncStateOptions<T> {
  /** Initial data value */
  initialData?: T
  
  /** Immediate execution on creation */
  immediate?: boolean
  
  /** Transform raw response */
  transform?: (data: any) => T
  
  /** Default error message */
  defaultError?: string
  
  /** Retry configuration */
  retry?: {
    count: number
    delay: number
  }
  
  /** Cache duration in ms */
  cacheDuration?: number
}

export interface AsyncState<T> {
  // State
  data: Ref<T | undefined>
  error: Ref<Error | null>
  isLoading: ComputedRef<boolean>
  isReady: ComputedRef<boolean>
  isError: ComputedRef<boolean>
  
  // Metadata
  lastUpdated: Ref<number | null>
  executeCount: Ref<number>
  
  // Actions
  execute: (...args: any[]) => Promise<T>
  refresh: () => Promise<T>
  reset: () => void
  invalidate: () => void
}

// Simple in-memory cache
const cache = new Map<string, { data: any; timestamp: number }>()

export function useAsyncState<T>(
  fetchFn: (...args: any[]) => Promise<T>,
  options: AsyncStateOptions<T> = {}
): AsyncState<T> {
  const {
    initialData,
    immediate = false,
    transform,
    defaultError = 'ไม่สามารถโหลดข้อมูลได้',
    retry,
    cacheDuration = 0,
  } = options

  // State
  const data = ref<T | undefined>(initialData) as Ref<T | undefined>
  const error = ref<Error | null>(null)
  const lastUpdated = ref<number | null>(null)
  const executeCount = ref(0)
  const cacheKey = ref<string>('')

  // Loading state with minimum duration for UX
  const loading = useLoadingState({
    minDuration: 200,
    delay: 50,
  })

  // Computed
  const isLoading = computed(() => loading.isLoading.value)
  const isReady = computed(() => !isLoading.value && data.value !== undefined)
  const isError = computed(() => error.value !== null)

  // Check cache
  const getCached = (key: string): T | undefined => {
    if (cacheDuration <= 0) return undefined
    
    const cached = cache.get(key)
    if (!cached) return undefined
    
    const isValid = Date.now() - cached.timestamp < cacheDuration
    if (!isValid) {
      cache.delete(key)
      return undefined
    }
    
    return cached.data
  }

  const setCached = (key: string, value: T): void => {
    if (cacheDuration > 0) {
      cache.set(key, { data: value, timestamp: Date.now() })
    }
  }

  // Execute with full error handling
  const execute = async (...args: any[]): Promise<T> => {
    const key = JSON.stringify(args)
    cacheKey.value = key

    // Check cache first
    const cached = getCached(key)
    if (cached !== undefined) {
      data.value = cached
      return cached
    }

    const op = loading.start('async-execute')
    error.value = null

    try {
      let result = await fetchFn(...args)

      // Transform if needed
      if (transform) {
        result = transform(result)
      }

      // Update state
      data.value = result
      lastUpdated.value = Date.now()
      executeCount.value++

      // Cache result
      setCached(key, result)

      return result
    } catch (err) {
      const normalizedError = err instanceof Error 
        ? err 
        : new Error(defaultError)

      // Retry logic
      if (retry && executeCount.value < retry.count) {
        await new Promise(r => setTimeout(r, retry.delay))
        return execute(...args)
      }

      error.value = normalizedError
      throw normalizedError
    } finally {
      await loading.stop(op.id)
    }
  }

  // Refresh (force re-fetch)
  const refresh = async (): Promise<T> => {
    // Clear cache for this key
    if (cacheKey.value) {
      cache.delete(cacheKey.value)
    }
    return execute()
  }

  // Reset to initial state
  const reset = (): void => {
    data.value = initialData
    error.value = null
    lastUpdated.value = null
    executeCount.value = 0
  }

  // Invalidate cache
  const invalidate = (): void => {
    if (cacheKey.value) {
      cache.delete(cacheKey.value)
    }
  }

  // Auto-execute if immediate
  if (immediate) {
    // Use setTimeout to avoid blocking setup
    setTimeout(() => execute(), 0)
  }

  return {
    data,
    error,
    isLoading,
    isReady,
    isError,
    lastUpdated,
    executeCount,
    execute,
    refresh,
    reset,
    invalidate,
  }
}
```

Create `src/composables/shared/useOptimisticUpdate.ts`:

```typescript
/**
 * Optimistic UI updates with rollback capability
 */

import { ref, type Ref } from 'vue'

export interface OptimisticUpdateOptions<T> {
  /** Commit function (actual API call) */
  commit: () => Promise<T>
  
  /** Rollback function if commit fails */
  rollback?: () => void
  
  /** Success callback */
  onSuccess?: (result: T) => void
  
  /** Error callback */
  onError?: (error: Error) => void
  
  /** Toast messages */
  messages?: {
    success?: string
    error?: string
  }
}

export interface OptimisticUpdateState<T> {
  isPending: Ref<boolean>
  isError: Ref<boolean>
  error: Ref<Error | null>
  
  execute: () => Promise<T>
  rollback: () => void
}

export function useOptimisticUpdate<T>(
  options: OptimisticUpdateOptions<T>
): OptimisticUpdateState<T> {
  const isPending = ref(false)
  const isError = ref(false)
  const error = ref<Error | null>(null)

  let rollbackSnapshot: any = null

  const execute = async (): Promise<T> => {
    isPending.value = true
    isError.value = false
    error.value = null

    try {
      const result = await options.commit()
      
      options.onSuccess?.(result)
      
      if (options.messages?.success) {
        // Toast success
      }
      
      return result
    } catch (err) {
      const normalizedError = err instanceof Error ? err : new Error(String(err))
      
      isError.value = true
      error.value = normalizedError
      
      // Rollback
      if (options.rollback) {
        options.rollback()
      }
      
      options.onError?.(normalizedError)
      
      if (options.messages?.error) {
        // Toast error
      }
      
      throw normalizedError
    } finally {
      isPending.value = false
    }
  }

  const rollback = (): void => {
    if (options.rollback) {
      options.rollback()
    }
  }

  return {
    isPending,
    isError,
    error,
    execute,
    rollback,
  }
}
```

---

## Phase 1: useAuth Composable

### Step 1.1: Create Auth Types

Create `src/composables/useAuth/types.ts`:

```typescript
/**
 * Auth composable type definitions
 */

import type { User as SupabaseUser } from '@supabase/supabase-js'

export type UserRole = 'admin' | 'pcu'
export type UserStatus = 'pending' | 'approved' | 'rejected'

export interface User extends SupabaseUser {
  role: UserRole
  status: UserStatus
}

export interface UserProfile {
  id: string
  username: string
  email: string
  role: UserRole
  status: UserStatus
  pcu_id: number | null
  pcu_name?: string
  created_at?: string
}

export interface PCU {
  id: number
  name: string
}

export interface LoginCredentials {
  email: string
  password: string
}

export interface RegisterData {
  username: string
  email: string
  password: string
  pcu_id: number
}

export interface AuthState {
  user: User | null
  profile: UserProfile | null
  isAuthenticated: boolean
  isAdmin: boolean
  userPcuId: number | null
  userPcuName: string
}

export interface AuthActions {
  login: (credentials: LoginCredentials) => Promise<void>
  register: (data: RegisterData) => Promise<void>
  logout: () => Promise<void>
  fetchSession: () => Promise<boolean>
  refreshProfile: () => Promise<void>
  updatePassword: (newPassword: string) => Promise<void>
  resetPassword: (email: string) => Promise<void>
}

export type AuthErrorCode = 
  | 'INVALID_CREDENTIALS'
  | 'USER_NOT_APPROVED'
  | 'USER_REJECTED'
  | 'EMAIL_EXISTS'
  | 'NETWORK_ERROR'
  | 'SESSION_EXPIRED'
  | 'UNKNOWN_ERROR'

export class AuthError extends Error {
  constructor(
    message: string,
    public code: AuthErrorCode,
    public shouldRedirect?: string
  ) {
    super(message)
    this.name = 'AuthError'
  }
}
```

### Step 1.2: Create useAuth Composable

Create `src/composables/useAuth/index.ts`:

```typescript
/**
 * Authentication composable - Centralized auth state and logic
 * Replaces src/store/auth.js with Vue 3 Composition API best practices
 */

import { ref, computed, type ComputedRef, type Ref } from 'vue'
import { useRouter } from 'vue-router'
import { supabase } from '@/supabaseClient'
import { useAsyncState } from '@/composables/shared/useAsyncState'
import { useToast } from '@/composables/useToast'
import { AuthError } from './types'
import type {
  User,
  UserProfile,
  PCU,
  LoginCredentials,
  RegisterData,
  AuthState,
  AuthActions,
  AuthErrorCode,
} from './types'

// ==========================================
// State (Module-level singleton)
// ==========================================

const user = ref<User | null>(null)
const profile = ref<UserProfile | null>(null)
const isInitialized = ref(false)

// ==========================================
// Composable Factory
// ==========================================

export function useAuth() {
  const router = useRouter()
  const { success: showSuccess, error: showError } = useToast()

  // ==========================================
  // Computed State
  // ==========================================

  const isAuthenticated = computed(() => !!user.value && profile.value?.status === 'approved')
  const isAdmin = computed(() => profile.value?.role === 'admin')
  const isPending = computed(() => profile.value?.status === 'pending')
  const isRejected = computed(() => profile.value?.status === 'rejected')
  
  const userPcuId = computed(() => profile.value?.pcu_id ?? null)
  const userPcuName = computed(() => profile.value?.pcu_name ?? 'ไม่ระบุ')
  
  const displayName = computed(() => profile.value?.username ?? user.value?.email ?? 'ผู้ใช้')
  const userId = computed(() => user.value?.id ?? null)

  // ==========================================
  // Async Operations
  // ==========================================

  const sessionState = useAsyncState<boolean>(
    async () => {
      const { data: { session }, error } = await supabase.auth.getSession()
      
      if (error) throw error
      
      if (session?.user) {
        user.value = session.user as User
        await fetchProfile()
        return true
      }
      
      return false
    },
    {
      immediate: false,
      defaultError: 'ไม่สามารถตรวจสอบเซสชันได้',
    }
  )

  const loginState = useAsyncState<void>(
    async (credentials: LoginCredentials) => {
      const { data, error } = await supabase.auth.signInWithPassword({
        email: credentials.email,
        password: credentials.password,
      })

      if (error) {
        const code = mapAuthError(error)
        throw new AuthError(
          getAuthErrorMessage(code),
          code
        )
      }

      user.value = data.user as User
      await fetchProfile()

      // Check approval status
      if (profile.value?.status === 'pending') {
        await logout()
        throw new AuthError(
          'บัญชีของคุณกำลังรอการอนุมัติจากผู้ดูแลระบบ',
          'USER_NOT_APPROVED',
          '/waiting-for-approval'
        )
      }

      if (profile.value?.status === 'rejected') {
        await logout()
        throw new AuthError(
          'คำขอลงทะเบียนของคุณถูกปฏิเสธโดยผู้ดูแลระบบ',
          'USER_REJECTED'
        )
      }

      showSuccess('เข้าสู่ระบบสำเร็จ', { title: 'ยินดีต้อนรับ' })
    },
    {
      defaultError: 'เข้าสู่ระบบไม่สำเร็จ',
    }
  )

  const registerState = useAsyncState<void>(
    async (data: RegisterData) => {
      const APP_BASE_URL = import.meta.env.VITE_APP_BASE_URL || window.location.origin
      
      const { error } = await supabase.auth.signUp({
        email: data.email,
        password: data.password,
        options: {
          data: {
            username: data.username.trim(),
            pcu_id: data.pcu_id,
            email: data.email.trim(),
          },
          emailRedirectTo: `${APP_BASE_URL}/waiting-for-approval`,
        },
      })

      if (error) {
        if (error.message.includes('User already registered')) {
          throw new AuthError('อีเมลนี้มีการลงทะเบียนแล้ว', 'EMAIL_EXISTS')
        }
        throw error
      }

      showSuccess('ลงทะเบียนสำเร็จ กรุณาตรวจสอบอีเมลเพื่อยืนยัน', {
        title: 'สำเร็จ',
        timeout: 8000,
      })
    },
    {
      defaultError: 'ลงทะเบียนไม่สำเร็จ',
    }
  )

  const logoutState = useAsyncState<void>(
    async () => {
      const { error } = await supabase.auth.signOut()
      if (error) throw error
      
      // Clear all state
      user.value = null
      profile.value = null
      
      showSuccess('ออกจากระบบสำเร็จ', { title: 'ลาก่อน' })
    },
    {
      defaultError: 'ออกจากระบบไม่สำเร็จ',
    }
  )

  // ==========================================
  // Helper Functions
  // ==========================================

  async function fetchProfile(): Promise<void> {
    if (!user.value?.id) return

    const { data, error } = await supabase
      .from('profiles_drugcupsabot')
      .select('*, pcus_drugcupsabot(name)')
      .eq('id', user.value.id)
      .single()

    if (error) throw error

    profile.value = {
      ...data,
      pcu_name: data.pcus_drugcupsabot?.name,
    } as UserProfile
  }

  async function refreshProfile(): Promise<void> {
    await fetchProfile()
  }

  async function fetchPCUList(): Promise<PCU[]> {
    const { data, error } = await supabase
      .from('pcus_drugcupsabot')
      .select('id, name')
      .order('name')

    if (error) throw error
    return data || []
  }

  function mapAuthError(error: any): AuthErrorCode {
    const message = error.message?.toLowerCase() || ''
    
    if (message.includes('invalid')) return 'INVALID_CREDENTIALS'
    if (message.includes('network')) return 'NETWORK_ERROR'
    if (message.includes('expired')) return 'SESSION_EXPIRED'
    
    return 'UNKNOWN_ERROR'
  }

  function getAuthErrorMessage(code: AuthErrorCode): string {
    const messages: Record<AuthErrorCode, string> = {
      INVALID_CREDENTIALS: 'อีเมลหรือรหัสผ่านไม่ถูกต้อง',
      USER_NOT_APPROVED: 'บัญชีรอการอนุมัติ',
      USER_REJECTED: 'บัญชีถูกปฏิเสธ',
      EMAIL_EXISTS: 'อีเมลนี้มีการลงทะเบียนแล้ว',
      NETWORK_ERROR: 'การเชื่อมต่อล้มเหลว',
      SESSION_EXPIRED: 'เซสชันหมดอายุ',
      UNKNOWN_ERROR: 'เกิดข้อผิดพลาดที่ไม่คาดคิด',
    }
    return messages[code]
  }

  // ==========================================
  // Public API
  // ==========================================

  return {
    // State (reactive)
    user: computed(() => user.value),
    profile: computed(() => profile.value),
    
    // Computed getters
    isAuthenticated,
    isAdmin,
    isPending,
    isRejected,
    userPcuId,
    userPcuName,
    displayName,
    userId,
    isInitialized: computed(() => isInitialized.value),
    
    // Loading states
    isLoading: computed(() => 
      sessionState.isLoading.value || 
      loginState.isLoading.value || 
      registerState.isLoading.value ||
      logoutState.isLoading.value
    ),
    
    // Errors
    error: computed(() => 
      sessionState.error.value || 
      loginState.error.value ||
      registerState.error.value ||
      logoutState.error.value
    ),
    
    // Actions
    fetchSession: sessionState.execute,
    login: loginState.execute,
    register: registerState.execute,
    logout: logoutState.execute,
    refreshProfile,
    fetchPCUList,
    
    // Raw states for advanced use
    _states: {
      session: sessionState,
      login: loginState,
      register: registerState,
      logout: logoutState,
    },
  }
}

// ==========================================
// Type Export
// ==========================================

export type UseAuthReturn = ReturnType<typeof useAuth>
```

---

## Phase 2: useItems Composable

### Step 2.1: Create Items Types

Create `src/composables/useItems/types.ts`:

```typescript
/**
 * Items composable type definitions
 */

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
  updated_at?: string
}

export interface ItemFilters {
  search?: string
  category?: string
  isAvailable?: boolean
  isActive?: boolean
  minPrice?: number
  maxPrice?: number
}

export interface ItemSort {
  field: keyof Item
  direction: 'asc' | 'desc'
}

export interface ItemPagination {
  page: number
  perPage: number
  total: number
}

export interface ItemFormData {
  name: string
  price_per_unit: number
  unit_pack: string
  category: string
  is_active: boolean
  is_available: boolean
  notes: string | null
}

export interface ItemsState {
  items: Item[]
  filteredItems: Item[]
  categories: string[]
  
  filters: ItemFilters
  sort: ItemSort
  pagination: ItemPagination
  
  selectedItem: Item | null
  editingItem: Item | null
}

export interface ItemsActions {
  fetchItems: () => Promise<void>
  createItem: (data: ItemFormData) => Promise<Item>
  updateItem: (id: number, data: Partial<ItemFormData>) => Promise<Item>
  deleteItem: (id: number) => Promise<void>
  selectItem: (item: Item | null) => void
  setFilter: (filter: Partial<ItemFilters>) => void
  setSort: (sort: ItemSort) => void
  setPage: (page: number) => void
  resetFilters: () => void
}
```

### Step 2.2: Create useItems Composable

Create `src/composables/useItems/index.ts`:

```typescript
/**
 * Items management composable
 * Centralizes item data, filtering, sorting, and CRUD operations
 */

import { ref, computed, type ComputedRef, type Ref } from 'vue'
import { supabase } from '@/supabaseClient'
import { useAsyncState } from '@/composables/shared/useAsyncState'
import { useOptimisticUpdate } from '@/composables/shared/useOptimisticUpdate'
import { useToast } from '@/composables/useToast'
import type {
  Item,
  ItemFilters,
  ItemSort,
  ItemPagination,
  ItemFormData,
  ItemsState,
  ItemsActions,
} from './types'

// ==========================================
// Helper Functions
// ==========================================

function normalizeItem(raw: any): Item {
  return {
    ...raw,
    price_per_unit: Number(raw.price_per_unit) || 0,
    is_active: raw.is_active ?? true,
    is_available: raw.is_available ?? true,
  }
}

function matchesFilters(item: Item, filters: ItemFilters): boolean {
  if (filters.search) {
    const search = filters.search.toLowerCase()
    if (!item.name.toLowerCase().includes(search)) return false
  }
  
  if (filters.category && item.category !== filters.category) return false
  if (filters.isAvailable !== undefined && item.is_available !== filters.isAvailable) return false
  if (filters.isActive !== undefined && item.is_active !== filters.isActive) return false
  if (filters.minPrice !== undefined && item.price_per_unit < filters.minPrice) return false
  if (filters.maxPrice !== undefined && item.price_per_unit > filters.maxPrice) return false
  
  return true
}

function sortItems(items: Item[], sort: ItemSort): Item[] {
  return [...items].sort((a, b) => {
    const aVal = a[sort.field]
    const bVal = b[sort.field]
    
    if (aVal === null) return sort.direction === 'asc' ? 1 : -1
    if (bVal === null) return sort.direction === 'asc' ? -1 : 1
    
    if (typeof aVal === 'string') {
      const comparison = aVal.localeCompare(bVal as string, 'th')
      return sort.direction === 'asc' ? comparison : -comparison
    }
    
    const comparison = (aVal as number) - (bVal as number)
    return sort.direction === 'asc' ? comparison : -comparison
  })
}

// ==========================================
// Composable
// ==========================================

export function useItems() {
  const { success: showSuccess, error: showError } = useToast()

  // ==========================================
  // State
  // ==========================================

  const items = ref<Item[]>([])
  const selectedItem = ref<Item | null>(null)
  const editingItem = ref<Item | null>(null)

  const filters = ref<ItemFilters>({})
  const sort = ref<ItemSort>({ field: 'category_order', direction: 'asc' })
  const pagination = ref<ItemPagination>({
    page: 1,
    perPage: 50,
    total: 0,
  })

  // ==========================================
  // Async Operations
  // ==========================================

  const fetchState = useAsyncState<Item[]>(
    async () => {
      const { data, error } = await supabase
        .from('items_drugcupsabot')
        .select('*')
        .order('category_order', { ascending: true, nullsFirst: false })
        .order('item_order', { ascending: true, nullsFirst: false })
        .order('name', { ascending: true })

      if (error) throw error
      
      return (data || []).map(normalizeItem)
    },
    {
      immediate: true,
      transform: (data) => {
        items.value = data
        return data
      },
      cacheDuration: 60000, // Cache for 1 minute
    }
  )

  const createState = useAsyncState<Item>(
    async (data: ItemFormData) => {
      // Calculate order values
      const categoryItems = items.value.filter(i => i.category === data.category)
      const maxCategoryOrder = Math.max(
        ...items.value.map(i => i.category_order || 0),
        0
      )
      const maxItemOrder = Math.max(
        ...categoryItems.map(i => i.item_order || 0),
        0
      )

      const isNewCategory = categoryItems.length === 0

      const { data: result, error } = await supabase
        .from('items_drugcupsabot')
        .insert({
          ...data,
          category_order: isNewCategory ? maxCategoryOrder + 1 : categoryItems[0]?.category_order,
          item_order: maxItemOrder + 1,
        })
        .select()
        .single()

      if (error) throw error
      
      const newItem = normalizeItem(result)
      items.value.push(newItem)
      
      showSuccess(`เพิ่มรายการ "${newItem.name}" สำเร็จ`)
      
      return newItem
    },
    {
      defaultError: 'ไม่สามารถเพิ่มรายการได้',
    }
  )

  const updateState = useAsyncState<Item>(
    async ({ id, data }: { id: number; data: Partial<ItemFormData> }) => {
      const { data: result, error } = await supabase
        .from('items_drugcupsabot')
        .update(data)
        .eq('id', id)
        .select()
        .single()

      if (error) throw error
      
      const updatedItem = normalizeItem(result)
      
      // Update local state
      const index = items.value.findIndex(i => i.id === id)
      if (index !== -1) {
        items.value[index] = updatedItem
      }
      
      showSuccess(`อัปเดตรายการ "${updatedItem.name}" สำเร็จ`)
      
      return updatedItem
    },
    {
      defaultError: 'ไม่สามารถอัปเดตรายการได้',
    }
  )

  // ==========================================
  // Computed
  // ==========================================

  const filteredItems = computed(() => {
    let result = items.value
    
    // Apply filters
    if (Object.keys(filters.value).length > 0) {
      result = result.filter(item => matchesFilters(item, filters.value))
    }
    
    // Apply sort
    result = sortItems(result, sort.value)
    
    return result
  })

  const paginatedItems = computed(() => {
    const start = (pagination.value.page - 1) * pagination.value.perPage
    const end = start + pagination.value.perPage
    
    pagination.value.total = filteredItems.value.length
    
    return filteredItems.value.slice(start, end)
  })

  const categories = computed(() => {
    const cats = new Set(items.value.map(i => i.category))
    return Array.from(cats).sort((a, b) => a.localeCompare(b, 'th'))
  })

  const availableItems = computed(() => 
    items.value.filter(i => i.is_available && i.is_active)
  )

  const stats = computed(() => ({
    total: items.value.length,
    active: items.value.filter(i => i.is_active).length,
    available: items.value.filter(i => i.is_available).length,
    byCategory: categories.value.map(cat => ({
      category: cat,
      count: items.value.filter(i => i.category === cat).length,
    })),
  }))

  // ==========================================
  // Actions
  // ==========================================

  const setFilter = (newFilters: Partial<ItemFilters>): void => {
    filters.value = { ...filters.value, ...newFilters }
    pagination.value.page = 1 // Reset to first page
  }

  const setSort = (newSort: ItemSort): void => {
    sort.value = newSort
  }

  const setPage = (page: number): void => {
    pagination.value.page = page
  }

  const resetFilters = (): void => {
    filters.value = {}
    sort.value = { field: 'category_order', direction: 'asc' }
    pagination.value.page = 1
  }

  const selectItem = (item: Item | null): void => {
    selectedItem.value = item
  }

  const startEdit = (item: Item): void => {
    editingItem.value = { ...item }
  }

  const cancelEdit = (): void => {
    editingItem.value = null
  }

  // Optimistic update for quick toggles
  const toggleAvailability = async (item: Item): Promise<void> => {
    const newValue = !item.is_available
    
    // Optimistic update
    const originalValue = item.is_available
    item.is_available = newValue
    
    const { execute, rollback } = useOptimisticUpdate({
      commit: async () => {
        const { error } = await supabase
          .from('items_drugcupsabot')
          .update({ is_available: newValue })
          .eq('id', item.id)
        
        if (error) throw error
      },
      rollback: () => {
        item.is_available = originalValue
      },
      onSuccess: () => {
        showSuccess(
          newValue 
            ? `เปิดใช้งาน "${item.name}" แล้ว` 
            : `ปิดใช้งาน "${item.name}" แล้ว`
        )
      },
      onError: (error) => {
        showError(error.message)
      },
    })
    
    try {
      await execute()
    } catch {
      // Error handled by optimistic update
    }
  }

  // ==========================================
  // Public API
  // ==========================================

  return {
    // State
    items,
    selectedItem,
    editingItem,
    filters,
    sort,
    pagination,
    
    // Computed
    filteredItems,
    paginatedItems,
    categories,
    availableItems,
    stats,
    
    // Loading states
    isLoading: fetchState.isLoading,
    isCreating: createState.isLoading,
    isUpdating: updateState.isLoading,
    
    // Errors
    error: computed(() => 
      fetchState.error.value || 
      createState.error.value || 
      updateState.error.value
    ),
    
    // Actions
    fetchItems: fetchState.execute,
    refresh: fetchState.refresh,
    createItem: createState.execute,
    updateItem: updateState.execute,
    toggleAvailability,
    
    // UI Actions
    selectItem,
    startEdit,
    cancelEdit,
    setFilter,
    setSort,
    setPage,
    resetFilters,
    
    // Direct access for advanced use
    _raw: {
      fetch: fetchState,
      create: createState,
      update: updateState,
    },
  }
}

export type UseItemsReturn = ReturnType<typeof useItems>
```

---

## Phase 3: useRequisition Composable

### Step 3.1: Create Requisition Types

Create `src/composables/useRequisition/types.ts`:

```typescript
/**
 * Requisition composable type definitions
 */

import type { Item } from '@/composables/useItems/types'

export type RequisitionStatus = 'draft' | 'submitted' | 'approved' | 'fulfilled' | 'rejected'

export interface RequisitionPeriod {
  id: number
  name: string
  start_date: string
  end_date: string
  status: 'open' | 'closed'
}

export interface RequisitionItem {
  id?: number
  item_id: number
  quantity: number
  approved_quantity: number | null
  on_hand_quantity: number | null
  price_at_request: number
  
  // Joined data
  item?: Item
  item_name?: string
  item_unit?: string
}

export interface Requisition {
  id: number
  pcu_id: number
  period_id: number
  requester_id: string
  status: RequisitionStatus
  submitted_at: string | null
  created_at: string
  
  // Joined data
  period?: RequisitionPeriod
  pcu_name?: string
  items: RequisitionItem[]
  
  // Calculated
  total_value?: number
  total_items?: number
}

export interface RequisitionFormData {
  [itemId: number]: {
    quantity: number | null
    onHand: number | null
  }
}

export interface RequisitionFilters {
  status?: RequisitionStatus[]
  periodId?: number
  pcuId?: number
  startDate?: string
  endDate?: string
}

export interface RequisitionSummary {
  totalRequests: number
  totalValue: number
  byStatus: Record<RequisitionStatus, number>
  byPeriod: Record<number, { name: string; count: number; value: number }>
}

export interface RequisitionCalculations {
  itemValue: (item: RequisitionItem) => number
  grandTotal: (items: RequisitionItem[]) => number
  itemCount: (items: RequisitionItem[]) => number
  canSubmit: (items: RequisitionItem[]) => boolean
  canApprove: (requisition: Requisition) => boolean
  canFulfill: (requisition: Requisition) => boolean
}
```

### Step 3.2: Create Calculation Utilities

Create `src/composables/useRequisition/calculations.ts`:

```typescript
/**
 * Requisition calculation utilities
 * Pure functions for price/quantity calculations
 */

import type { RequisitionItem, Requisition, RequisitionCalculations } from './types'

export const calculations: RequisitionCalculations = {
  itemValue(item: RequisitionItem): number {
    const quantity = item.approved_quantity ?? item.quantity
    return quantity * item.price_at_request
  },

  grandTotal(items: RequisitionItem[]): number {
    return items.reduce((sum, item) => sum + this.itemValue(item), 0)
  },

  itemCount(items: RequisitionItem[]): number {
    return items.filter(item => (item.quantity || 0) > 0).length
  },

  canSubmit(items: RequisitionItem[]): boolean {
    return items.some(item => (item.quantity || 0) > 0)
  },

  canApprove(requisition: Requisition): boolean {
    return requisition.status === 'submitted'
  },

  canFulfill(requisition: Requisition): boolean {
    return requisition.status === 'approved'
  },
}

// Form-specific calculations
export function calculateFormValue(
  formData: { quantity: number | null },
  itemPrice: number
): number {
  return (formData.quantity || 0) * itemPrice
}

export function calculateFormTotal(
  formData: Record<number, { quantity: number | null }>,
  items: Array<{ id: number; price_per_unit: number }>
): number {
  return items.reduce((sum, item) => {
    const quantity = formData[item.id]?.quantity || 0
    return sum + quantity * item.price_per_unit
  }, 0)
}

// Validation helpers
export function validateQuantities(
  items: RequisitionItem[]
): { valid: boolean; errors: string[] } {
  const errors: string[] = []
  
  items.forEach(item => {
    if (item.quantity !== null && item.quantity < 0) {
      errors.push(`จำนวนเบิกต้องไม่ติดลบ`)
    }
    if (item.on_hand_quantity !== null && item.on_hand_quantity < 0) {
      errors.push(`ยอดคงเหลือต้องไม่ติดลบ`)
    }
  })
  
  return { valid: errors.length === 0, errors }
}
```

### Step 3.3: Create useRequisition Composable

Create `src/composables/useRequisition/index.ts`:

```typescript
/**
 * Requisition management composable
 * Complete lifecycle: create, edit, submit, approve, fulfill
 */

import { ref, computed, type ComputedRef, type Ref } from 'vue'
import { supabase } from '@/supabaseClient'
import { useAsyncState } from '@/composables/shared/useAsyncState'
import { useOptimisticUpdate } from '@/composables/shared/useOptimisticUpdate'
import { useToast } from '@/composables/useToast'
import { calculations, calculateFormTotal } from './calculations'
import type {
  Requisition,
  RequisitionPeriod,
  RequisitionItem,
  RequisitionFormData,
  RequisitionFilters,
  RequisitionStatus,
} from './types'
import type { Item } from '@/composables/useItems/types'

// ==========================================
// Helper Functions
// ==========================================

function normalizeRequisition(raw: any): Requisition {
  return {
    ...raw,
    items: raw.requisition_items_drugcupsabot?.map((ri: any) => ({
      id: ri.id,
      item_id: ri.item_id,
      quantity: ri.quantity,
      approved_quantity: ri.approved_quantity,
      on_hand_quantity: ri.on_hand_quantity,
      price_at_request: Number(ri.price_at_request),
      item: ri.items_drugcupsabot,
      item_name: ri.items_drugcupsabot?.name,
      item_unit: ri.items_drugcupsabot?.unit_pack,
    })) || [],
    period: raw.requisition_periods_drugcupsabot,
    pcu_name: raw.pcus_drugcupsabot?.name,
    total_value: 0, // Calculated
    total_items: 0, // Calculated
  }
}

function calculateRequisitionTotals(req: Requisition): Requisition {
  return {
    ...req,
    total_value: calculations.grandTotal(req.items),
    total_items: calculations.itemCount(req.items),
  }
}

// ==========================================
// Composable
// ==========================================

export function useRequisition() {
  const { success: showSuccess, error: showError } = useToast()

  // ==========================================
  // State
  // ==========================================

  const requisitions = ref<Requisition[]>([])
  const currentRequisition = ref<Requisition | null>(null)
  const currentPeriod = ref<RequisitionPeriod | null>(null)
  
  const formData = ref<RequisitionFormData>({})
  const availableItems = ref<Item[]>([])

  // ==========================================
  // Async Operations
  // ==========================================

  // Fetch periods
  const periodsState = useAsyncState<RequisitionPeriod[]>(
    async () => {
      const { data, error } = await supabase
        .from('requisition_periods_drugcupsabot')
        .select('*')
        .order('start_date', { ascending: false })

      if (error) throw error
      return data || []
    },
    {
      cacheDuration: 300000, // 5 minutes
    }
  )

  // Fetch user's requisitions
  const fetchUserRequisitions = useAsyncState<Requisition[]>(
    async (pcuId: number) => {
      const { data, error } = await supabase
        .from('requisitions_drugcupsabot')
        .select(`
          *,
          requisition_periods_drugcupsabot(*),
          pcus_drugcupsabot(name),
          requisition_items_drugcupsabot(
            *,
            items_drugcupsabot(*)
          )
        `)
        .eq('pcu_id', pcuId)
        .order('created_at', { ascending: false })

      if (error) throw error
      
      const normalized = (data || []).map(normalizeRequisition).map(calculateRequisitionTotals)
      requisitions.value = normalized
      return normalized
    },
    {
      defaultError: 'ไม่สามารถโหลดประวัติการเบิกได้',
    }
  )

  // Fetch single requisition
  const fetchRequisition = useAsyncState<Requisition>(
    async (id: number) => {
      const { data, error } = await supabase
        .from('requisitions_drugcupsabot')
        .select(`
          *,
          requisition_periods_drugcupsabot(*),
          pcus_drugcupsabot(name),
          requisition_items_drugcupsabot(
            *,
            items_drugcupsabot(*)
          )
        `)
        .eq('id', id)
        .single()

      if (error) throw error
      
      const normalized = calculateRequisitionTotals(normalizeRequisition(data))
      currentRequisition.value = normalized
      return normalized
    },
    {
      defaultError: 'ไม่สามารถโหลดข้อมูลใบเบิกได้',
    }
  )

  // Create new requisition
  const createRequisition = useAsyncState<Requisition>(
    async ({ pcuId, periodId }: { pcuId: number; periodId: number }) => {
      const { data: { user } } = await supabase.auth.getUser()
      
      const { data, error } = await supabase
        .from('requisitions_drugcupsabot')
        .insert({
          pcu_id: pcuId,
          period_id: periodId,
          requester_id: user?.id,
          status: 'draft',
        })
        .select()
        .single()

      if (error) throw error
      
      return normalizeRequisition(data)
    },
    {
      defaultError: 'ไม่สามารถสร้างใบเบิกได้',
    }
  )

  // Save requisition (create or update)
  const saveRequisition = useAsyncState<void>(
    async (params: {
      requisitionId?: number
      pcuId: number
      periodId: number
      items: Array<{
        item_id: number
        quantity: number
        on_hand_quantity: number | null
        price_at_request: number
      }>
      status: RequisitionStatus
    }) => {
      let reqId = params.requisitionId

      // Create if new
      if (!reqId) {
        const newReq = await createRequisition.execute({
          pcuId: params.pcuId,
          periodId: params.periodId,
        })
        reqId = newReq.id
      }

      // Delete existing items
      await supabase
        .from('requisition_items_drugcupsabot')
        .delete()
        .eq('requisition_id', reqId)

      // Insert new items
      if (params.items.length > 0) {
        const { error } = await supabase
          .from('requisition_items_drugcupsabot')
          .insert(
            params.items.map(item => ({
              requisition_id: reqId,
              ...item,
            }))
          )

        if (error) throw error
      }

      // Update status
      const { error: statusError } = await supabase
        .from('requisitions_drugcupsabot')
        .update({
          status: params.status,
          submitted_at: params.status === 'submitted' ? new Date().toISOString() : null,
        })
        .eq('id', reqId!)

      if (statusError) throw statusError

      // Success message
      const message = params.status === 'submitted'
        ? 'ส่งใบเบิกเรียบร้อยแล้ว'
        : 'บันทึกฉบับร่างเรียบร้อยแล้ว'
      
      showSuccess(message, { title: 'สำเร็จ' })
    },
    {
      defaultError: 'ไม่สามารถบันทึกใบเบิกได้',
    }
  )

  // Admin: Approve requisition
  const approveRequisition = useAsyncState<void>(
    async (params: {
      requisitionId: number
      items: Array<{ id: number; approved_quantity: number }>
    }) => {
      // Update approved quantities
      const { error: rpcError } = await supabase.rpc('update_approved_quantities', {
        items_data: params.items,
      })

      if (rpcError) throw rpcError

      // Update status
      const { error } = await supabase
        .from('requisitions_drugcupsabot')
        .update({ status: 'approved' })
        .eq('id', params.requisitionId)

      if (error) throw error

      showSuccess('อนุมัติใบเบิกเรียบร้อยแล้ว', { title: 'สำเร็จ' })
    },
    {
      defaultError: 'ไม่สามารถอนุมัติใบเบิกได้',
    }
  )

  // Admin: Fulfill requisition
  const fulfillRequisition = useAsyncState<void>(
    async (requisitionId: number) => {
      const { error } = await supabase
        .from('requisitions_drugcupsabot')
        .update({ status: 'fulfilled' })
        .eq('id', requisitionId)

      if (error) throw error

      showSuccess('ยืนยันการจ่ายเวชภัณฑ์เรียบร้อยแล้ว', { title: 'สำเร็จ' })
    },
    {
      defaultError: 'ไม่สามารถยืนยันการจ่ายได้',
    }
  )

  // ==========================================
  // Form Management
  // ==========================================

  const initializeForm = (items: Item[], existingItems?: RequisitionItem[]) => {
    availableItems.value = items
    
    const initial: RequisitionFormData = {}
    items.forEach(item => {
      initial[item.id] = {
        quantity: null,
        onHand: null,
      }
    })

    // Populate existing data
    if (existingItems) {
      existingItems.forEach(item => {
        if (initial[item.item_id]) {
          initial[item.item_id] = {
            quantity: item.quantity,
            onHand: item.on_hand_quantity,
          }
        }
      })
    }

    formData.value = initial
  }

  const updateFormItem = (
    itemId: number,
    field: 'quantity' | 'onHand',
    value: number | null
  ) => {
    if (!formData.value[itemId]) {
      formData.value[itemId] = { quantity: null, onHand: null }
    }
    formData.value[itemId][field] = value
  }

  const getFormItemsForSubmit = (): Array<{
    item_id: number
    quantity: number
    on_hand_quantity: number | null
    price_at_request: number
  }> => {
    return Object.entries(formData.value)
      .filter(([_, data]) => data.quantity && data.quantity > 0)
      .map(([itemId, data]) => {
        const item = availableItems.value.find(i => i.id === Number(itemId))
        return {
          item_id: Number(itemId),
          quantity: data.quantity!,
          on_hand_quantity: data.onHand,
          price_at_request: item?.price_per_unit || 0,
        }
      })
  }

  const validateForm = (): { valid: boolean; errors: string[] } => {
    const errors: string[] = []
    const items = getFormItemsForSubmit()

    if (items.length === 0) {
      errors.push('ต้องมีอย่างน้อย 1 รายการที่มีจำนวนเบิกมากกว่า 0')
    }

    items.forEach(item => {
      if (item.quantity < 0) errors.push('จำนวนเบิกต้องไม่ติดลบ')
      if (item.on_hand_quantity !== null && item.on_hand_quantity < 0) {
        errors.push('ยอดคงเหลือต้องไม่ติดลบ')
      }
    })

    return { valid: errors.length === 0, errors }
  }

  const clearForm = () => {
    formData.value = {}
    availableItems.value = []
  }

  // ==========================================
  // Computed
  // ==========================================

  const formTotal = computed(() => 
    calculateFormTotal(formData.value, availableItems.value)
  )

  const formItemCount = computed(() => 
    Object.values(formData.value).filter(d => d.quantity && d.quantity > 0).length
  )

  const canSubmitForm = computed(() => {
    const { valid } = validateForm()
    return valid && !saveRequisition.isLoading.value
  })

  const draftRequisitions = computed(() => 
    requisitions.value.filter(r => r.status === 'draft')
  )

  const submittedRequisitions = computed(() => 
    requisitions.value.filter(r => r.status === 'submitted')
  )

  const approvedRequisitions = computed(() => 
    requisitions.value.filter(r => r.status === 'approved')
  )

  // ==========================================
  // Public API
  // ==========================================

  return {
    // State
    requisitions,
    currentRequisition,
    currentPeriod,
    formData,
    availableItems,

    // Computed
    periods: periodsState.data,
    isLoadingPeriods: periodsState.isLoading,
    
    formTotal,
    formItemCount,
    canSubmitForm,
    
    draftRequisitions,
    submittedRequisitions,
    approvedRequisitions,

    // Async actions
    fetchPeriods: periodsState.execute,
    fetchUserRequisitions: fetchUserRequisitions.execute,
    fetchRequisition: fetchRequisition.execute,
    createRequisition: createRequisition.execute,
    saveRequisition: saveRequisition.execute,
    approveRequisition: approveRequisition.execute,
    fulfillRequisition: fulfillRequisition.execute,

    // Form management
    initializeForm,
    updateFormItem,
    getFormItemsForSubmit,
    validateForm,
    clearForm,

    // Utilities
    calculations,
    
    // Direct access
    _raw: {
      periods: periodsState,
      fetch: fetchRequisition,
      save: saveRequisition,
    },
  }
}

export type UseRequisitionReturn = ReturnType<typeof useRequisition>
```

---

## Phase 4: Public API & Migration

### Step 4.1: Create Public Exports

Update `src/composables/index.ts`:

```typescript
/**
 * Composables Public API
 * Centralized exports for clean imports
 */

// Auth
export { useAuth } from './useAuth'
export type { UseAuthReturn } from './useAuth'
export type {
  User,
  UserProfile,
  LoginCredentials,
  RegisterData,
} from './useAuth/types'

// Items
export { useItems } from './useItems'
export type { UseItemsReturn } from './useItems'
export type {
  Item,
  ItemFilters,
  ItemFormData,
} from './useItems/types'

// Requisition
export { useRequisition } from './useRequisition'
export type { UseRequisitionReturn } from './useRequisition'
export type {
  Requisition,
  RequisitionPeriod,
  RequisitionItem,
  RequisitionStatus,
} from './useRequisition/types'

// Shared utilities
export { useAsyncState } from './shared/useAsyncState'
export { useOptimisticUpdate } from './shared/useOptimisticUpdate'

// Re-export commonly used composables
export { useToast } from './useToast'
export { useLoadingState, useSubmitLoading, useFetchLoading } from './useLoadingState'
export { useFormValidation } from './useValidation'
export { useFormSubmit } from './useFormSubmit'
export { useThrottledFunction, useDebouncedInput } from './utils/rateLimit'
```

### Step 4.2: Migration Example - PcuDashboard.vue

**Before (using store + manual fetch):**
```javascript
// Old approach
import { useAuthStore } from '@/store/auth'
const auth = useAuthStore()
const requisitions = ref([])
// ... manual fetch logic
```

**After (using composables):**
```vue
<!-- src/views/pcu/PcuDashboard.vue -->
<template>
  <div class="container">
    <ErrorBoundary title="ไม่สามารถโหลดแดชบอร์ดได้">
      <div v-if="isLoading" class="loading">กำลังโหลด...</div>
      
      <template v-else>
        <h2>หน้าหลัก {{ userPcuName }}</h2>
        
        <!-- Open Periods -->
        <section class="periods-section">
          <h3>รอบการเบิกปัจจุบัน</h3>
          <PeriodList
            :periods="openPeriods"
            :requisitions="draftRequisitions"
            @create="createNewRequisition"
            @edit="editRequisition"
          />
        </section>
        
        <!-- History -->
        <section class="history-section">
          <h3>ประวัติการเบิก</h3>
          <RequisitionHistory :requisitions="submittedRequisitions" />
        </section>
      </template>
    </ErrorBoundary>
  </div>
</template>

<script setup lang="ts">
import { onMounted, computed } from 'vue'
import { useRouter } from 'vue-router'
import { ErrorBoundary } from '@/components/ErrorBoundary'
import { useAuth, useRequisition } from '@/composables'
import PeriodList from '@/components/PeriodList.vue'
import RequisitionHistory from '@/components/RequisitionHistory.vue'

const router = useRouter()

// Auth composable replaces store
const { 
  userPcuId, 
  userPcuName, 
  isAuthenticated 
} = useAuth()

// Requisition composable replaces manual fetch
const {
  periods,
  draftRequisitions,
  submittedRequisitions,
  fetchPeriods,
  fetchUserRequisitions,
  createRequisition,
  isLoadingPeriods,
  isLoading: isLoadingRequisitions,
} = useRequisition()

const isLoading = computed(() => 
  isLoadingPeriods.value || isLoadingRequisitions.value
)

const openPeriods = computed(() => 
  periods.value?.filter(p => p.status === 'open') || []
)

onMounted(async () => {
  if (!userPcuId.value) return
  
  await Promise.all([
    fetchPeriods(),
    fetchUserRequisitions(userPcuId.value),
  ])
})

const createNewRequisition = async (periodId: number) => {
  const requisition = await createRequisition({
    pcuId: userPcuId.value!,
    periodId,
  })
  
  router.push(`/pcu/requisition/${periodId}/${requisition.id}`)
}

const editRequisition = (requisition: any) => {
  router.push(`/pcu/requisition/${requisition.period_id}/${requisition.id}`)
}
</script>
```

---

## 🚨 Critical Implementation Rules

### DO:
- ✅ Keep composables focused (single responsibility)
- ✅ Use `useAsyncState` for all data fetching
- ✅ Export types for TypeScript consumers
- ✅ Handle loading and error states consistently
- ✅ Use optimistic updates for better UX
- ✅ Cache data appropriately
- ✅ Clean up on unmount (cancel requests)

### DON'T:
- ❌ Put UI logic in composables
- ❌ Mutate props or external state directly
- ❌ Forget to handle cleanup
- ❌ Mix concerns (auth logic in items composable)
- ❌ Expose internal state unnecessarily
- ❌ Skip error handling

---

## 📋 Migration Checklist

```
For each composable:
[ ] Create directory with types.ts
[ ] Implement core logic with useAsyncState
[ ] Add optimistic updates where beneficial
[ ] Export clean public API
[ ] Write JSDoc comments
[ ] Test independently
[ ] Migrate one component as proof of concept
[ ] Gradually migrate remaining components
[ ] Delete old store/modules once migrated
```

---

## 🎯 Success Criteria

Composables implementation is complete when:

1. Zero `store/` usage in components (fully migrated)
2. All data fetching uses `useAsyncState`
3. Consistent loading/error patterns across app
4. Type-safe composable APIs
5. Optimistic updates for quick actions
6. Proper cleanup on component unmount
7. 50%+ reduction in component code complexity

---

## 🆘 Troubleshooting

### Issue: "Composable state not shared between components"
**Solution**: Ensure state is defined at module level (outside function), not inside composable.

### Issue: "Multiple fetch requests on mount"
**Solution**: Use `immediate: false` and `useAsyncState` cache, or deduplicate with shared promise.

### Issue: "Type inference not working"
**Solution**: Export explicit types and use `ReturnType<typeof useX>` for complex return types.

### Issue: "Memory leak on rapid navigation"
**Solution**: Ensure all async operations check `isMounted` or use abortable promises.

---

**Remember**: Composables are the backbone of Vue 3's Composition API. Invest time in getting the architecture right, and your components will become simple, declarative, and maintainable.