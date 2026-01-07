# Forms API v12 Specification

## Overview

The v12 design builds on v11's simplifications while addressing practical gaps around nested data, form-level validation, and widget ergonomics. It maintains the schema-driven approach with React Hook Form as the runtime, but provides better tooling for complex field structures and a more complete widget contract.

## Key Changes from v11

1. **Hybrid field key strategy**: Top-level keys are primary, but nested paths are supported where needed (validation errors, effects).
2. **Form-level validation** restored to schema.
3. **Partial field configs**: Not all `TValues` keys require config entries.
4. **Expanded widget props**: `error`, `disabled`, `onBlur`, `touched`, `dirty` included.
5. **Widget utilities**: First-class hooks for arrays and nested objects.
6. **Removed redundant `name`** from `FieldConfig` (derived from map key).

## Design Goals

- Enforce typed field names with escape hatches only where necessary.
- Make async behavior explicit without leaking into widgets.
- Support field dependencies with clear data flow.
- Provide complete widget contract including error/disabled state.
- Supply utilities so nested structures don't require reinvention.
- Keep layout pluggable and form logic isolated.

---

## TypeScript API

### Core Types

```ts
import {
  FieldErrors,
  FieldPath,
  FieldValues,
  PathValue,
  UseFormReturn,
} from 'react-hook-form';

// Primary: top-level keys for schema definition
export type FieldKey<TValues extends FieldValues> = keyof TValues & string;

// Secondary: full paths for errors, effects, and advanced use cases
export type FieldPath<TValues extends FieldValues> = FieldPath<TValues>;

export type DefaultValuesSync<TValues extends FieldValues> =
  | Partial<TValues>
  | ((ctx: DefaultValuesContext) => Partial<TValues>);

export type DefaultValuesAsync<TValues extends FieldValues> = (
  ctx: DefaultValuesContext
) => Promise<Partial<TValues>>;

export type DefaultValuesConfig<TValues extends FieldValues> =
  | DefaultValuesSync<TValues>
  | DefaultValuesAsync<TValues>;

export interface DefaultValuesContext {
  /** Initial data passed to the form (e.g., for edit mode) */
  initial?: unknown;
  /** URL params or other context */
  params?: Record<string, string>;
}
```

### Form Schema

```ts
export interface FormSchema<TValues extends FieldValues> {
  /** Unique identifier for caching and debugging */
  id: string;
  
  /** Field configurations (partial â not all TValues keys required) */
  fields: FieldsConfig<TValues>;
  
  /** Initial values â sync or async (can load drafts here) */
  defaultValues?: DefaultValuesConfig<TValues>;
  
  /** Form-level validation (cross-field rules) */
  validate?: FormValidator<TValues>;
  
  /** When to run validation initially */
  validationMode?: ValidationMode; // default: 'onSubmit'
  
  /** When to re-validate after first submission */
  revalidationMode?: RevalidationMode; // default: 'onChange'
  
  /** Transform values after validation, before submit */
  transformBeforeSubmit?: (values: TValues) => TValues | Promise<TValues>;
}

export type ValidationMode = 'onSubmit' | 'onChange' | 'onBlur' | 'onTouched';
export type RevalidationMode = 'onChange' | 'onBlur';
```

### Fields Configuration

Fields are declared as a **partial** typed map. Only fields that need configuration are included; others can be managed programmatically or via effects.

```ts
export type FieldsConfig<TValues extends FieldValues> = {
  [K in FieldKey<TValues>]?: FieldConfig<TValues, K>;
};

export interface FieldConfig<
  TValues extends FieldValues,
  K extends FieldKey<TValues>
> {
  /** Display label */
  label?: string;
  
  /** Help text shown below field */
  description?: string;
  
  /** Widget component to render */
  widget: WidgetComponent<TValues, K>;
  
  /** Static or sync-computed widget props */
  props?: WidgetProps<TValues>;
  
  /** Async widget props (e.g., fetching options) */
  asyncProps?: AsyncWidgetProps<TValues>;
  
  /** Fields that trigger re-evaluation of conditions, asyncProps, and deriveValue */
  dependsOn?: FieldKey<TValues>[];
  
  /** Derive this field's value from other fields (runs on init + dependency changes) */
  deriveValue?: (values: TValues) => PathValue<TValues, K> | Promise<PathValue<TValues, K>>;
  
  /** Cache key for async deriveValue to prevent duplicate fetches */
  deriveValueCacheKey?: (values: TValues) => string;
  
  /** Visibility condition */
  visible?: boolean | ConditionFn<TValues>;
  
  /** Disabled condition */
  disabled?: boolean | ConditionFn<TValues>;
  
  /** Required condition (also used in validation) */
  required?: boolean | ConditionFn<TValues>;
  
  /** Field-level validation */
  validate?: FieldValidator<TValues, K>;
  
  /** Side effect when this field changes (use for resets, not derivations) */
  onChange?: FieldEffect<TValues, K>;
  
  /** Placeholder for the field */
  placeholder?: string;
}
```

