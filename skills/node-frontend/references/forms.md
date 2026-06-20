# Node Frontend — Forms Reference (React Hook Form + Zod)

## Setup

```bash
npm install react-hook-form zod @hookform/resolvers
```

---

## Basic Form Pattern

```tsx
// src/pages/Login/LoginForm.tsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { Input } from '@/components/Input'
import { Button } from '@/components/Button'

const loginSchema = z.object({
  email: z.string().min(1, 'Email is required').email('Invalid email'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
  rememberMe: z.boolean().default(false),
})

type LoginFormValues = z.infer<typeof loginSchema>

export function LoginForm({ onSubmit }: { onSubmit: (data: LoginFormValues) => Promise<void> }) {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    setError,
  } = useForm<LoginFormValues>({
    resolver: zodResolver(loginSchema),
    defaultValues: { email: '', password: '', rememberMe: false },
  })

  const handleFormSubmit = async (data: LoginFormValues) => {
    try {
      await onSubmit(data)
    } catch (err) {
      setError('root', { message: 'Invalid email or password' })
    }
  }

  return (
    <form onSubmit={handleSubmit(handleFormSubmit)} noValidate>
      <Input
        label="Email"
        type="email"
        autoComplete="email"
        error={errors.email?.message}
        {...register('email')}
      />
      <Input
        label="Password"
        type="password"
        autoComplete="current-password"
        error={errors.password?.message}
        {...register('password')}
      />
      <label>
        <input type="checkbox" {...register('rememberMe')} />
        Remember me
      </label>
      {errors.root && (
        <p role="alert" className="text-red-600">{errors.root.message}</p>
      )}
      <Button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Signing in…' : 'Sign in'}
      </Button>
    </form>
  )
}
```

---

## Zod Schema Patterns

```ts
import { z } from 'zod'

// Reusable field schemas
const emailField = z.string().min(1, 'Required').email('Invalid email')
const passwordField = z.string().min(8, 'At least 8 characters')

// Object with cross-field validation
const registerSchema = z
  .object({
    email: emailField,
    password: passwordField,
    confirmPassword: z.string(),
    role: z.enum(['admin', 'user', 'viewer']),
    age: z.number({ coerce: true }).int().min(18, 'Must be 18 or older').optional(),
    tags: z.array(z.string()).min(1, 'At least one tag required'),
    address: z.object({
      street: z.string().min(1),
      city: z.string().min(1),
      postcode: z.string().regex(/^\d{5}$/, 'Invalid postcode'),
    }),
    metadata: z.record(z.string(), z.unknown()).optional(),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: 'Passwords do not match',
    path: ['confirmPassword'],
  })

type RegisterFormValues = z.infer<typeof registerSchema>

// Discriminated union for multi-step forms
const paymentSchema = z.discriminatedUnion('method', [
  z.object({ method: z.literal('card'), cardNumber: z.string().length(16) }),
  z.object({ method: z.literal('paypal'), paypalEmail: emailField }),
])
```

---

## Controller for Custom / Third-party Inputs

```tsx
import { Controller, useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import Select from 'react-select'

export function ProductForm() {
  const { control, handleSubmit } = useForm<FormValues>({
    resolver: zodResolver(schema),
  })

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Controller
        name="category"
        control={control}
        render={({ field, fieldState }) => (
          <>
            <Select
              {...field}
              options={categoryOptions}
              onChange={(opt) => field.onChange(opt?.value)}
              value={categoryOptions.find((o) => o.value === field.value) ?? null}
            />
            {fieldState.error && <p>{fieldState.error.message}</p>}
          </>
        )}
      />
    </form>
  )
}
```

---

## useFieldArray — Dynamic Lists

```tsx
import { useFieldArray, useForm } from 'react-hook-form'
import { z } from 'zod'

const schema = z.object({
  items: z
    .array(z.object({ name: z.string().min(1), qty: z.number({ coerce: true }).int().min(1) }))
    .min(1, 'Add at least one item'),
})
type FormValues = z.infer<typeof schema>

export function OrderForm() {
  const { register, control, handleSubmit, formState: { errors } } =
    useForm<FormValues>({ resolver: zodResolver(schema), defaultValues: { items: [{ name: '', qty: 1 }] } })

  const { fields, append, remove } = useFieldArray({ control, name: 'items' })

  return (
    <form onSubmit={handleSubmit(console.log)}>
      {fields.map((field, index) => (
        <div key={field.id}>
          <input
            {...register(`items.${index}.name`)}
            placeholder="Item name"
          />
          {errors.items?.[index]?.name && (
            <p>{errors.items[index].name?.message}</p>
          )}
          <input
            type="number"
            {...register(`items.${index}.qty`, { valueAsNumber: true })}
          />
          <button type="button" onClick={() => remove(index)}>Remove</button>
        </div>
      ))}
      {errors.items?.root && <p>{errors.items.root.message}</p>}
      <button type="button" onClick={() => append({ name: '', qty: 1 })}>Add item</button>
      <button type="submit">Submit</button>
    </form>
  )
}
```

---

## useWatch — Reactive Field Values

```tsx
import { useWatch } from 'react-hook-form'

function PaymentMethodFields({ control }: { control: Control<FormValues> }) {
  const method = useWatch({ control, name: 'paymentMethod' })

  return (
    <>
      {method === 'card' && <CardFields control={control} />}
      {method === 'paypal' && <PaypalFields control={control} />}
    </>
  )
}
```

---

## Form with TanStack Mutation

```tsx
export function CreateUserForm() {
  const createUser = useCreateUser()
  const navigate = useNavigate()

  const { register, handleSubmit, formState: { errors, isSubmitting } } =
    useForm<CreateUserValues>({ resolver: zodResolver(createUserSchema) })

  const onSubmit = async (data: CreateUserValues) => {
    const user = await createUser.mutateAsync(data)
    navigate(`/users/${user.id}`)
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* fields */}
      {createUser.isError && (
        <p role="alert">{createUser.error.message}</p>
      )}
      <Button type="submit" loading={isSubmitting || createUser.isPending}>
        Create
      </Button>
    </form>
  )
}
```

---

## Multi-step Form Pattern

```tsx
const STEPS = ['personal', 'address', 'payment'] as const
type Step = (typeof STEPS)[number]

export function MultiStepForm() {
  const [step, setStep] = useState<Step>('personal')
  const methods = useForm<FullFormValues>({
    resolver: zodResolver(fullSchema),
    mode: 'onTouched',
  })

  const next = async () => {
    const fieldsForStep: Record<Step, (keyof FullFormValues)[]> = {
      personal: ['firstName', 'lastName', 'email'],
      address: ['street', 'city', 'postcode'],
      payment: ['cardNumber', 'expiry', 'cvv'],
    }
    const valid = await methods.trigger(fieldsForStep[step])
    if (valid) setStep(STEPS[STEPS.indexOf(step) + 1]!)
  }

  return (
    <FormProvider {...methods}>
      <form onSubmit={methods.handleSubmit(onFinalSubmit)}>
        {step === 'personal' && <PersonalStep />}
        {step === 'address' && <AddressStep />}
        {step === 'payment' && <PaymentStep />}
        <div>
          {step !== 'personal' && (
            <button type="button" onClick={() => setStep(STEPS[STEPS.indexOf(step) - 1]!)}>
              Back
            </button>
          )}
          {step !== 'payment' ? (
            <button type="button" onClick={next}>Next</button>
          ) : (
            <button type="submit">Submit</button>
          )}
        </div>
      </form>
    </FormProvider>
  )
}
```
