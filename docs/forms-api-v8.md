# Forms API v8 Specification

## Overview
The v8 design simplifies the schema-driven form system introduced in v7 while tightening type safety and clarifying async flows. It keeps React Hook Form (RHF) as the runtime engine but enforces stricter, typed field keys, separates sync/async widget props, and collapses validation into a single, predictable shape. Effects are unified around field dependencies, making async props and derived values easier to reason about.

## Design goals
- Enforce typed field names everywhere (`FieldPath<TValues>`), removing escape hatches that break autocomplete and validation.
- Make async behavior explicit for widget props, validators, and defaults without leaking loading/error flags into widgets.
- Provide first-class support for field dependencies, arrays, and grouped fields for nested objects.
- Simplify validation to a single function form with clear result semantics and optional built-in rules.
- Keep layout pluggable while isolating form logic from rendering concerns.

## TypeScript API
### Core types
```ts
import {
  FieldErrors,
  FieldPath,
  FieldValues,
  PathValue,
  UseFormReturn,
} from 'react-hook-form';

export type FieldKey<TValues extends FieldValues> = FieldPath<TValues>;

export type DefaultValuesSync<TValues extends FieldValues> =
  | Partial<TValues>
  | ((ctx: { initial?: unknown }) => Partial<TValues>);

export type DefaultValuesAsync<TValues extends FieldValues> = (
  ctx: { initial?: unknown }
) => Promise<Partial<TValues>>;

export type DefaultValuesConfig<TValues extends FieldValues> =
  | DefaultValuesSync<TValues>
  | DefaultValuesAsync<TValues>;

export interface FormSchema<TValues extends FieldValues> {
  id: string;
  fields: FieldsConfig<TValues>;
  defaultValues?: DefaultValuesConfig<TValues>;
  validationMode?: ValidationMode; // default: 'onSubmit'
  revalidationMode?: Exclude<ValidationMode, 'onSubmit'>; // default: 'onChange'
  transform?: (values: TValues) => TValues | Promise<TValues>; // runs before submit
}
```

### Fields
Fields are declared as a typed map to preserve field-name autocompletion. Each field supports visibility, disability, and required conditions; nested objects and arrays are supported through grouped nodes.

```ts
export type FieldsConfig<TValues extends FieldValues> = {
  [K in FieldKey<TValues>]: FieldNode<TValues, K>;
};

export type FieldNode<TValues extends FieldValues, K extends FieldKey<TValues>> =
  | FieldConfig<TValues, K>
  | FieldArrayConfig<TValues, K>
  | FieldGroupConfig<TValues, K>;
```

#### Simple field
```ts
export interface FieldConfig<TValues extends FieldValues, K extends FieldKey<TValues>> {
  kind: 'field';
  label?: string;
  widget: WidgetRef<TValues, K>;
  props?: WidgetProps<TValues, K>;
  asyncProps?: AsyncWidgetProps<TValues, K>;
  dependsOn?: Dependencies<TValues>;
  visible?: ConditionFn<TValues>;
  disabled?: ConditionFn<TValues>;
  required?: boolean | ConditionFn<TValues>;
  validate?: Validator<TValues, K>;
  onChange?: FieldEffect<TValues, K>;
}
```

#### Field arrays
```ts
export interface FieldArrayConfig<TValues extends FieldValues, K extends FieldKey<TValues>> {
  kind: 'array';
  label?: string;
  of: FieldNode<TValues, K>;
  minItems?: number;
  maxItems?: number;
  validate?: Validator<TValues, K>;
}
```

#### Field groups
```ts
export interface FieldGroupConfig<TValues extends FieldValues, K extends FieldKey<TValues>> {
  kind: 'group';
  label?: string;
  fields: FieldsConfig<TValues>;
  visible?: ConditionFn<TValues>;
  disabled?: ConditionFn<TValues>;
}
```

### Widget references and props
Widgets are referenced directly by component. Async props are resolved externally; widgets receive only the resolved props, keeping loading/error concerns out of widget signatures.