### Widget System

#### Widget Component Type

```ts
export type WidgetComponent<
  TValues extends FieldValues,
  K extends FieldKey<TValues>
> = React.ComponentType<WidgetRenderProps<TValues, K>>;
```

#### Widget Props (What Widgets Receive)

```ts
export interface WidgetRenderProps<
  TValues extends FieldValues,
  K extends FieldKey<TValues>
> {
  /** Field name (top-level key) */
  name: K;
  
  /** Display label */
  label?: string;
  
  /** Help text */
  description?: string;
  
  /** Placeholder text */
  placeholder?: string;
  
  /** Current value */
  value: PathValue<TValues, K>;
  
  /** Update handler */
  onChange: (value: PathValue<TValues, K>) => void;
  
  /** Blur handler (for onBlur validation) */
  onBlur: () => void;
  
  /** Whether field is required */
  required: boolean;
  
  /** Whether field is disabled */
  disabled: boolean;
  
  /** Validation error message (if any) */
  error?: string;
  
  /** Whether field has been touched (blurred at least once) */
  touched: boolean;
  
  /** Whether value differs from default */
  dirty: boolean;
  
  /** Extra props from `props` or `asyncProps` config */
  extra: Record<string, unknown>;
  
  /** For nested error handling â see Widget Utilities section */
  fieldErrors?: Record<string, string>;
  
  /** Set error on a nested path (for complex widgets) */
  setNestedError?: (path: string, message: string | null) => void;
}
```

#### Widget Props Configuration

```ts
export type WidgetProps<TValues extends FieldValues> =
  | Record<string, unknown>
  | WidgetPropsFactory<TValues>;

export type WidgetPropsFactory<TValues extends FieldValues> = (ctx: {
  values: TValues;
}) => Record<string, unknown>;

export interface AsyncWidgetProps<TValues extends FieldValues> {
  /** Async factory function */
  factory: (ctx: { values: TValues }) => Promise<Record<string, unknown>>;
  
  /** Cache key for memoization; if omitted, refetches on every dependency change */
  cacheKey?: string | ((values: TValues) => string);
  
  /** Fallback props while loading */
  fallback?: Record<string, unknown>;
}
```

### Conditions and Dependencies

```ts
export type ConditionFn<TValues extends FieldValues> = (
  values: TValues
) => boolean;
```

### Validation

#### Field-Level Validation

```ts
export type ValidationResult = string | null;
export type AsyncValidationResult = Promise<string | null>;

/** Declarative validation rules */
export interface ValidationRules<
  TValues extends FieldValues,
  K extends FieldKey<TValues>
> {
  required?: boolean | ConditionFn<TValues>;
  min?: number;
  max?: number;
  minLength?: number;
  maxLength?: number;
  pattern?: RegExp | { value: RegExp; message: string };
  custom?: (
    value: PathValue<TValues, K>,
    values: TValues
  ) => ValidationResult | AsyncValidationResult;
}

/** Validator can be rules object or function */
export type FieldValidator<
  TValues extends FieldValues,
  K extends FieldKey<TValues>
> =
  | ValidationRules<TValues, K>
  | ((
      value: PathValue<TValues, K>,
      values: TValues
    ) => ValidationResult | AsyncValidationResult);
```

#### Form-Level Validation

```ts
export type FormValidatorResult<TValues extends FieldValues> = {
  /** Field-specific errors (supports nested paths) */
  fieldErrors?: Partial<Record<FieldPath<TValues> | string, string>>;
  /** Form-level error (not tied to specific field) */
  formError?: string;
} | null;

export type FormValidator<TValues extends FieldValues> = (
  values: TValues
) => FormValidatorResult<TValues> | Promise<FormValidatorResult<TValues>>;
```

### Effects

Effects run when watched fields change and can update other fields.

