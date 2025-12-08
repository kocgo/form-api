# form-api


# Forms API – Latest Design Snapshot

This is the current agreed API design for our config‑driven form system. Implementation comes later.

---

## 1. Form Schema

```ts
type DefaultValuesSync<TValues> =
  | Partial<TValues>
  | ((ctx: { initial?: unknown }) => Partial<TValues>);

// Async defaults resolved via React Query inside SchemaForm
type DefaultValuesAsync<TValues> = (
  ctx: { initial?: unknown },
) => Promise<Partial<TValues>>;

type DefaultValuesConfig<TValues> =
  | DefaultValuesSync<TValues>
  | DefaultValuesAsync<TValues>;

interface FormSchemaOptions {
  // Should submit wait for in‑flight async validators?
  waitForAsyncValidationOnSubmit?: boolean; // default: true
}

export interface FormSchema<TValues> {
  id: string;
  defaultValues: DefaultValuesConfig<TValues>;
  validationSchema?: unknown; // Yup/Zod/etc – sync core rules
  fields: FieldConfig<TValues>[];
  formValidators?: {
    sync?: SyncFormValidator<TValues>[];
    async?: AsyncFormValidator<TValues>[];
  };
  effects?: FormEffects<TValues>;
  transformOnSubmit?: (values: TValues) => TValues | Promise<TValues>;
  options?: FormSchemaOptions;
}
```

* `defaultValues` can be static, sync factory, or async factory (resolved via React Query and `reset`).
* `validationSchema` is for basic sync validation.
* `fields` describe widgets, validators, conditions, props, etc.
* `formValidators` are cross‑field rules.
* `effects` describes side‑effects (init, field change, submit success/error).
* `transformOnSubmit` allows mapping UI model → API model after validation.

---

## 2. Field Config

```ts
export interface FieldWidgetProps<TValues> {
  name: FieldPath<TValues>;
  label?: string;
  meta?: Record<string, unknown>;
  propsLoading?: boolean;
  propsError?: unknown;
}

// Conditions are pure predicates over values
export interface FieldConditions<TValues> {
  visibleWhen?: (values: TValues) => boolean;   // default: true
  disabledWhen?: (values: TValues) => boolean;  // default: false
  requiredWhen?: (values: TValues) => boolean;  // optional, informs UI + rules
}

// Field‑level validators
export type ValidationLevel = 'error' | 'warning' | 'info';

export interface SyncFieldValidator<TValues> {
  id?: string;
  level?: ValidationLevel;
  when?: (values: TValues) => boolean;
  validate: (value: unknown, values: TValues) => string | null;
}

export interface AsyncFieldValidatorDirect<TValues> {
  id?: string;
  level?: ValidationLevel;
  debounceMs?: number;
  when?: (values: TValues) => boolean;
  validate: (value: unknown, values: TValues) => Promise<string | null>;
}

export interface AsyncFieldValidatorRQ<TValues, TData = unknown> {
  id?: string;
  level?: ValidationLevel;
  debounceMs?: number;
  mode: 'react-query';
  queryKey: (ctx: { value: unknown; values: TValues }) => any[];
  queryFn: (ctx: { value: unknown; values: TValues }) => Promise<TData>;
  mapResultToError: (
    data: TData,
    ctx: { value: unknown; values: TValues },
  ) => string | null;
  enabled?: (ctx: { value: unknown; values: TValues }) => boolean;
}

export interface FieldValidators<TValues> {
  sync?: SyncFieldValidator<TValues>[];
  async?: (AsyncFieldValidatorDirect<TValues> | AsyncFieldValidatorRQ<TValues>)[];
}

// Widget props can be static, sync, or async
export type WidgetPropsFactorySync<TValues> = (ctx: {
  values: TValues;
  getValue: <K extends keyof TValues>(name: K) => TValues[K];
}) => Record<string, unknown>;

export type WidgetPropsFactoryAsync<TValues> = (ctx: {
  values: TValues;
  getValue: <K extends keyof TValues>(name: K) => TValues[K];
}) => Promise<Record<string, unknown>>;

export type WidgetPropsConfig<TValues> =
  | Record<string, unknown>
  | WidgetPropsFactorySync<TValues>
  | WidgetPropsFactoryAsync<TValues>;

export interface FieldConfig<TValues> {
  name: keyof TValues | string;
  label?: string;
  component: React.ComponentType<FieldWidgetProps<TValues> & any>;
  props?: WidgetPropsConfig<TValues>;
  validators?: FieldValidators<TValues>;
  conditions?: FieldConditions<TValues>;
  // local transform; can be sync or async
  onValueChange?: (params: {
    value: unknown;
    values: TValues;
    api: EffectApi<TValues>;
  }) => void | Promise<void>;
}
```

