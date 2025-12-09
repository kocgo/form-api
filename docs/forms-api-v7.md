# Forms API v7 Specification

## Overview
This document outlines the v7 design for a schema-driven form system built on top of `react-hook-form` with React Query powering asynchronous defaults and widget props. The API emphasizes declarative configuration: developers describe form fields, validation rules, effects, and layout in a typed schema paired with a layout component, while the engine handles wiring, async flows, and validation lifecycle.

## Design Goals
- Provide a minimal, strongly typed schema for defining fields, validation, defaults, and effects.
- Keep the form description in pure config (`FormSchema`) without embedding rendering logic.
- Support async defaults, async validators, and async widget props through internal data-fetching, not user plumbing.
- Enable pluggable layouts via a React component that receives `Field` and `react-hook-form` methods.
- Rely solely on field- and form-level validators—no separate validation schema.

## TypeScript API
Below are the v7 types. Names and signatures are fixed; the engine implementation is out of scope for this spec. Each section includes brief guidance on usage.

### Form schema
Define one `FormSchema` per form. It is pure configuration describing defaults, fields, validators, effects, and optional submit transformation.

```ts
type DefaultValuesSync<TValues> =
  | Partial<TValues>
  | ((ctx: { initial?: unknown }) => Partial<TValues>);

type DefaultValuesAsync<TValues> = (
  ctx: { initial?: unknown }
) => Promise<Partial<TValues>>;

type DefaultValuesConfig<TValues> =
  | DefaultValuesSync<TValues>
  | DefaultValuesAsync<TValues>;

export interface FormSchema<TValues> {
  id: string;
  defaultValues: DefaultValuesConfig<TValues>;
  fields: FieldConfig<TValues>[];
  formValidators?: {
    sync?: SyncFormValidator<TValues>[];
    async?: AsyncFormValidator<TValues>[];
  };
  effects?: FormEffects<TValues>;
  // transform UI model -> API model after validation, before onSubmit
  transformOnSubmit?: (values: TValues) => TValues | Promise<TValues>;
}
```

- `defaultValues` can be sync or async; async defaults are resolved internally (e.g., via React Query) based on optional `initial` context.
- `formValidators` holds sync/async form-level validation rules; there is no separate validation schema.
- `transformOnSubmit` allows final shaping of validated values before they reach `onSubmit`.

### Field config
Fields declare their rendering component, props, validators, and UI-only conditions. Widget props can be static, sync factories, or async factories; async props are resolved for the consumer automatically.

```ts
export interface FieldWidgetProps<TValues> {
  name: FieldPath<TValues>;
  label?: string;
  meta?: Record<string, unknown>;
  propsLoading?: boolean;
  propsError?: unknown;
}

// UI-only conditions
export interface FieldConditions<TValues> {
  visibleWhen?: (values: TValues) => boolean;   // default: true
  disabledWhen?: (values: TValues) => boolean;  // default: false
  requiredWhen?: (values: TValues) => boolean;  // optional
}

// Field-level validators
export interface SyncFieldValidator<TValues> {
  id?: string;
  when?: (values: TValues) => boolean;
  validate: (value: unknown, values: TValues) => string | null;
}

export interface AsyncFieldValidator<TValues> {
  id?: string;
  debounceMs?: number;
  when?: (values: TValues) => boolean;
  validate: (value: unknown, values: TValues) => Promise<string | null>;
}

export interface FieldValidators<TValues> {
  sync?: SyncFieldValidator<TValues>[];
  async?: AsyncFieldValidator<TValues>[];
}

// Widget props (static / sync / async)
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
  // local transform; sync only
  onValueChange?: (params: {
    value: unknown;
    values: TValues;
    api: EffectApi<TValues>;
  }) => void;
}
```

- `component` receives `FieldWidgetProps` plus any component-specific props. The engine injects `propsLoading`/`propsError` when async props are used.
- `conditions` only affect UI state (visibility, disabled, required) and are re-evaluated as values change.
- `validators` attach sync/async rules; async validators can be debounced.
- `onValueChange` enables per-field side effects (sync) using the effect API for controlled updates.

### Form-level validators
Use these to validate across multiple fields. Each validator lists the fields it covers so the engine can trigger and map errors appropriately.

```ts
export interface FormRuleResult<TValues> {
  [fieldName: string]: string; // can include '_form' for global error
}

export interface SyncFormValidator<TValues> {
  id: string;
  fields: (keyof TValues | string)[];
  when?: (values: TValues) => boolean;
  validate: (values: TValues) => FormRuleResult<TValues> | null;
}

export interface AsyncFormValidator<TValues> {
  id: string;
  fields: (keyof TValues | string)[];
  when?: (values: TValues) => boolean;
  validate: (values: TValues) => Promise<FormRuleResult<TValues> | null>;
}
```

- `FormRuleResult` maps field names to error messages; use `_form` for a form-level error.
- `when` lets rules short-circuit based on current values.
- Async validators can fetch server-side validation; the engine manages lifecycle (including optional debounce on field-level validators only).

### Effects
Effects allow reactive updates based on field changes. Use them for derived fields, cascading resets, or other side effects that should run as values change.

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
  watch: (keyof TValues | string)[];
  when?: (values: TValues) => boolean;
  run: (api: EffectApi<TValues>) => void | Promise<void>;
  debounceMs?: number;
}

export interface FormEffects<TValues> {
  onFieldChange?: FieldChangeEffect<TValues>[];
}
```

- `watch` lists the fields that trigger the effect; `when` gates execution.
- Effects can be async; the engine will handle awaited side effects and optional debounce.
- `EffectApi` provides safe value access and updates without direct interaction with react-hook-form internals.

### SchemaForm & layout (with validation mode)
`SchemaForm` binds a `FormSchema` to a layout. The layout is a React component receiving a `Field` component plus full RHF methods and state to render the form. Validation modes mirror `react-hook-form` options.

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

export type ValidationMode = 'onSubmit' | 'onChange' | 'onBlur' | 'onTouched';

export interface SchemaFormProps<TValues> {
  schema: FormSchema<TValues>;
  layout: React.ComponentType<SchemaFormLayoutProps<TValues>>;
  onSubmit: (values: TValues) => Promise<
    | void
    | { fieldErrors?: Record<string, string>; formError?: string }
  >;

  // mirrors RHF's `mode` and `reValidateMode`
  validationMode?: ValidationMode;                         // default: 'onSubmit'
  revalidationMode?: Exclude<ValidationMode, 'onSubmit'>;  // default: 'onChange'
}
```

- The layout renders structure, consuming `Field` for each schema field and `methods`/state for form controls. This keeps rendering fully customizable.
- `SchemaForm` resolves async defaults and widget props internally (likely via React Query) so consumers only supply config and layout.
- Validation uses only the provided validators; there is no external validation schema.

## Usage notes
- Treat the schema as the source of truth; avoid imperative RHF calls outside the layout unless they go through `EffectApi` or `methods`.
- Compose schemas with consistent `id` values to aid caching and debugging for async flows.
- For async validators and props, prefer debouncing where appropriate to limit network traffic; the engine will honor `debounceMs` when set.