```ts
export interface EffectApi<TValues extends FieldValues> {
  /** Get all current values */
  getValues: () => TValues;
  
  /** Get single field value */
  getValue: <K extends FieldKey<TValues>>(name: K) => PathValue<TValues, K>;
  
  /** Set field value */
  setValue: <K extends FieldKey<TValues>>(
    name: K,
    value: PathValue<TValues, K>,
    options?: { 
      shouldValidate?: boolean;
      shouldDirty?: boolean;
      shouldTouch?: boolean;
    }
  ) => void;
  
  /** Reset field to default */
  resetField: (name: FieldKey<TValues>) => void;
  
  /** Set error on any field (including nested paths) */
  setError: (name: string, message: string) => void;
  
  /** Clear error on field */
  clearError: (name: string) => void;
}

export type FieldEffect<
  TValues extends FieldValues,
  K extends FieldKey<TValues>
> = (ctx: {
  /** Field that changed */
  name: K;
  /** New value */
  value: PathValue<TValues, K>;
  /** Previous value */
  previousValue: PathValue<TValues, K>;
  /** All current values */
  values: TValues;
  /** API for updates */
  api: EffectApi<TValues>;
}) => void | Promise<void>;
```

### SchemaForm Component

```ts
export interface SchemaFormProps<TValues extends FieldValues> {
  /** Form configuration */
  schema: FormSchema<TValues>;
  
  /** Layout component */
  layout: LayoutComponent<TValues>;
  
  /** Submit handler */
  onSubmit: (values: TValues) => Promise<SubmitResult<TValues> | void>;
  
  /** Initial data for edit mode */
  initialData?: unknown;
  
  /** Additional context for defaultValues */
  params?: Record<string, string>;
  
  /** Called when form becomes dirty/clean */
  onDirtyChange?: (isDirty: boolean) => void;
  
  /** Called when values change (for drafts, analytics, etc.) */
  onValuesChange?: (
    values: TValues,
    meta: {
      dirty: boolean;
      source: 'user' | 'effect' | 'reset' | 'initial' | 'derived';
      changedField?: FieldKey<TValues>;
    }
  ) => void;
  
  /** Imperative handle for external control */
  formRef?: React.Ref<SchemaFormHandle<TValues>>;
  
  /** Disable entire form */
  disabled?: boolean;
}

export interface SchemaFormHandle<TValues extends FieldValues> {
  /** Get all current values */
  getValues: () => TValues;
  
  /** Set multiple values */
  setValues: (values: Partial<TValues>) => void;
  
  /** Reset form to defaults or provided values */
  reset: (values?: Partial<TValues>) => void;
  
  /** Submit the form programmatically */
  submit: () => Promise<void>;
  
  /** Check if form has unsaved changes */
  isDirty: () => boolean;
  
  /** Check if form is valid (runs validation) */
  isValid: () => Promise<boolean>;
}

export type SubmitResult<TValues extends FieldValues> = {
  fieldErrors?: Partial<Record<FieldPath<TValues> | string, string>>;
  formError?: string;
};
```

### Layout System

```ts
export type LayoutComponent<TValues extends FieldValues> = React.ComponentType<
  LayoutProps<TValues>
>;

export interface LayoutProps<TValues extends FieldValues> {
  /** Render a configured field by name */
  Field: React.ComponentType<{ name: FieldKey<TValues> }>;
  
  /** List of configured field names (for iteration) */
  fieldNames: FieldKey<TValues>[];
  
  /** Full RHF methods for advanced use */
  methods: UseFormReturn<TValues>;
  
  /** Current form values */
  values: TValues;
  
  /** Current errors */
  errors: FieldErrors<TValues>;
  
  /** Form-level error (from validation or submit) */
  formError?: string;
  
  /** Submission state */
  isSubmitting: boolean;
  
  /** Whether async defaults are loading */
  isLoading: boolean;
  
  /** Whether form has unsaved changes */
  isDirty: boolean;
  
  /** Whether form is valid */
  isValid: boolean;
  
  /** Validate specific fields programmatically */
  validateFields: (names: FieldKey<TValues>[]) => Promise<boolean>;
  
  /** Submit the form programmatically */
  submit: () => Promise<void>;
  
  /** Reset form to defaults */
  reset: () => void;
}
```

---

## Widget Utilities

These hooks help widgets manage complex internal structures while integrating properly with the form.

### useWidgetArray

For widgets that render arrays (lists, repeaters, etc.).