* `FieldConfig` connects a `name` to a widget component.
* `conditions` drive visibility/disabled/required at the UI level.
* `validators` add extra field‑level rules (sync or async).
* `props` can be:

  * static object,
  * sync factory,
  * async factory resolved via React Query in the engine (with `propsLoading/propsError`).
* `onValueChange` is for small, local value adjustments.

---

## 3. Form‑level Validators

```ts
export interface FormRuleResult<TValues> {
  [fieldName: string]: string; // including optional '_form' for global error
}

export interface SyncFormValidator<TValues> {
  id: string;
  level?: ValidationLevel;
  fields: (keyof TValues | string)[];
  when?: (values: TValues) => boolean;
  validate: (values: TValues) => FormRuleResult<TValues> | null;
}

export interface AsyncFormValidator<TValues> {
  id: string;
  level?: ValidationLevel;
  fields: (keyof TValues | string)[];
  when?: (values: TValues) => boolean;
  validate: (values: TValues) => Promise<FormRuleResult<TValues> | null>;
}
```

* `fields` indicate which fields a rule relates to (for step‑level validation, UI, etc.).
* Results can target specific fields or `_form` as a global error.

---

## 4. Effects

```ts
export interface EffectApi<TValues> {
  getValues: () => TValues;
  setValue: (
    name: keyof TValues | string,
    value: any,
    opts?: { silent?: boolean },
  ) => void;
  resetField: (name: keyof TValues | string) => void;
}

export interface FieldChangeEffect<TValues> {
  id?: string;
  watch: (keyof TValues | string)[]; // fields to watch
  when?: (values: TValues) => boolean;
  run: (api: EffectApi<TValues>) => void | Promise<void>; // asyncable
  debounceMs?: number;
}

export interface InitEffect<TValues> {
  id?: string;
  run: (api: EffectApi<TValues>) => void | Promise<void>;
}

export interface SubmitSuccessEffect<TValues, TResult = unknown> {
  id?: string;
  run: (params: {
    api: EffectApi<TValues>;
    values: TValues;
    result: TResult | unknown;
  }) => void | Promise<void>;
}

export interface SubmitErrorEffect<TValues> {
  id?: string;
  run: (params: {
    api: EffectApi<TValues>;
    values: TValues;
    error: unknown;
  }) => void | Promise<void>;
}

export interface FormEffects<TValues> {
  onInit?: InitEffect<TValues>[];
  onFieldChange?: FieldChangeEffect<TValues>[];
  onSubmitSuccess?: SubmitSuccessEffect<TValues>[];
  onSubmitError?: SubmitErrorEffect<TValues>[];
}
```

* `onInit` runs once after defaults are ready.
* `onFieldChange` runs when watched fields change (with optional debounce, async‑able).
* `onSubmitSuccess` / `onSubmitError` are optional domain‑level hooks.

---

## 5. SchemaForm Component & Layout API

```ts
export interface SchemaFormLayoutProps<TValues> {
  Field: React.ComponentType<{ name: keyof TValues | string }>;
  methods: UseFormReturn<TValues>; // full RHF API
  values: TValues;
  errors: FieldErrors<TValues>;
  isSubmitting: boolean;
  defaultsLoading: boolean;
  validateFields: (names: (keyof TValues | string)[]) => Promise<boolean>;
}

export interface SchemaFormProps<TValues> {
  schema: FormSchema<TValues>;
  layout: React.ComponentType<SchemaFormLayoutProps<TValues>>;
  onSubmit: (values: TValues) => Promise<
    | void
    | { fieldErrors?: Record<string, string>; formError?: string }
  >;
}
```

Usage per form:

* `MyForm.schema.ts` – exports `MyFormValues` + `MyFormSchema`.
* `MyForm.layout.tsx` – exports `MyFormLayout` using `Field` + layout logic.
* `MyForm.tsx` – wires everything:

```tsx
export function MyForm() {
  return (
    <SchemaForm<MyFormValues>
      schema={MyFormSchema}
      layout={MyFormLayout}
      onSubmit={async (values) => {
        // call API and optionally return { fieldErrors, formError }
      }}
    />
  );
}
```

This snapshot is the current reference for implementing the engine and writing schemas/layouts.
