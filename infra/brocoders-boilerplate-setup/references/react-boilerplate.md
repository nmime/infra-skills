# Extensive React Boilerplate Reference

## Overview

Production-ready React/Next.js boilerplate with:
- Next.js 14 with App Router
- TypeScript
- Tailwind CSS + Shadcn/UI
- React Query for data fetching
- React Hook Form for forms
- Zod for validation
- i18n internationalization
- Authentication flows
- Dark mode support

## Project Structure

```
frontend/
├── src/
│   ├── app/               # Next.js App Router pages
│   │   ├── [locale]/      # Internationalized routes
│   │   │   ├── (auth)/    # Auth layout group
│   │   │   │   ├── login/
│   │   │   │   ├── register/
│   │   │   │   └── forgot-password/
│   │   │   ├── (main)/    # Main app layout group
│   │   │   │   ├── dashboard/
│   │   │   │   ├── profile/
│   │   │   │   └── settings/
│   │   │   └── layout.tsx
│   │   └── api/           # API routes
│   ├── components/        # Reusable components
│   │   ├── ui/            # Shadcn/UI components
│   │   └── forms/         # Form components
│   ├── hooks/             # Custom React hooks
│   ├── lib/               # Utility libraries
│   ├── services/          # API service functions
│   ├── stores/            # State management
│   └── types/             # TypeScript types
├── public/                # Static assets
├── messages/              # i18n translation files
└── tailwind.config.ts
```

## Key Commands

```bash
# Development
npm run dev              # Start dev server
npm run dev:turbo        # Start with Turbopack

# Build
npm run build            # Production build
npm run start            # Start production server
npm run analyze          # Bundle analyzer

# Testing
npm run test             # Run tests
npm run test:watch       # Watch mode
npm run test:coverage    # With coverage

# Code Quality
npm run lint             # ESLint
npm run lint:fix         # Fix lint issues
npm run format           # Prettier
npm run typecheck        # TypeScript check

# Components
npx shadcn-ui@latest add button  # Add Shadcn component
```

## API Service Pattern

```typescript
// src/services/api/users.ts
import { apiClient } from '@/lib/api-client';
import { User, UpdateUserDto } from '@/types/user';

export const usersApi = {
  getMe: () => apiClient.get<User>('/auth/me'),
  
  update: (id: string, data: UpdateUserDto) =>
    apiClient.patch<User>(`/users/${id}`, data),
    
  delete: (id: string) =>
    apiClient.delete(`/users/${id}`),
};
```

## React Query Usage

```typescript
// src/hooks/use-user.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { usersApi } from '@/services/api/users';

export function useCurrentUser() {
  return useQuery({
    queryKey: ['user', 'me'],
    queryFn: usersApi.getMe,
  });
}

export function useUpdateUser() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: ({ id, data }) => usersApi.update(id, data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['user'] });
    },
  });
}
```

## Form Pattern with React Hook Form + Zod

```typescript
// src/components/forms/profile-form.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const profileSchema = z.object({
  firstName: z.string().min(1, 'Required'),
  lastName: z.string().min(1, 'Required'),
  email: z.string().email('Invalid email'),
});

type ProfileFormData = z.infer<typeof profileSchema>;

export function ProfileForm() {
  const form = useForm<ProfileFormData>({
    resolver: zodResolver(profileSchema),
  });

  const onSubmit = (data: ProfileFormData) => {
    // Handle submit
  };

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {/* Form fields */}
    </form>
  );
}
```

## Adding New Page

```typescript
// src/app/[locale]/(main)/posts/page.tsx
import { getTranslations } from 'next-intl/server';

export async function generateMetadata({ params: { locale } }) {
  const t = await getTranslations({ locale, namespace: 'Posts' });
  return { title: t('title') };
}

export default function PostsPage() {
  return (
    <div className="container py-8">
      <h1 className="text-3xl font-bold">Posts</h1>
      {/* Content */}
    </div>
  );
}
```

## Authentication Context

```typescript
// Usage in components
import { useAuth } from '@/hooks/use-auth';

export function Header() {
  const { user, isAuthenticated, logout } = useAuth();
  
  if (!isAuthenticated) {
    return <LoginButton />;
  }
  
  return (
    <div>
      <span>Welcome, {user.firstName}</span>
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```

## i18n Translation

```typescript
// messages/en.json
{
  "Common": {
    "save": "Save",
    "cancel": "Cancel"
  },
  "Profile": {
    "title": "Profile Settings"
  }
}

// Usage in component
import { useTranslations } from 'next-intl';

export function ProfilePage() {
  const t = useTranslations('Profile');
  return <h1>{t('title')}</h1>;
}
```

## Theme Switching

```typescript
import { useTheme } from 'next-themes';

export function ThemeToggle() {
  const { theme, setTheme } = useTheme();
  
  return (
    <button onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}>
      Toggle Theme
    </button>
  );
}
```