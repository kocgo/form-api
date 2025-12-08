# Forms API – Design v5 (Validation Schema Dropped)

This is the current agreed API design for our config‑driven form system **without `validationSchema`**. All validation is expressed via field and form validators.

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
  // Should submit wait for in‑flight async field validators?
  waitForAsyncValidationOnSubmit?: boolean; // default: true
}

export interface FormSchema<TValues> {
  id: string;
  defaultValues: DefaultValuesConfig<TValues>;
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

* No `validationSchema` anymore.
* All validation is done via `FieldConfig.validators` and `formValidators`.
* `defaultValues` can be static, sync factory, or async factory (engine uses React Query + `reset`).
* `transformOnSubmit` is the place to coerce/map values before calling `onSubmit`.

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
  requiredWhen?: (values: TValues) => boolean;  // optional, informs UI + validators
}

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
* `conditions` drive visibility/disabled/required at UI level.
* `validators` add field‑level rules only (no global schema).
* `props` can be static / sync factory / async factory (engine resolves via React Query, passing `propsLoading`/`propsError`).

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

* `fields` marks which fields the rule cares about (for step validation, UI, etc.).
* A `_form` key in the result represents a global error.

All cross‑field and business rules live here (sync or async).

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
  run: (api: EffectApi<TValues>) => void | Promise<void>;
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

* `onInit`: runs once after default values are ready.
* `onFieldChange`: runs on watched field changes (asyncable, debounced).
* `onSubmitSuccess` / `onSubmitError`: optional domain‑level hooks.

---

## 5. SchemaForm & Layout API

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

Per form, we still follow the same 3‑file pattern:

* `MyForm.schema.ts` → `MyFormValues`, `MyFormSchema`.
* `MyForm.layout.tsx` → `MyFormLayout` (uses `Field`, `validateFields`, etc.).
* `MyForm.tsx` → wires everything with `<SchemaForm schema layout onSubmit />`.

All validation is now via `validators` and `formValidators`; there is no separate `validationSchema` concept in this engine.