```ts
export interface UseWidgetArrayOptions<T> {
  /** Minimum items (prevents removing below this) */
  minItems?: number;
  /** Maximum items (prevents adding above this) */
  maxItems?: number;
}

export interface UseWidgetArrayReturn<T> {
  /** Current items */
  items: T[];
  
  /** Add item to end */
  append: (item: T) => void;
  
  /** Add item at index */
  insert: (index: number, item: T) => void;
  
  /** Remove item at index */
  remove: (index: number) => void;
  
  /** Move item from one index to another */
  move: (from: number, to: number) => void;
  
  /** Swap two items */
  swap: (indexA: number, indexB: number) => void;
  
  /** Replace item at index */
  update: (index: number, item: T) => void;
  
  /** Replace all items */
  replace: (items: T[]) => void;
  
  /** Whether more items can be added */
  canAppend: boolean;
  
  /** Whether items can be removed */
  canRemove: boolean;
  
  /** Get error for specific index and optional nested path */
  getError: (index: number, path?: string) => string | undefined;
}

declare function useWidgetArray<T>(
  props: WidgetRenderProps<any, any>,
  options?: UseWidgetArrayOptions<T>
): UseWidgetArrayReturn<T>;
```

### useWidgetObject

For widgets that render nested objects with multiple fields.

```ts
export interface UseWidgetObjectReturn<T extends Record<string, unknown>> {
  /** Current object value */
  value: T;
  
  /** Update single nested field */
  setField: <K extends keyof T>(field: K, value: T[K]) => void;
  
  /** Update multiple fields at once */
  setFields: (updates: Partial<T>) => void;
  
  /** Get error for nested field */
  getError: (field: keyof T) => string | undefined;
  
  /** Mark nested field as touched */
  touchField: (field: keyof T) => void;
}

declare function useWidgetObject<T extends Record<string, unknown>>(
  props: WidgetRenderProps<any, any>
): UseWidgetObjectReturn<T>;
```

---

## Behavioral Notes

1. **Field keys**: Schema configuration uses top-level keys (`FieldKey<TValues>`). Nested paths (`FieldPath<TValues>`) are supported in validation results and effects for targeting nested errors.

2. **Partial configs**: Not all `TValues` keys need config entries. Unconfigured fields can be managed via effects or aren't user-editable.

3. **Async props lifecycle**: Async props show `fallback` during loading. Widgets never see loading/error state â the field wrapper handles that.

4. **Validation order**: 
   - Required check (from `required` config)
   - Field-level `validate` 
   - Form-level `validate`
   - All async validators awaited in parallel

5. **Dependencies**: `dependsOn` triggers re-evaluation of:
   - `visible`, `disabled`, `required` conditions
   - `props` factory (if function)
   - `asyncProps` factory
   - `deriveValue` computation
   - Does NOT re-run `onChange` effects (those are triggered by user changes only)

6. **Derived values**: `deriveValue` runs on:
   - Initial load (including draft hydration)
   - When any `dependsOn` field changes
   - Use `deriveValueCacheKey` to prevent duplicate async fetches across fields
   - Fields with `deriveValue` should typically be `disabled: true`

7. **Effects vs Derivations**:
   - Use `deriveValue` for computed values (runs on init, draft-safe)
   - Use `onChange` for side effects like resets, analytics (user-triggered only)

8. **Draft support**:
   - Hydration: use `defaultValues` async function to load drafts
   - Dehydration: use `formRef.getValues()` or `onValuesChange` callback
   - `deriveValue` fields auto-populate on draft load (unlike `onChange`)

9. **Transform**: `transformBeforeSubmit` runs after all validation passes, before `onSubmit`. Errors thrown here surface as `formError`.

10. **Performance**: 
    - Field components should be isolated (only re-render when their specific field changes)
    - Async props are memoized via `cacheKey`
    - Async `deriveValue` memoized via `deriveValueCacheKey`
    - Effects only run for user-initiated changes

---

## Built-in Widgets (Optional Package)

The core library is widget-agnostic, but a companion package provides common widgets:

```ts
// @your-org/form-widgets

export const TextInput: WidgetComponent<any, any>;
export const TextArea: WidgetComponent<any, any>;
export const NumberInput: WidgetComponent<any, any>;
export const Select: WidgetComponent<any, any>;      // props: { options }
export const MultiSelect: WidgetComponent<any, any>; // props: { options }
export const Checkbox: WidgetComponent<any, any>;
export const CheckboxGroup: WidgetComponent<any, any>; // props: { options }
export const RadioGroup: WidgetComponent<any, any>;    // props: { options }
export const DatePicker: WidgetComponent<any, any>;
export const DateRangePicker: WidgetComponent<any, any>;
export const FilePicker: WidgetComponent<any, any>;
export const ArrayField: WidgetComponent<any, any>;  // props: { renderItem }
export const ObjectField: WidgetComponent<any, any>; // props: { fields }
```
