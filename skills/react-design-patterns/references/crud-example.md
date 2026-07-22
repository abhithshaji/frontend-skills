# Worked Example: CRUD Form

A complete example composing **Custom Hooks + Provider + Compound Components** into a reusable `<Form>` system, then a User CRUD screen built on it. Use this as the mental model for form-heavy features.

**Architecture:**

```
useForm (custom hook: all form state & logic)
   └─> Form (provider: puts form state into context)
        ├─> Form.Field / Form.Select / Form.Textarea (compound components: consume context)
        └─> Form.Buttons
UserCRUD (feature component: composes Form + list rendering + CRUD handlers)
```

## 1. The hook — `hooks/useForm.js`

All form state/logic lives here, reusable by any form UI.

```jsx
import { useState, useCallback } from "react";

export const useForm = (initialValues = {}, onSubmit, validate) => {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleChange = useCallback((name, value) => {
    setValues(prev => ({ ...prev, [name]: value }));
    // Clear the field's error as the user types.
    // Functional update keeps this callback dependency-free (stable reference).
    setErrors(prev => (prev[name] ? { ...prev, [name]: undefined } : prev));
  }, []);

  const handleBlur = useCallback(
    (name) => {
      setTouched(prev => ({ ...prev, [name]: true }));
      if (validate) {
        const fieldErrors = validate(values);
        if (fieldErrors[name]) {
          setErrors(prev => ({ ...prev, [name]: fieldErrors[name] }));
        }
      }
    },
    [values, validate]
  );

  const handleSubmit = useCallback(
    async (e) => {
      if (e) e.preventDefault();

      if (validate) {
        const validationErrors = validate(values);
        setErrors(validationErrors);
        if (Object.keys(validationErrors).length > 0) {
          // Mark all fields touched so every error becomes visible
          setTouched(Object.fromEntries(Object.keys(values).map(k => [k, true])));
          return;
        }
      }

      setIsSubmitting(true);
      try {
        await onSubmit(values);
      } catch (error) {
        console.error("Form submission error:", error);
      } finally {
        setIsSubmitting(false);
      }
    },
    [values, validate, onSubmit]
  );

  const resetForm = useCallback(() => {
    setValues(initialValues);
    setErrors({});
    setTouched({});
    setIsSubmitting(false);
  }, [initialValues]);

  const setFieldValue = useCallback(
    (name, value) => handleChange(name, value),
    [handleChange]
  );

  const setFieldError = useCallback((name, error) => {
    setErrors(prev => ({ ...prev, [name]: error }));
  }, []);

  return {
    values, errors, touched, isSubmitting,
    handleChange, handleBlur, handleSubmit,
    resetForm, setFieldValue, setFieldError,
  };
};
```

## 2. The context — `components/form/FormContext.js`

```jsx
import { createContext, useContext } from "react";

export const FormContext = createContext(null);

export const useFormContext = () => {
  const context = useContext(FormContext);
  if (!context) {
    throw new Error("Form components must be used within a <Form>");
  }
  return context;
};
```

## 3. The provider root — `components/form/Form.jsx`

```jsx
import { useForm } from "../../hooks/useForm";
import { FormContext } from "./FormContext";

export const Form = ({ initialValues = {}, onSubmit, validate, children }) => {
  const formState = useForm(initialValues, onSubmit, validate);

  return (
    <FormContext.Provider value={formState}>
      <form onSubmit={formState.handleSubmit} noValidate>
        {children}
      </form>
    </FormContext.Provider>
  );
};
```

## 4. A compound field — `components/form/FormField.jsx`

Fields read everything from context — no prop drilling.

```jsx
import { useFormContext } from "./FormContext";

export const FormField = ({ name, label, type = "text", placeholder, required = false }) => {
  const { values, errors, touched, handleChange, handleBlur } = useFormContext();
  const hasError = touched[name] && errors[name];

  return (
    <div className="form-field">
      {label && (
        <label htmlFor={name}>
          {label}
          {required && <span aria-hidden="true">*</span>}
        </label>
      )}
      <input
        id={name}
        name={name}
        type={type}
        value={values[name] || ""}
        onChange={(e) => handleChange(name, e.target.value)}
        onBlur={() => handleBlur(name)}
        placeholder={placeholder}
        aria-invalid={!!hasError}
        aria-describedby={hasError ? `${name}-error` : undefined}
      />
      {hasError && (
        <span id={`${name}-error`} role="alert">{errors[name]}</span>
      )}
    </div>
  );
};
```

Build `FormSelect`, `FormTextarea`, `FormButtons` the same way (consume `useFormContext`).

## 5. Assemble the compound API — `components/form/index.js`

