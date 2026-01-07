# Forms API v12 — Usage Examples

This document provides practical examples covering common form scenarios.

---

## Table of Contents

1. [Simple Form](#1-simple-form)
2. [Form with Validation](#2-form-with-validation)
3. [Conditional Fields](#3-conditional-fields)
4. [Async Default Values](#4-async-default-values)
5. [Async Widget Props (Dynamic Options)](#5-async-widget-props-dynamic-options)
6. [Derived Values vs Effects](#6-derived-values-vs-effects)
7. [Cross-Field Validation](#7-cross-field-validation)
8. [Nested Object Fields](#8-nested-object-fields)
9. [Array Fields (Repeaters)](#9-array-fields-repeaters)
10. [Complex Real-World Form](#10-complex-real-world-form)
11. [Custom Widgets](#11-custom-widgets)
12. [Custom Layout](#12-custom-layout)
13. [Draft Support (Save & Resume)](#13-draft-support-save--resume)

---

## 1. Simple Form

Basic contact form with minimal configuration.

```tsx
// schemas/contact-form.ts
import { FormSchema } from '@your-org/forms';
import { TextInput, TextArea } from '@your-org/form-widgets';

interface ContactFormValues {
  name: string;
  email: string;
  message: string;
}

export const contactFormSchema: FormSchema<ContactFormValues> = {
  id: 'contact-form',
  
  defaultValues: {
    name: '',
    email: '',
    message: '',
  },
  
  fields: {
    name: {
      label: 'Your Name',
      widget: TextInput,
      required: true,
      placeholder: 'John Doe',
    },
    email: {
      label: 'Email Address',
      widget: TextInput,
      required: true,
      props: { type: 'email' },
    },
    message: {
      label: 'Message',
      widget: TextArea,
      required: true,
      props: { rows: 5 },
    },
  },
};
```

```tsx
// pages/contact.tsx
import { SchemaForm } from '@your-org/forms';
import { StackLayout } from '@your-org/form-layouts';
import { contactFormSchema } from '../schemas/contact-form';

export function ContactPage() {
  const handleSubmit = async (values: ContactFormValues) => {
    await api.sendContactMessage(values);
    toast.success('Message sent!');
  };

  return (
    <SchemaForm
      schema={contactFormSchema}
      layout={StackLayout}
      onSubmit={handleSubmit}
    />
  );
}
```

---

## 2. Form with Validation

Registration form with various validation rules.

```tsx
// schemas/registration-form.ts
import { FormSchema } from '@your-org/forms';
import { TextInput, Checkbox } from '@your-org/form-widgets';

interface RegistrationValues {
  username: string;
  email: string;
  password: string;
  confirmPassword: string;
  acceptTerms: boolean;
}

export const registrationSchema: FormSchema<RegistrationValues> = {
  id: 'registration',
  
  defaultValues: {
    username: '',
    email: '',
    password: '',
    confirmPassword: '',
    acceptTerms: false,
  },
  
  fields: {
    username: {
      label: 'Username',
      widget: TextInput,
      validate: {
        required: true,
        minLength: 3,
        maxLength: 20,
        pattern: {
          value: /^[a-zA-Z0-9_]+$/,
          message: 'Only letters, numbers, and underscores allowed',
        },
        // Async validation: check if username is taken
        custom: async (value) => {
          if (!value) return null;
          const taken = await api.checkUsername(value);
          return taken ? 'Username is already taken' : null;
        },
      },
    },
    
    email: {
      label: 'Email',
      widget: TextInput,
      props: { type: 'email' },
      validate: {
        required: true,
        pattern: {
          value: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
          message: 'Please enter a valid email address',
        },
      },
    },
    
    password: {
      label: 'Password',
      widget: TextInput,
      props: { type: 'password' },
      validate: {
        required: true,
        minLength: 8,
        custom: (value) => {
          if (!value) return null;
          const hasUpper = /[A-Z]/.test(value);
          const hasLower = /[a-z]/.test(value);
          const hasNumber = /[0-9]/.test(value);
          if (!hasUpper || !hasLower || !hasNumber) {
            return 'Password must contain uppercase, lowercase, and number';
          }
          return null;
        },
      },
    },
    
    confirmPassword: {
      label: 'Confirm Password',
      widget: TextInput,
      props: { type: 'password' },
      dependsOn: ['password'],
      validate: (value, values) => {
        if (!value) return 'Please confirm your password';
        if (value !== values.password) return 'Passwords do not match';
        return null;
      },
    },
    
    acceptTerms: {
      label: 'I accept the terms and conditions',
      widget: Checkbox,
      validate: (value) => value ? null : 'You must accept the terms',
    },
  },
};
```

---

## 3. Conditional Fields

Form with fields that show/hide based on other values.

```tsx
// schemas/shipping-form.ts
import { FormSchema } from '@your-org/forms';
import { TextInput, Select, Checkbox } from '@your-org/form-widgets';

interface ShippingValues {
  shippingMethod: 'standard' | 'express' | 'pickup';
  address: string;
  city: string;
  zipCode: string;
  pickupLocation: string;
  giftWrap: boolean;
  giftMessage: string;
}

export const shippingSchema: FormSchema<ShippingValues> = {
  id: 'shipping',
  
  defaultValues: {
    shippingMethod: 'standard',
    address: '',
    city: '',
    zipCode: '',
    pickupLocation: '',
    giftWrap: false,
    giftMessage: '',
  },
  
  fields: {
    shippingMethod: {
      label: 'Shipping Method',
      widget: Select,
      required: true,
      props: {
        options: [
          { value: 'standard', label: 'Standard Shipping (5-7 days)' },
          { value: 'express', label: 'Express Shipping (2-3 days)' },
          { value: 'pickup', label: 'Store Pickup' },
        ],
      },
    },
    
    // Only visible when NOT pickup
    address: {
      label: 'Street Address',
      widget: TextInput,
      visible: (values) => values.shippingMethod !== 'pickup',
      required: (values) => values.shippingMethod !== 'pickup',
    },
    
    city: {
      label: 'City',
      widget: TextInput,
      visible: (values) => values.shippingMethod !== 'pickup',
      required: (values) => values.shippingMethod !== 'pickup',
    },
    
    zipCode: {
      label: 'ZIP Code',
      widget: TextInput,
      visible: (values) => values.shippingMethod !== 'pickup',
      required: (values) => values.shippingMethod !== 'pickup',
      validate: {
        pattern: /^\d{5}(-\d{4})?$/,
      },
    },
    
    // Only visible when pickup
    pickupLocation: {
      label: 'Pickup Location',
      widget: Select,
      visible: (values) => values.shippingMethod === 'pickup',
      required: (values) => values.shippingMethod === 'pickup',
      props: {
        options: [
          { value: 'downtown', label: 'Downtown Store' },
          { value: 'mall', label: 'Mall Location' },
          { value: 'warehouse', label: 'Warehouse' },
        ],
      },
    },
    
    giftWrap: {
      label: 'Add gift wrapping (+$5)',
      widget: Checkbox,
    },
    
    // Only visible when gift wrap is checked
    giftMessage: {
      label: 'Gift Message',
      widget: TextInput,
      visible: (values) => values.giftWrap,
      placeholder: 'Enter a message for the recipient',
    },
  },
};
```

---

## 4. Async Default Values

Edit form that loads existing data.

```tsx
// schemas/profile-form.ts
import { FormSchema } from '@your-org/forms';
import { TextInput, Select, DatePicker } from '@your-org/form-widgets';

interface ProfileValues {
  firstName: string;
  lastName: string;
  email: string;
  phone: string;
  birthDate: string;
  timezone: string;
}

export const profileSchema: FormSchema<ProfileValues> = {
  id: 'profile',
  
  // Async default values — loads user profile
  defaultValues: async (ctx) => {
    // ctx.initial might contain user ID or partial data
    const userId = ctx.params?.userId ?? ctx.initial?.id;
    
    if (!userId) {
      // New profile — return empty defaults
      return {
        firstName: '',
        lastName: '',
        email: '',
        phone: '',
        birthDate: '',
        timezone: 'UTC',
      };
    }
    
    // Edit mode — fetch existing profile
    const profile = await api.getProfile(userId);
    return {
      firstName: profile.firstName,
      lastName: profile.lastName,
      email: profile.email,
      phone: profile.phone ?? '',
      birthDate: profile.birthDate ?? '',
      timezone: profile.timezone ?? 'UTC',
    };
  },
  
  fields: {
    firstName: {
      label: 'First Name',
      widget: TextInput,
      required: true,
    },
    lastName: {
      label: 'Last Name',
      widget: TextInput,
      required: true,
    },
    email: {
      label: 'Email',
      widget: TextInput,
      required: true,
      disabled: true, // Can't change email
      description: 'Contact support to change your email',
    },
    phone: {
      label: 'Phone Number',
      widget: TextInput,
      props: { type: 'tel' },
    },
    birthDate: {
      label: 'Birth Date',
      widget: DatePicker,
    },
    timezone: {
      label: 'Timezone',
      widget: Select,
      props: {
        options: [
          { value: 'UTC', label: 'UTC' },
          { value: 'America/New_York', label: 'Eastern Time' },
          { value: 'America/Chicago', label: 'Central Time' },
          { value: 'America/Denver', label: 'Mountain Time' },
          { value: 'America/Los_Angeles', label: 'Pacific Time' },
        ],
      },
    },
  },
};

// Usage
function ProfilePage({ userId }: { userId: string }) {
  return (
    <SchemaForm
      schema={profileSchema}
      layout={StackLayout}
      params={{ userId }}
      onSubmit={async (values) => {
        await api.updateProfile(userId, values);
        toast.success('Profile updated!');
      }}
    />
  );
}
```

---

## 5. Async Widget Props (Dynamic Options)

Form with dropdowns that load options based on other field values.

```tsx
// schemas/location-form.ts
import { FormSchema } from '@your-org/forms';
import { Select } from '@your-org/form-widgets';

interface LocationValues {
  country: string;
  state: string;
  city: string;
}

export const locationSchema: FormSchema<LocationValues> = {
  id: 'location',
  
  defaultValues: {
    country: '',
    state: '',
    city: '',
  },
  
  fields: {
    country: {
      label: 'Country',
      widget: Select,
      required: true,
      // Static options (could also be async)
      props: {
        options: [
          { value: 'us', label: 'United States' },
          { value: 'ca', label: 'Canada' },
          { value: 'mx', label: 'Mexico' },
        ],
      },
    },
    
    state: {
      label: 'State/Province',
      widget: Select,
      required: true,
      dependsOn: ['country'],
      
      // Disable until country is selected
      disabled: (values) => !values.country,
      
      // Load states based on country
      asyncProps: {
        factory: async ({ values }) => {
          if (!values.country) {
            return { options: [] };
          }
          const states = await api.getStates(values.country);
          return {
            options: states.map((s) => ({ value: s.code, label: s.name })),
          };
        },
        // Cache by country so switching back doesn't refetch
        cacheKey: (values) => `states-${values.country}`,
        // Show while loading
        fallback: { options: [], placeholder: 'Loading states...' },
      },
    },
    
    city: {
      label: 'City',
      widget: Select,
      required: true,
      dependsOn: ['country', 'state'],
      
      disabled: (values) => !values.state,
      
      asyncProps: {
        factory: async ({ values }) => {
          if (!values.country || !values.state) {
            return { options: [] };
          }
          const cities = await api.getCities(values.country, values.state);
          return {
            options: cities.map((c) => ({ value: c.id, label: c.name })),
          };
        },
        cacheKey: (values) => `cities-${values.country}-${values.state}`,
        fallback: { options: [], placeholder: 'Loading cities...' },
      },
    },
  },
};
```

---

## 6. Derived Values vs Effects

Form showing the difference between `deriveValue` (computed) and `onChange` (side effects).

```tsx
// schemas/invoice-form.ts
import { FormSchema } from '@your-org/forms';
import { Select, NumberInput, TextInput } from '@your-org/form-widgets';

interface InvoiceValues {
  customerId: string;
  customerName: string;
  customerEmail: string;
  taxRate: number;
  subtotal: number;
  taxAmount: number;
  total: number;
}

export const invoiceSchema: FormSchema<InvoiceValues> = {
  id: 'invoice',
  
  defaultValues: {
    customerId: '',
    customerName: '',
    customerEmail: '',
    taxRate: 0,
    subtotal: 0,
    taxAmount: 0,
    total: 0,
  },
  
  fields: {
    customerId: {
      label: 'Customer',
      widget: Select,
      required: true,
      asyncProps: {
        factory: async () => {
          const customers = await api.getCustomers();
          return {
            options: customers.map((c) => ({ value: c.id, label: c.name })),
          };
        },
        cacheKey: 'customers',
      },
    },
    
    // DERIVED: customerName comes from customerId
    // Works on draft load — if draft has customerId, name auto-populates
    customerName: {
      label: 'Customer Name',
      widget: TextInput,
      disabled: true,
      dependsOn: ['customerId'],
      deriveValue: async (values) => {
        if (!values.customerId) return '';
        const customer = await fetchCustomer(values.customerId);
        return customer.name;
      },
      deriveValueCacheKey: (values) => `customer-${values.customerId}`,
    },
    
    // DERIVED: customerEmail comes from customerId
    // Same cache key means one fetch for both name and email
    customerEmail: {
      label: 'Customer Email',
      widget: TextInput,
      disabled: true,
      dependsOn: ['customerId'],
      deriveValue: async (values) => {
        if (!values.customerId) return '';
        const customer = await fetchCustomer(values.customerId);
        return customer.email;
      },
      deriveValueCacheKey: (values) => `customer-${values.customerId}`,
    },
    
    // DERIVED: taxRate comes from customer's region
    taxRate: {
      label: 'Tax Rate (%)',
      widget: NumberInput,
      disabled: true,
      dependsOn: ['customerId'],
      deriveValue: async (values) => {
        if (!values.customerId) return 0;
        const customer = await fetchCustomer(values.customerId);
        return customer.taxRate ?? 0;
      },
      deriveValueCacheKey: (values) => `customer-${values.customerId}`,
    },
    
    // User input
    subtotal: {
      label: 'Subtotal',
      widget: NumberInput,
      required: true,
      props: { min: 0, step: 0.01, prefix: '$' },
    },
    
    // DERIVED: taxAmount = subtotal * taxRate
    // Sync derivation — no async needed
    taxAmount: {
      label: 'Tax Amount',
      widget: NumberInput,
      disabled: true,
      dependsOn: ['subtotal', 'taxRate'],
      deriveValue: (values) => {
        const amount = values.subtotal * (values.taxRate / 100);
        return Math.round(amount * 100) / 100;
      },
    },
    
    // DERIVED: total = subtotal + taxAmount
    total: {
      label: 'Total',
      widget: NumberInput,
      disabled: true,
      dependsOn: ['subtotal', 'taxAmount'],
      deriveValue: (values) => {
        return Math.round((values.subtotal + values.taxAmount) * 100) / 100;
      },
    },
  },
};
```

### Why `deriveValue` over `onChange`?

```tsx
// Draft loads with: { customerId: '123', subtotal: 500, ... }

// With onChange: ❌ customerName stays empty
// onChange only fires when user changes customerId

// With deriveValue: ✅ customerName auto-populates
// deriveValue runs on init, sees customerId, fetches customer
```

### When to Use Each

```tsx
fields: {
  country: {
    widget: Select,
    // EFFECT: Reset dependent field when parent changes
    // This is a side effect, not a derivation
    onChange: ({ value, previousValue, api }) => {
      if (previousValue && value !== previousValue) {
        api.resetField('state'); // Clear state when country changes
      }
    },
  },
  
  state: {
    widget: Select,
    dependsOn: ['country'],
    // DERIVED OPTIONS: Options come from country
    asyncProps: {
      factory: async ({ values }) => ({
        options: await getStates(values.country),
      }),
    },
  },
  
  fullAddress: {
    widget: TextInput,
    disabled: true,
    dependsOn: ['street', 'city', 'state', 'zip'],
    // DERIVED VALUE: Computed from other fields
    deriveValue: (values) => {
      return `${values.street}, ${values.city}, ${values.state} ${values.zip}`;
    },
  },
}
```

---

## 7. Cross-Field Validation

Form-level validation for rules spanning multiple fields.

```tsx
// schemas/booking-form.ts
import { FormSchema } from '@your-org/forms';
import { DatePicker, NumberInput, Select } from '@your-org/form-widgets';

interface BookingValues {
  checkIn: string;
  checkOut: string;
  guests: number;
  roomType: 'standard' | 'deluxe' | 'suite';
}

export const bookingSchema: FormSchema<BookingValues> = {
  id: 'booking',
  
  defaultValues: {
    checkIn: '',
    checkOut: '',
    guests: 1,
    roomType: 'standard',
  },
  
  fields: {
    checkIn: {
      label: 'Check-in Date',
      widget: DatePicker,
      required: true,
      validate: (value) => {
        if (!value) return null;
        const date = new Date(value);
        const today = new Date();
        today.setHours(0, 0, 0, 0);
        if (date < today) return 'Check-in date cannot be in the past';
        return null;
      },
    },
    
    checkOut: {
      label: 'Check-out Date',
      widget: DatePicker,
      required: true,
      dependsOn: ['checkIn'],
    },
    
    guests: {
      label: 'Number of Guests',
      widget: NumberInput,
      required: true,
      validate: { min: 1, max: 10 },
    },
    
    roomType: {
      label: 'Room Type',
      widget: Select,
      required: true,
      props: {
        options: [
          { value: 'standard', label: 'Standard (max 2 guests)' },
          { value: 'deluxe', label: 'Deluxe (max 4 guests)' },
          { value: 'suite', label: 'Suite (max 6 guests)' },
        ],
      },
    },
  },
  
  // Form-level validation — cross-field rules
  validate: async (values) => {
    const errors: Record<string, string> = {};
    
    // Check-out must be after check-in
    if (values.checkIn && values.checkOut) {
      const checkIn = new Date(values.checkIn);
      const checkOut = new Date(values.checkOut);
      
      if (checkOut <= checkIn) {
        errors.checkOut = 'Check-out must be after check-in';
      }
      
      // Maximum stay of 30 days
      const daysDiff = (checkOut.getTime() - checkIn.getTime()) / (1000 * 60 * 60 * 24);
      if (daysDiff > 30) {
        errors.checkOut = 'Maximum stay is 30 days';
      }
    }
    
    // Room capacity validation
    const maxGuests: Record<string, number> = {
      standard: 2,
      deluxe: 4,
      suite: 6,
    };
    
    if (values.guests > maxGuests[values.roomType]) {
      errors.guests = `${values.roomType} room only accommodates ${maxGuests[values.roomType]} guests`;
    }
    
    // Async validation: check availability
    if (values.checkIn && values.checkOut && values.roomType) {
      const available = await api.checkAvailability({
        checkIn: values.checkIn,
        checkOut: values.checkOut,
        roomType: values.roomType,
      });
      
      if (!available) {
        return {
          formError: 'No rooms available for selected dates. Please try different dates.',
        };
      }
    }
    
    if (Object.keys(errors).length === 0) {
      return null;
    }
    
    return { fieldErrors: errors };
  },
};
```

---

## 8. Nested Object Fields

Form with address object using a custom widget.

```tsx
// schemas/customer-form.ts
import { FormSchema } from '@your-org/forms';
import { TextInput } from '@your-org/form-widgets';
import { AddressWidget } from '../widgets/AddressWidget';

interface Address {
  street: string;
  city: string;
  state: string;
  zipCode: string;
  country: string;
}

interface CustomerValues {
  name: string;
  email: string;
  billingAddress: Address;
  shippingAddress: Address;
  sameAsBilling: boolean;
}

export const customerSchema: FormSchema<CustomerValues> = {
  id: 'customer',
  
  defaultValues: {
    name: '',
    email: '',
    billingAddress: { street: '', city: '', state: '', zipCode: '', country: 'US' },
    shippingAddress: { street: '', city: '', state: '', zipCode: '', country: 'US' },
    sameAsBilling: true,
  },
  
  fields: {
    name: {
      label: 'Full Name',
      widget: TextInput,
      required: true,
    },
    
    email: {
      label: 'Email',
      widget: TextInput,
      required: true,
      props: { type: 'email' },
    },
    
    billingAddress: {
      label: 'Billing Address',
      widget: AddressWidget, // Custom widget handles nested fields
      required: true,
      props: {
        showCountry: true,
      },
      // Validation for nested object
      validate: (value) => {
        if (!value.street) return 'Street is required';
        if (!value.city) return 'City is required';
        if (!value.state) return 'State is required';
        if (!value.zipCode) return 'ZIP code is required';
        return null;
      },
    },
    
    sameAsBilling: {
      label: 'Shipping address same as billing',
      widget: Checkbox,
      // When toggled on, copy billing to shipping
      onChange: ({ value, values, api }) => {
        if (value) {
          api.setValue('shippingAddress', { ...values.billingAddress });
        }
      },
    },
    
    shippingAddress: {
      label: 'Shipping Address',
      widget: AddressWidget,
      visible: (values) => !values.sameAsBilling,
      required: (values) => !values.sameAsBilling,
      props: {
        showCountry: true,
      },
    },
  },
};
```

```tsx
// widgets/AddressWidget.tsx
import { WidgetRenderProps, useWidgetObject } from '@your-org/forms';
import { TextInput, Select } from '@your-org/form-widgets';

interface Address {
  street: string;
  city: string;
  state: string;
  zipCode: string;
  country: string;
}

interface AddressWidgetProps {
  showCountry?: boolean;
}

export function AddressWidget(
  props: WidgetRenderProps<any, any> & { extra: AddressWidgetProps }
) {
  const { value, setField, getError } = useWidgetObject<Address>(props);
  const { showCountry = false } = props.extra;
  
  return (
    <div className="address-widget">
      <TextInput
        label="Street Address"
        value={value.street}
        onChange={(v) => setField('street', v)}
        error={getError('street')}
        required
      />
      
      <div className="grid grid-cols-2 gap-4">
        <TextInput
          label="City"
          value={value.city}
          onChange={(v) => setField('city', v)}
          error={getError('city')}
          required
        />
        
        <TextInput
          label="State"
          value={value.state}
          onChange={(v) => setField('state', v)}
          error={getError('state')}
          required
        />
      </div>
      
      <div className="grid grid-cols-2 gap-4">
        <TextInput
          label="ZIP Code"
          value={value.zipCode}
          onChange={(v) => setField('zipCode', v)}
          error={getError('zipCode')}
          required
        />
        
        {showCountry && (
          <Select
            label="Country"
            value={value.country}
            onChange={(v) => setField('country', v)}
            options={[
              { value: 'US', label: 'United States' },
              { value: 'CA', label: 'Canada' },
            ]}
          />
        )}
      </div>
    </div>
  );
}
```

---

## 9. Array Fields (Repeaters)

Form with repeatable items.

```tsx
// schemas/team-form.ts
import { FormSchema } from '@your-org/forms';
import { TextInput } from '@your-org/form-widgets';
import { TeamMembersWidget } from '../widgets/TeamMembersWidget';

interface TeamMember {
  name: string;
  email: string;
  role: 'admin' | 'member' | 'viewer';
}

interface TeamValues {
  teamName: string;
  description: string;
  members: TeamMember[];
}

export const teamSchema: FormSchema<TeamValues> = {
  id: 'team',
  
  defaultValues: {
    teamName: '',
    description: '',
    members: [{ name: '', email: '', role: 'member' }], // Start with one empty member
  },
  
  fields: {
    teamName: {
      label: 'Team Name',
      widget: TextInput,
      required: true,
      validate: { minLength: 3, maxLength: 50 },
    },
    
    description: {
      label: 'Description',
      widget: TextInput,
      placeholder: 'What does this team do?',
    },
    
    members: {
      label: 'Team Members',
      widget: TeamMembersWidget,
      required: true,
      props: {
        minItems: 1,
        maxItems: 20,
      },
      validate: (value) => {
        if (!value || value.length === 0) {
          return 'At least one team member is required';
        }
        
        // Check for duplicate emails
        const emails = value.map((m) => m.email.toLowerCase());
        const duplicates = emails.filter((e, i) => emails.indexOf(e) !== i);
        if (duplicates.length > 0) {
          return 'Duplicate email addresses are not allowed';
        }
        
        // Validate each member
        for (let i = 0; i < value.length; i++) {
          if (!value[i].name) return `Member ${i + 1}: Name is required`;
          if (!value[i].email) return `Member ${i + 1}: Email is required`;
        }
        
        return null;
      },
    },
  },
  
  // Form-level: at least one admin
  validate: (values) => {
    const hasAdmin = values.members.some((m) => m.role === 'admin');
    if (!hasAdmin) {
      return {
        fieldErrors: {
          members: 'At least one team member must be an admin',
        },
      };
    }
    return null;
  },
};
```

```tsx
// widgets/TeamMembersWidget.tsx
import { WidgetRenderProps, useWidgetArray } from '@your-org/forms';
import { TextInput, Select } from '@your-org/form-widgets';
import { Plus, Trash2, GripVertical } from 'lucide-react';

interface TeamMember {
  name: string;
  email: string;
  role: 'admin' | 'member' | 'viewer';
}

export function TeamMembersWidget(
  props: WidgetRenderProps<any, any>
) {
  const { minItems = 1, maxItems = 20 } = props.extra;
  
  const {
    items,
    append,
    remove,
    move,
    update,
    canAppend,
    canRemove,
    getError,
  } = useWidgetArray<TeamMember>(props, { minItems, maxItems });
  
  const emptyMember: TeamMember = { name: '', email: '', role: 'member' };
  
  return (
    <div className="team-members-widget">
      {props.error && (
        <div className="text-red-500 text-sm mb-2">{props.error}</div>
      )}
      
      {items.map((member, index) => (
        <div key={index} className="member-row flex gap-2 mb-4 p-4 border rounded">
          <button type="button" className="drag-handle cursor-move">
            <GripVertical />
          </button>
          
          <div className="flex-1 grid grid-cols-3 gap-4">
            <TextInput
              label="Name"
              value={member.name}
              onChange={(v) => update(index, { ...member, name: v })}
              error={getError(index, 'name')}
              required
            />
            
            <TextInput
              label="Email"
              value={member.email}
              onChange={(v) => update(index, { ...member, email: v })}
              error={getError(index, 'email')}
              required
            />
            
            <Select
              label="Role"
              value={member.role}
              onChange={(v) => update(index, { ...member, role: v })}
              options={[
                { value: 'admin', label: 'Admin' },
                { value: 'member', label: 'Member' },
                { value: 'viewer', label: 'Viewer' },
              ]}
            />
          </div>
          
          {canRemove && (
            <button
              type="button"
              onClick={() => remove(index)}
              className="text-red-500"
            >
              <Trash2 />
            </button>
          )}
        </div>
      ))}
      
      {canAppend && (
        <button
          type="button"
          onClick={() => append(emptyMember)}
          className="flex items-center gap-2 text-blue-500"
        >
          <Plus /> Add Member
        </button>
      )}
    </div>
  );
}
```

---

## 10. Complex Real-World Form

Putting it all together: a job application form.

```tsx
// schemas/job-application.ts
import { FormSchema } from '@your-org/forms';
import { 
  TextInput, 
  TextArea, 
  Select, 
  DatePicker,
  FilePicker,
  Checkbox,
} from '@your-org/form-widgets';
import { WorkExperienceWidget } from '../widgets/WorkExperienceWidget';
import { EducationWidget } from '../widgets/EducationWidget';

interface WorkExperience {
  company: string;
  title: string;
  startDate: string;
  endDate: string | null;
  current: boolean;
  description: string;
}

interface Education {
  institution: string;
  degree: string;
  field: string;
  graduationYear: number;
}

interface JobApplicationValues {
  // Personal
  firstName: string;
  lastName: string;
  email: string;
  phone: string;
  
  // Position
  positionId: string;
  salary: number;
  startDate: string;
  
  // Experience
  workExperience: WorkExperience[];
  education: Education[];
  
  // Documents
  resume: File | null;
  coverLetter: string;
  
  // Compliance
  authorizedToWork: boolean;
  requiresSponsorship: boolean;
  
  // Consent
  agreeToTerms: boolean;
}

export const jobApplicationSchema: FormSchema<JobApplicationValues> = {
  id: 'job-application',
  
  defaultValues: async (ctx) => {
    // Could load draft from localStorage or API
    const draft = await loadDraft(ctx.params?.jobId);
    
    return {
      firstName: draft?.firstName ?? '',
      lastName: draft?.lastName ?? '',
      email: draft?.email ?? '',
      phone: draft?.phone ?? '',
      positionId: ctx.params?.jobId ?? '',
      salary: draft?.salary ?? 0,
      startDate: '',
      workExperience: draft?.workExperience ?? [],
      education: draft?.education ?? [],
      resume: null,
      coverLetter: draft?.coverLetter ?? '',
      authorizedToWork: false,
      requiresSponsorship: false,
      agreeToTerms: false,
    };
  },
  
  fields: {
    firstName: {
      label: 'First Name',
      widget: TextInput,
      required: true,
    },
    
    lastName: {
      label: 'Last Name',
      widget: TextInput,
      required: true,
    },
    
    email: {
      label: 'Email',
      widget: TextInput,
      required: true,
      props: { type: 'email' },
      validate: {
        pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
        custom: async (value) => {
          // Check if already applied
          const exists = await api.checkExistingApplication(value);
          if (exists) return 'An application with this email already exists';
          return null;
        },
      },
    },
    
    phone: {
      label: 'Phone',
      widget: TextInput,
      required: true,
      props: { type: 'tel' },
    },
    
    positionId: {
      label: 'Position',
      widget: Select,
      required: true,
      asyncProps: {
        factory: async () => {
          const positions = await api.getOpenPositions();
          return {
            options: positions.map((p) => ({
              value: p.id,
              label: `${p.title} - ${p.department}`,
            })),
          };
        },
        cacheKey: 'open-positions',
      },
      // When position changes, update salary expectation range
      onChange: async ({ value, api }) => {
        if (!value) return;
        const position = await api.getPosition(value);
        // Set to midpoint of range as default
        const midpoint = (position.salaryMin + position.salaryMax) / 2;
        api.setValue('salary', Math.round(midpoint / 1000) * 1000);
      },
    },
    
    salary: {
      label: 'Expected Salary',
      widget: NumberInput,
      required: true,
      dependsOn: ['positionId'],
      props: { prefix: '$', step: 1000 },
      asyncProps: {
        factory: async ({ values }) => {
          if (!values.positionId) return { min: 0, max: 1000000 };
          const position = await api.getPosition(values.positionId);
          return {
            min: position.salaryMin,
            max: position.salaryMax,
            helperText: `Range: $${position.salaryMin.toLocaleString()} - $${position.salaryMax.toLocaleString()}`,
          };
        },
        cacheKey: (values) => `salary-range-${values.positionId}`,
      },
    },
    
    startDate: {
      label: 'Available Start Date',
      widget: DatePicker,
      required: true,
      validate: (value) => {
        const date = new Date(value);
        const twoWeeks = new Date();
        twoWeeks.setDate(twoWeeks.getDate() + 14);
        if (date < twoWeeks) {
          return 'Start date must be at least 2 weeks from now';
        }
        return null;
      },
    },
    
    workExperience: {
      label: 'Work Experience',
      widget: WorkExperienceWidget,
      description: 'List your relevant work experience, most recent first',
      props: { maxItems: 10 },
    },
    
    education: {
      label: 'Education',
      widget: EducationWidget,
      props: { maxItems: 5 },
    },
    
    resume: {
      label: 'Resume',
      widget: FilePicker,
      required: true,
      props: {
        accept: '.pdf,.doc,.docx',
        maxSize: 5 * 1024 * 1024, // 5MB
      },
    },
    
    coverLetter: {
      label: 'Cover Letter',
      widget: TextArea,
      description: 'Tell us why you would be a great fit for this role',
      props: { rows: 8 },
      validate: {
        minLength: 100,
        maxLength: 5000,
      },
    },
    
    authorizedToWork: {
      label: 'Are you authorized to work in the United States?',
      widget: Checkbox,
      required: true,
      validate: (value) => {
        if (!value) return 'You must be authorized to work to apply';
        return null;
      },
    },
    
    requiresSponsorship: {
      label: 'Will you now or in the future require visa sponsorship?',
      widget: Checkbox,
    },
    
    agreeToTerms: {
      label: 'I certify that all information provided is accurate and complete',
      widget: Checkbox,
      required: true,
      validate: (value) => value ? null : 'You must agree to continue',
    },
  },
  
  // Cross-field validation
  validate: (values) => {
    const errors: Record<string, string> = {};
    
    // If no work experience, require explanation in cover letter
    if (values.workExperience.length === 0 && values.coverLetter.length < 200) {
      errors.coverLetter = 'Please provide more detail in your cover letter since you have no listed work experience';
    }
    
    return Object.keys(errors).length ? { fieldErrors: errors } : null;
  },
  
  // Transform before submit
  transformBeforeSubmit: async (values) => {
    // Could upload resume to S3 and replace File with URL
    return values;
  },
};
```

---

## 11. Custom Widgets

Creating your own widgets.

```tsx
// widgets/RatingWidget.tsx
import { WidgetRenderProps } from '@your-org/forms';
import { Star } from 'lucide-react';

interface RatingWidgetExtra {
  maxStars?: number;
  allowHalf?: boolean;
}

export function RatingWidget(
  props: WidgetRenderProps<any, any> & { extra: RatingWidgetExtra }
) {
  const { 
    value, 
    onChange, 
    label, 
    required, 
    disabled, 
    error,
    extra: { maxStars = 5, allowHalf = false } 
  } = props;
  
  const handleClick = (rating: number) => {
    if (disabled) return;
    // Toggle off if clicking same rating
    onChange(value === rating ? 0 : rating);
  };
  
  return (
    <div className="rating-widget">
      {label && (
        <label className="block text-sm font-medium mb-1">
          {label}
          {required && <span className="text-red-500 ml-1">*</span>}
        </label>
      )}
      
      <div className="flex gap-1">
        {Array.from({ length: maxStars }, (_, i) => i + 1).map((star) => (
          <button
            key={star}
            type="button"
            onClick={() => handleClick(star)}
            disabled={disabled}
            className={`p-1 transition-colors ${
              disabled ? 'cursor-not-allowed opacity-50' : 'cursor-pointer'
            }`}
          >
            <Star
              className={`w-6 h-6 ${
                star <= value
                  ? 'fill-yellow-400 text-yellow-400'
                  : 'text-gray-300'
              }`}
            />
          </button>
        ))}
        <span className="ml-2 text-sm text-gray-500">
          {value} / {maxStars}
        </span>
      </div>
      
      {error && (
        <p className="text-red-500 text-sm mt-1">{error}</p>
      )}
    </div>
  );
}

// Usage in schema
const feedbackSchema: FormSchema<{ rating: number; comment: string }> = {
  id: 'feedback',
  defaultValues: { rating: 0, comment: '' },
  fields: {
    rating: {
      label: 'How would you rate your experience?',
      widget: RatingWidget,
      required: true,
      props: { maxStars: 5 },
      validate: (value) => value > 0 ? null : 'Please provide a rating',
    },
    comment: {
      label: 'Additional Comments',
      widget: TextArea,
    },
  },
};
```

---

## 12. Custom Layout

Creating specialized form layouts.

```tsx
// layouts/WizardLayout.tsx
import { LayoutProps } from '@your-org/forms';
import { useState } from 'react';
import { ChevronLeft, ChevronRight } from 'lucide-react';

interface WizardStep {
  title: string;
  fields: string[];
}

interface WizardLayoutProps<TValues> extends LayoutProps<TValues> {
  steps: WizardStep[];
}

export function createWizardLayout<TValues>(steps: WizardStep[]) {
  return function WizardLayout(props: LayoutProps<TValues>) {
    const { Field, values, errors, isSubmitting, validateFields, submit } = props;
    const [currentStep, setCurrentStep] = useState(0);
    
    const step = steps[currentStep];
    const isFirstStep = currentStep === 0;
    const isLastStep = currentStep === steps.length - 1;
    
    const handleNext = async () => {
      // Validate current step's fields
      const valid = await validateFields(step.fields as any[]);
      if (valid) {
        setCurrentStep((s) => s + 1);
      }
    };
    
    const handlePrev = () => {
      setCurrentStep((s) => s - 1);
    };
    
    const handleSubmit = async () => {
      const valid = await validateFields(step.fields as any[]);
      if (valid) {
        await submit();
      }
    };
    
    return (
      <div className="wizard-layout">
        {/* Progress indicator */}
        <div className="flex justify-between mb-8">
          {steps.map((s, i) => (
            <div
              key={i}
              className={`flex items-center ${
                i <= currentStep ? 'text-blue-600' : 'text-gray-400'
              }`}
            >
              <div
                className={`w-8 h-8 rounded-full flex items-center justify-center ${
                  i <= currentStep ? 'bg-blue-600 text-white' : 'bg-gray-200'
                }`}
              >
                {i + 1}
              </div>
              <span className="ml-2">{s.title}</span>
              {i < steps.length - 1 && (
                <div className="w-12 h-0.5 mx-4 bg-gray-200" />
              )}
            </div>
          ))}
        </div>
        
        {/* Current step fields */}
        <div className="space-y-6">
          <h2 className="text-xl font-semibold">{step.title}</h2>
          
          {step.fields.map((fieldName) => (
            <Field key={fieldName} name={fieldName as any} />
          ))}
        </div>
        
        {/* Navigation */}
        <div className="flex justify-between mt-8 pt-4 border-t">
          <button
            type="button"
            onClick={handlePrev}
            disabled={isFirstStep}
            className="flex items-center gap-2 px-4 py-2 disabled:opacity-50"
          >
            <ChevronLeft /> Previous
          </button>
          
          {isLastStep ? (
            <button
              type="button"
              onClick={handleSubmit}
              disabled={isSubmitting}
              className="px-6 py-2 bg-blue-600 text-white rounded"
            >
              {isSubmitting ? 'Submitting...' : 'Submit'}
            </button>
          ) : (
            <button
              type="button"
              onClick={handleNext}
              className="flex items-center gap-2 px-4 py-2 bg-blue-600 text-white rounded"
            >
              Next <ChevronRight />
            </button>
          )}
        </div>
      </div>
    );
  };
}

// Usage
const JobApplicationWizardLayout = createWizardLayout<JobApplicationValues>([
  { title: 'Personal Information', fields: ['firstName', 'lastName', 'email', 'phone'] },
  { title: 'Position Details', fields: ['positionId', 'salary', 'startDate'] },
  { title: 'Experience', fields: ['workExperience', 'education'] },
  { title: 'Documents', fields: ['resume', 'coverLetter'] },
  { title: 'Final Steps', fields: ['authorizedToWork', 'requiresSponsorship', 'agreeToTerms'] },
]);

// In component
<SchemaForm
  schema={jobApplicationSchema}
  layout={JobApplicationWizardLayout}
  onSubmit={handleSubmit}
/>
```

---

## Common Patterns

### Saving Draft on Change

```tsx
const schema: FormSchema<MyValues> = {
  // ... fields
};

function MyForm() {
  const handleDirtyChange = (isDirty: boolean) => {
    // Could show "unsaved changes" indicator
  };
  
  return (
    <SchemaForm
      schema={schema}
      layout={StackLayout}
      onSubmit={handleSubmit}
      onDirtyChange={handleDirtyChange}
    />
  );
}
```

### Confirming Before Leave

```tsx
function MyForm() {
  const [isDirty, setIsDirty] = useState(false);
  
  // React Router example
  usePrompt(
    'You have unsaved changes. Are you sure you want to leave?',
    isDirty
  );
  
  return (
    <SchemaForm
      schema={schema}
      layout={StackLayout}
      onSubmit={handleSubmit}
      onDirtyChange={setIsDirty}
    />
  );
}
```

### Server-Side Errors

```tsx
async function handleSubmit(values: MyValues) {
  try {
    await api.save(values);
  } catch (err) {
    if (err.code === 'VALIDATION_ERROR') {
      // Return field errors to show on form
      return {
        fieldErrors: err.fields, // { email: 'Already exists' }
        formError: err.message,
      };
    }
    throw err;
  }
}
```

### Computed/Derived Fields

For fields that are purely computed and shouldn't be in the schema:

```tsx
// Don't include `total` in defaultValues or fields
// Calculate it in the layout or a display component

function OrderLayout(props: LayoutProps<OrderValues>) {
  const { values, Field } = props;
  
  const total = values.items.reduce((sum, item) => sum + item.price * item.qty, 0);
  
  return (
    <div>
      <Field name="items" />
      <div className="text-xl font-bold">
        Total: ${total.toFixed(2)}
      </div>
    </div>
  );
}
```

---

## 13. Draft Support (Save & Resume)

Complete example showing draft hydration and dehydration.

```tsx
// schemas/application-form.ts
import { FormSchema } from '@your-org/forms';

interface ApplicationValues {
  personalInfo: {
    name: string;
    email: string;
  };
  companyId: string;
  companyName: string;  // Derived from companyId
  position: string;
  coverLetter: string;
}

export const applicationSchema: FormSchema<ApplicationValues> = {
  id: 'job-application',
  
  // HYDRATION: Load draft or fresh form
  defaultValues: async (ctx) => {
    const draftKey = `draft-${ctx.params?.jobId}`;
    
    // Try localStorage first
    const savedDraft = localStorage.getItem(draftKey);
    if (savedDraft) {
      try {
        return JSON.parse(savedDraft);
      } catch {
        // Invalid draft, continue to defaults
      }
    }
    
    // Or fetch from API
    if (ctx.params?.draftId) {
      const draft = await api.getDraft(ctx.params.draftId);
      if (draft) return draft;
    }
    
    // Fresh form
    return {
      personalInfo: { name: '', email: '' },
      companyId: ctx.params?.companyId ?? '',
      companyName: '',
      position: '',
      coverLetter: '',
    };
  },
  
  fields: {
    personalInfo: {
      label: 'Personal Information',
      widget: PersonalInfoWidget,
      required: true,
    },
    
    companyId: {
      label: 'Company',
      widget: CompanySelect,
      required: true,
    },
    
    // DERIVED: Works on draft load!
    // If draft has companyId, name auto-populates
    companyName: {
      widget: TextInput,
      disabled: true,
      dependsOn: ['companyId'],
      deriveValue: async (values) => {
        if (!values.companyId) return '';
        const company = await fetchCompany(values.companyId);
        return company.name;
      },
    },
    
    position: {
      label: 'Position',
      widget: TextInput,
      required: true,
    },
    
    coverLetter: {
      label: 'Cover Letter',
      widget: TextArea,
      props: { rows: 10 },
    },
  },
};
```

```tsx
// components/ApplicationForm.tsx
import { useRef, useCallback } from 'react';
import { SchemaForm, SchemaFormHandle } from '@your-org/forms';
import { applicationSchema } from '../schemas/application-form';
import { useDebouncedCallback } from 'use-debounce';

interface Props {
  jobId: string;
  draftId?: string;
}

export function ApplicationForm({ jobId, draftId }: Props) {
  const formRef = useRef<SchemaFormHandle<ApplicationValues>>(null);
  const draftKey = `draft-${jobId}`;
  
  // DEHYDRATION: Debounced auto-save
  const saveDraft = useDebouncedCallback(
    async (values: ApplicationValues) => {
      // Save to localStorage (instant)
      localStorage.setItem(draftKey, JSON.stringify(values));
      
      // Save to API (background)
      try {
        await api.saveDraft(jobId, values);
      } catch (err) {
        console.error('Failed to save draft to server', err);
      }
    },
    2000 // 2 second debounce
  );
  
  // Manual save button
  const handleManualSave = useCallback(async () => {
    const values = formRef.current?.getValues();
    if (!values) return;
    
    localStorage.setItem(draftKey, JSON.stringify(values));
    await api.saveDraft(jobId, values);
    toast.success('Draft saved!');
  }, [jobId, draftKey]);
  
  // Final submit
  const handleSubmit = async (values: ApplicationValues) => {
    await api.submitApplication(jobId, values);
    
    // Clean up drafts
    localStorage.removeItem(draftKey);
    await api.deleteDraft(jobId);
    
    toast.success('Application submitted!');
    router.push('/applications');
  };
  
  return (
    <div>
      <SchemaForm
        schema={applicationSchema}
        layout={StackLayout}
        formRef={formRef}
        params={{ jobId, draftId }}
        onValuesChange={(values, meta) => {
          // Only auto-save on user changes, not initial load or derived values
          if (meta.source === 'user') {
            saveDraft(values);
          }
        }}
        onSubmit={handleSubmit}
      />
      
      <div className="flex gap-4 mt-4">
        <button onClick={handleManualSave}>
          Save Draft
        </button>
        <button onClick={() => formRef.current?.submit()}>
          Submit Application
        </button>
      </div>
    </div>
  );
}
```

### Key Points

| Concern | How It's Handled |
|---------|------------------|
| Hydration | `defaultValues` async function |
| Dehydration | `onValuesChange` + `formRef.getValues()` |
| Derived fields on load | `deriveValue` runs automatically |
| Avoid saving on load | Check `meta.source === 'user'` |
| Multiple storage backends | Your code decides (localStorage, API, etc.) |
| Debouncing | Your code handles it (useDebouncedCallback) |
