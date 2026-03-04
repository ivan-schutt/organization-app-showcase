# SWR Caching Strategy — Shared Cache Across Components

**Date:** 2025
**Category:** Architecture / Data Access

---

## Problem

Before introducing SWR, each component independently fetched its data using `useEffect` + `useState`:

```typescript
// This pattern was repeated in every component that needed activities
const [activities, setActivities] = useState([]);
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);

useEffect(() => {
    const load = async () => {
        try {
            const data = await activitiesService.getActivities();
            setActivities(data);
        } catch (err) {
            setError(err);
        } finally {
            setLoading(false);
        }
    };
    load();
}, []);
```

**Issues:**
- **Duplicate requests**: If 3 components need activities, 3 API calls fire on mount
- **No cache sharing**: Each component has its own copy of the data
- **Stale data**: After creating an activity in one component, others don't know about it
- **Boilerplate**: Loading/error state management repeated everywhere
- **No auto-refresh**: Data goes stale without manual intervention

## Solution: Standardized SWR Hooks

### The Pattern

Every data type follows the same hook structure:

```typescript
import useSWR, { mutate as globalMutate } from 'swr';

const CACHE_KEY = 'activities';

const defaultConfig: SWRConfiguration = {
    refreshInterval: 5 * 60 * 1000,    // Auto-refresh every 5 minutes
    revalidateOnFocus: true,            // Refresh when tab regains focus
    revalidateOnReconnect: true,        // Refresh on network reconnect
    shouldRetryOnError: true,
    errorRetryCount: 3,
    dedupingInterval: 2000,             // Dedup identical requests within 2s
};

export function useActivities(config?: SWRConfiguration) {
    const { data, error, isLoading, mutate: localMutate } = useSWR(
        CACHE_KEY,
        () => activitiesService.getActivities(),
        { ...defaultConfig, ...config }
    );

    const createActivity = async (input: CreateActivityInput) => {
        const result = await activitiesService.create(input);
        if (result) await localMutate(); // Refresh cache
        return result;
    };

    const updateActivity = async (id: string, input: UpdateActivityInput) => {
        const result = await activitiesService.update(id, input);
        if (result) await localMutate();
        return result;
    };

    const deleteActivity = async (id: string) => {
        const result = await activitiesService.delete(id);
        if (result) await localMutate();
        return result;
    };

    return {
        activities: data || [],
        isLoading,
        error,
        refresh: () => localMutate(),
        mutate: localMutate,
        createActivity,
        updateActivity,
        deleteActivity,
    };
}

// External cache management
export async function clearActivitiesCache() {
    await globalMutate(CACHE_KEY, undefined, false);
}
```

### Applied To

| Hook | Cache Key | Data Source |
|------|-----------|-------------|
| `useActivities()` | `'activities'` | `activitiesService.getActivities()` |
| `useProjects()` | `'projects'` | Re-exports from `useActivities` (shared cache) |
| `useCategories()` | `'categories'` | `categoriesService.getCategories()` |
| `useSettings()` | `'user-settings'` | `settingsService.getSettings()` |
| `useChat()` | `'chat-messages/{id}'` | `chatService.getMessages(id)` |

## Cache Lifecycle

```
Request #1 (ComponentA mounts)
  → Cache miss → API call (~150ms) → Store in global cache → Return data

Request #2 (ComponentB mounts, < 2s later)
  → Cache hit (dedup) → Return cached data instantly (~2ms) → No API call

Request #3 (ComponentC mounts, > 2s later)
  → Cache hit (stale) → Return cached data → Background revalidation

Mutation (user creates activity)
  → Service call → mutate() → All 3 components receive fresh data

5 minutes pass (any consumer mounted)
  → Auto-refresh → Background fetch → Update if data changed

User switches tab and comes back
  → Revalidate on focus → Background fetch → Update if data changed
```

## Global SWR Configuration

Applied via `SWRProvider` at the root of the component tree:

```typescript
const swrConfig: SWRConfiguration = {
    refreshInterval: 5 * 60 * 1000,
    revalidateOnFocus: true,
    revalidateOnReconnect: true,
    dedupingInterval: 2000,
    shouldRetryOnError: true,
    errorRetryCount: 3,
    onErrorRetry: (error, key, config, revalidate, { retryCount }) => {
        if (retryCount >= 3) return; // Give up after 3 retries
        const delay = Math.min(1000 * Math.pow(2, retryCount), 10000);
        setTimeout(() => revalidate({ retryCount }), delay);
    },
    keepPreviousData: true, // Show stale data while fetching new
};
```

**Retry strategy**: Exponential backoff — 1s, 2s, 4s (capped at 10s), then stop.

## Results

| Metric | Before (useEffect) | After (SWR) |
|--------|-------------------|-------------|
| API calls on page with 3 consumers | 3 | 1 |
| Cache sharing | None | Automatic via cache key |
| Stale data after mutation | Until manual refresh | Instant update |
| Loading state boilerplate | ~10 lines per component | 0 (built into hook) |
| Auto-refresh | None | Every 5 min + on focus |
| Error retry | None | 3 attempts with backoff |

## Data Access Decision Framework

The application uses two data access patterns depending on the use case:

**SWR hooks** (read-heavy, multi-consumer):
- Activities, Projects, Categories, Settings, Chat messages
- Benefits: caching, deduplication, auto-revalidation, shared state

**Direct service calls** (write-heavy, single-consumer):
- Pomodoro timer state, calendar event drag-and-drop
- Benefits: immediate control, no cache overhead for rapidly changing state
