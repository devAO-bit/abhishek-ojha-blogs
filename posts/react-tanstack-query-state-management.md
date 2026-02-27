---
title: "Why I Replaced Redux with TanStack Query for Server State"
excerpt: "TanStack Query handles caching, background refetching, and optimistic updates out of the box. Here's how I migrated the Dev Project Planner from manual fetch logic to TQ."
tags:
  - React
  - TypeScript
  - TanStack Query
  - State Management
date: "2025-03-29"
featured: false
coverEmoji: "🔄"
---

Redux is great for global UI state — theme, sidebar open/closed, user session. But for server data (lists of tasks, project details, user profiles), it creates a mountain of boilerplate that TanStack Query dissolves entirely.

## The Problem with Rolling Your Own

Before TanStack Query, a typical data-fetching pattern looked like this:

```jsx
const [tasks, setTasks] = useState([]);
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);

useEffect(() => {
  fetch('/api/tasks')
    .then(r => r.json())
    .then(data => { setTasks(data); setLoading(false); })
    .catch(e => { setError(e); setLoading(false); });
}, []);
```

Multiply this across 20 components, add cache invalidation after mutations, and handle background refresh — it becomes unmaintainable fast.

## TanStack Query in Practice

```jsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// Fetching
const { data: tasks, isLoading, error } = useQuery({
  queryKey: ['tasks', projectId],
  queryFn: () => fetch(`/api/projects/${projectId}/tasks`).then(r => r.json()),
  staleTime: 30_000, // consider fresh for 30s
});

// Mutating with automatic cache invalidation
const qc = useQueryClient();
const createTask = useMutation({
  mutationFn: (newTask) => fetch('/api/tasks', { method: 'POST', body: JSON.stringify(newTask) }),
  onSuccess: () => qc.invalidateQueries({ queryKey: ['tasks', projectId] }),
});
```

## Key Wins

- **Automatic background refetch** when the tab regains focus
- **Deduplication** — 10 components subscribing to the same query key = 1 network request
- **Optimistic updates** for instant UI feedback before the server confirms
- **DevTools** — the TanStack Query devtools panel shows cache state in real time

## When to Keep Redux (or Zustand)

Don't replace everything. Keep a lightweight store (I now prefer Zustand over Redux) for:

- Auth state / user session
- UI state (modals, drawers, selected items)
- Multi-step form state

Server state → TanStack Query. UI state → Zustand. That's the stack I ship with now.
