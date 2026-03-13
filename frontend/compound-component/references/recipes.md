# Compound Component Recipes

Quick-reference implementations for common component types.
Each recipe shows: sub-component breakdown, context shape, and consumer API.

---

## Tabs

**Sub-components**: `Tabs`, `Tabs.List`, `Tabs.Tab`, `Tabs.Panel`

**Context shape**:
```ts
{ activeTab: string | null; setActiveTab: (v: string) => void }
```

**Consumer API**:
```jsx
<Tabs defaultValue="a">
  <Tabs.List>
    <Tabs.Tab value="a">Alpha</Tabs.Tab>
    <Tabs.Tab value="b" disabled>Beta</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel value="a">Alpha content</Tabs.Panel>
  <Tabs.Panel value="b">Beta content</Tabs.Panel>
</Tabs>
```

---

## Accordion

**Sub-components**: `Accordion`, `Accordion.Item`, `Accordion.Trigger`, `Accordion.Content`

**Context shape** (root): `{ openItems: string[]; toggle: (v: string) => void; multiple: boolean }`
**Context shape** (item): `{ value: string; isOpen: boolean }`

**Consumer API**:
```jsx
// Single open (default)
<Accordion defaultValue="item-1">
  <Accordion.Item value="item-1">
    <Accordion.Trigger>What is this?</Accordion.Trigger>
    <Accordion.Content>It is a compound component.</Accordion.Content>
  </Accordion.Item>
  <Accordion.Item value="item-2">
    <Accordion.Trigger>Why use it?</Accordion.Trigger>
    <Accordion.Content>Clean consumer API.</Accordion.Content>
  </Accordion.Item>
</Accordion>

// Multiple open
<Accordion multiple defaultValue={['item-1', 'item-2']}>
  ...
</Accordion>
```

**Note**: `Accordion.Item` provides its own nested context (`ItemContext`) so `Trigger` and `Content` can find their own `isOpen` without needing to know the item value.

---

## Dropdown / Menu

**Sub-components**: `Menu`, `Menu.Trigger`, `Menu.List`, `Menu.Item`, `Menu.Separator`, `Menu.Label`

**Context shape**: `{ isOpen: boolean; close: () => void; toggle: () => void }`

**Consumer API**:
```jsx
<Menu>
  <Menu.Trigger>Actions</Menu.Trigger>
  <Menu.List>
    <Menu.Label>File</Menu.Label>
    <Menu.Item onSelect={() => save()}>Save</Menu.Item>
    <Menu.Item onSelect={() => saveAs()}>Save As...</Menu.Item>
    <Menu.Separator />
    <Menu.Item onSelect={() => del()} disabled>Delete</Menu.Item>
  </Menu.List>
</Menu>
```

**Implementation note**: Use a `useClickOutside` hook on `Menu.List` to close on outside clicks. Render `Menu.List` conditionally or with `hidden` — prefer conditional for performance.

---

## Modal / Dialog

**Sub-components**: `Modal`, `Modal.Trigger`, `Modal.Content`, `Modal.Title`, `Modal.Description`, `Modal.Close`

**Context shape**: `{ isOpen: boolean; open: () => void; close: () => void }`

**Consumer API**:
```jsx
<Modal>
  <Modal.Trigger>Open dialog</Modal.Trigger>
  <Modal.Content>
    <Modal.Title>Confirm deletion</Modal.Title>
    <Modal.Description>This action cannot be undone.</Modal.Description>
    <div>
      <Modal.Close>Cancel</Modal.Close>
      <button onClick={handleDelete}>Delete</button>
    </div>
  </Modal.Content>
</Modal>
```

**Implementation notes**:
- Render `Modal.Content` in a portal (`ReactDOM.createPortal(content, document.body)`)
- Add focus trap when open (see `references/accessibility.md`)
- Lock body scroll when open: `document.body.style.overflow = 'hidden'`
- `Modal.Close` calls `close()` from context

---

## Stepper / Wizard

**Sub-components**: `Stepper`, `Stepper.Step`, `Stepper.Nav`, `Stepper.Controls`

**Context shape**: `{ currentStep: number; totalSteps: number; next: () => void; prev: () => void; goTo: (n: number) => void; canGoNext: boolean; canGoPrev: boolean }`

**Consumer API**:
```jsx
<Stepper defaultStep={0} onComplete={handleSubmit}>
  <Stepper.Nav /> {/* renders step indicators */}
  <Stepper.Step index={0} label="Account">
    <AccountForm />
  </Stepper.Step>
  <Stepper.Step index={1} label="Profile">
    <ProfileForm />
  </Stepper.Step>
  <Stepper.Step index={2} label="Confirm">
    <ConfirmStep />
  </Stepper.Step>
  <Stepper.Controls /> {/* renders prev/next buttons */}
</Stepper>
```

---

## Card (with sections)

**Sub-components**: `Card`, `Card.Header`, `Card.Image`, `Card.Body`, `Card.Footer`, `Card.Actions`

**Context shape**: minimal or none — Card is usually layout-only, no shared state needed.

**Consumer API**:
```jsx
<Card>
  <Card.Image src="/photo.jpg" alt="Profile" />
  <Card.Header>
    <h3>Jane Doe</h3>
  </Card.Header>
  <Card.Body>
    <p>Software engineer at Acme Corp.</p>
  </Card.Body>
  <Card.Footer>
    <Card.Actions>
      <button>Follow</button>
      <button>Message</button>
    </Card.Actions>
  </Card.Footer>
</Card>
```

**Note**: Card may not need context at all — it's primarily a layout compound component. The value is the clean, semantic composition API, not implicit state sharing.

---

## Form (with field primitives)

**Sub-components**: `Form`, `Form.Field`, `Form.Label`, `Form.Input`, `Form.Textarea`, `Form.Error`, `Form.Submit`

**Context shape** (Form root): `{ onSubmit, errors, register }` (often wraps react-hook-form or similar)
**Context shape** (Field): `{ fieldId: string; name: string; error: string | undefined }`

**Consumer API**:
```jsx
<Form onSubmit={handleSubmit} errors={errors} register={register}>
  <Form.Field name="email">
    <Form.Label>Email address</Form.Label>
    <Form.Input type="email" placeholder="you@example.com" />
    <Form.Error /> {/* shows error for this field automatically */}
  </Form.Field>
  <Form.Field name="password">
    <Form.Label>Password</Form.Label>
    <Form.Input type="password" />
    <Form.Error />
  </Form.Field>
  <Form.Submit>Sign in</Form.Submit>
</Form>
```

**Note**: `Form.Label` and `Form.Error` auto-connect to the field via `Form.Field`'s context — `htmlFor` and `aria-describedby` are wired automatically using the `fieldId` from context.

---

## Select / Combobox

**Sub-components**: `Select`, `Select.Trigger`, `Select.Value`, `Select.List`, `Select.Option`, `Select.Separator`

**Context shape**: `{ value: string | null; displayValue: string | null; setSelected: (v: string, label: string) => void; isOpen: boolean; toggle: () => void; close: () => void }`

**Consumer API**:
```jsx
<Select defaultValue="us" onChange={setCountry}>
  <Select.Trigger>
    <Select.Value placeholder="Select country..." />
  </Select.Trigger>
  <Select.List>
    <Select.Option value="us">United States</Select.Option>
    <Select.Option value="uk">United Kingdom</Select.Option>
    <Select.Separator />
    <Select.Option value="other" disabled>Other</Select.Option>
  </Select.List>
</Select>
```