```jsx
import { Form as FormRoot } from "./Form";
import { FormButtons } from "./FormButtons";
import { FormField } from "./FormField";
import { FormSelect } from "./FormSelect";
import { FormTextarea } from "./FormTextarea";

export const Form = Object.assign(FormRoot, {
  Field: FormField,
  Textarea: FormTextarea,
  Select: FormSelect,
  Buttons: FormButtons,
});

export { FormButtons, FormField, FormSelect, FormTextarea };
```

## 6. The feature — `UserCRUD.jsx`

```jsx
import { useState } from "react";
import { Form } from "../components/form";

const validateUser = (values) => {
  const errors = {};

  if (!values.name?.trim()) {
    errors.name = "Name is required";
  } else if (values.name.length < 3) {
    errors.name = "Name must be at least 3 characters";
  }

  if (!values.email?.trim()) {
    errors.email = "Email is required";
  } else if (!/^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i.test(values.email)) {
    errors.email = "Invalid email address";
  }

  if (!values.role) errors.role = "Role is required";

  return errors;
};

export const UserCRUD = () => {
  const [users, setUsers] = useState([
    { id: crypto.randomUUID(), name: "John Doe", email: "john@example.com", role: "admin" },
    { id: crypto.randomUUID(), name: "Jane Smith", email: "jane@example.com", role: "user" },
  ]);
  const [editingUser, setEditingUser] = useState(null);
  const [showForm, setShowForm] = useState(false);

  const roleOptions = [
    { value: "admin", label: "Administrator" },
    { value: "user", label: "User" },
    { value: "guest", label: "Guest" },
  ];

  // CREATE — id from randomUUID, not users.length (length reuses ids after deletes)
  const handleCreate = async (values) => {
    await api.createUser(values); // your API call
    setUsers(prev => [...prev, { id: crypto.randomUUID(), ...values }]);
    setShowForm(false);
  };

  // UPDATE
  const handleUpdate = async (values) => {
    await api.updateUser(editingUser.id, values);
    setUsers(prev =>
      prev.map(user => (user.id === editingUser.id ? { ...user, ...values } : user))
    );
    setEditingUser(null);
    setShowForm(false);
  };

  // DELETE
  const handleDelete = async (id) => {
    if (!window.confirm("Are you sure you want to delete this user?")) return;
    await api.deleteUser(id);
    setUsers(prev => prev.filter(user => user.id !== id));
  };

  const openEdit = (user) => { setEditingUser(user); setShowForm(true); };
  const openCreate = () => { setEditingUser(null); setShowForm(true); };
  const closeForm = () => { setEditingUser(null); setShowForm(false); };

  return (
    <div>
      <h2>User Management</h2>

      {!showForm && <button onClick={openCreate}>Add New User</button>}

      {showForm && (
        <div>
          <h3>{editingUser ? "Edit User" : "Create New User"}</h3>
          {/* key forces a fresh form (fresh initialValues) when target changes */}
          <Form
            key={editingUser?.id ?? "new"}
            initialValues={editingUser || { name: "", email: "", role: "" }}
            onSubmit={editingUser ? handleUpdate : handleCreate}
            validate={validateUser}
          >
            <Form.Field name="name" label="Full Name" required />
            <Form.Field name="email" label="Email Address" type="email" required />
            <Form.Select name="role" label="Role" options={roleOptions} required />
            <Form.Buttons
              submitText={editingUser ? "Update User" : "Create User"}
              showReset={!editingUser}
              onCancel={closeForm}
            />
          </Form>
        </div>
      )}

      <table>
        <thead>
          <tr><th>Name</th><th>Email</th><th>Role</th><th>Actions</th></tr>
        </thead>
        <tbody>
          {users.map(user => (
            <tr key={user.id}>
              <td>{user.name}</td>
              <td>{user.email}</td>
              <td>{user.role}</td>
              <td>
                <button onClick={() => openEdit(user)}>Edit</button>
                <button onClick={() => handleDelete(user.id)}>Delete</button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
};
```

## Takeaways

- **Layering:** logic (hook) → distribution (provider/context) → UI blocks (compound components) → feature (composition). Each layer is independently testable and reusable.
- The `key={editingUser?.id ?? "new"}` trick resets form state when switching between create/edit targets — cleaner than syncing `initialValues` with an effect.
- Generate ids with `crypto.randomUUID()` (or take server ids), never from array length.
- For production forms consider **React Hook Form + Zod** — same architecture, battle-tested; this pattern is for understanding and for cases where you need full control.
- Replace `alert()`/`window.confirm()` with toasts/dialog components in real apps; consider the Optimistic UI pattern ([async-patterns.md](async-patterns.md)) for snappier mutations.