```ts
export type WidgetRef<TValues extends FieldValues, K extends FieldKey<TValues>> =
  React.ComponentType<WidgetRenderProps<TValues, K>>;

export interface WidgetRenderProps<TValues extends FieldValues, K extends FieldKey<TValues>> {
  name: K;
  label?: string;
  required?: boolean;
  value: PathValue<TValues, K>;
  onChange: (value: PathValue<TValues, K>) => void;
  meta?: Record<string, unknown>;
  props: Record<string, unknown>;
}

export type WidgetPropsFactorySync<TValues extends FieldValues> = (ctx: {
  values: TValues;
}) => Record<string, unknown>;

export type WidgetPropsFactoryAsync<TValues extends FieldValues> = (ctx: {
  values: TValues;
}) => Promise<Record<string, unknown>>;

export type WidgetProps<TValues extends FieldValues, K extends FieldKey<TValues>> =
  | Record<string, unknown>
  | WidgetPropsFactorySync<TValues>;

export interface AsyncWidgetProps<TValues extends FieldValues, K extends FieldKey<TValues>> {
  factory: WidgetPropsFactoryAsync<TValues>;
  cacheKey?: string | ((values: TValues) => string);
}
```

### Dependencies and effects
Dependencies make data-loading and side effects explicit. Effects run when watched fields change and can update other fields through the effect API. This replaces v7’s split between `onValueChange` and `onFieldChange`.

```ts
export type Dependencies<TValues extends FieldValues> = FieldKey<TValues>[];

export type ConditionFn<TValues extends FieldValues> = (values: TValues) => boolean;

export interface EffectApi<TValues extends FieldValues> {
  getValues: () => TValues;
  setValue: (name: FieldKey<TValues>, value: any, opts?: { silent?: boolean }) => void;
  resetField: (name: FieldKey<TValues>) => void;
}

export type FieldEffect<TValues extends FieldValues, K extends FieldKey<TValues>> = (ctx: {
  name: K;
  value: PathValue<TValues, K>;
  values: TValues;
  api: EffectApi<TValues>;
}) => void | Promise<void>;
```

### Validation
Validation is unified into a single optional function per field or form. It can return sync or async results. Built-in primitive rules cover common cases without extra validators.

```ts
export type ValidationResult = string | null | Promise<string | null>;

export interface ValidationRules<TValues extends FieldValues, K extends FieldKey<TValues>> {
  required?: boolean | ConditionFn<TValues>;
  min?: number;
  max?: number;
  pattern?: RegExp;
  custom?: (value: PathValue<TValues, K>, values: TValues) => ValidationResult;
}

export type Validator<TValues extends FieldValues, K extends FieldKey<TValues>> =
  | ValidationRules<TValues, K>
  | ((value: PathValue<TValues, K>, values: TValues) => ValidationResult);

export type FormValidatorResult<TValues extends FieldValues> = Partial<
  Record<FieldKey<TValues> | '_form', string>
>;

export type FormValidator<TValues extends FieldValues> = (
  values: TValues
) => FormValidatorResult<TValues> | Promise<FormValidatorResult<TValues>>;
```

### SchemaForm & layout
`SchemaForm` binds the schema to a layout component. The layout receives resolved values, errors, and a `Field` renderer. Async defaults and async widget props are handled internally (e.g., via React Query), and widgets never receive loading/error flags directly.

```ts
export interface SchemaFormLayoutProps<TValues extends FieldValues> {
  Field: React.ComponentType<{ name: FieldKey<TValues> }>;
  methods: UseFormReturn<TValues>;
  values: TValues;
  errors: FieldErrors<TValues>;
  isSubmitting: boolean;
  defaultsLoading: boolean;
  validateFields: (names: FieldKey<TValues>[]) => Promise<boolean>;
}

export type ValidationMode = 'onSubmit' | 'onChange' | 'onBlur' | 'onTouched';

export interface SchemaFormProps<TValues extends FieldValues> {
  schema: FormSchema<TValues>;
  layout: React.ComponentType<SchemaFormLayoutProps<TValues>>;
  onSubmit: (values: TValues) => Promise<
    | void
    | { fieldErrors?: Partial<Record<FieldKey<TValues>, string>>; formError?: string }
  >;
}
```

## Behavioral notes
- **Typed names only**: All field references use `FieldPath<TValues>`; no untyped `string` escape hatches.
- **Explicit async props**: Widgets receive only resolved props. Async props are declared separately with optional cache keys; loading and error handling live in the field wrapper, not the widget.
- **Validation order**: Field-level rules run before form-level validators. Async validators are awaited; debounce is left to the validator implementation or the form engine’s wrapper.
- **Dependencies**: `dependsOn` hints the engine to re-run async props and effects when upstream fields change, reducing unnecessary rerenders and fetches.
- **Field arrays and groups**: Arrays model repeated structures; groups allow nested objects with shared visibility/disabled conditions.
- **Transform**: `transform` runs after validation but before `onSubmit`; failures bubble through `onSubmit` error handling.
- **Performance**: Engines SHOULD isolate rerenders per field, memoize async props via `cacheKey`, and avoid cascading effects without dependency declaration.
