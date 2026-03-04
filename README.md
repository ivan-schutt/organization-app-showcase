# Organization App

> This is a public showcase of a private repository. Here you'll find architecture details, optimization case studies, and demos of the application. 

I love modular and easy options for calendar activities, and I constantly use a Pomodoro timer to track sessions. I wanted to be able to automatically set intervals and easily build my weekly schedule calculated by sessions. This would let me create a strategy based on time usage and, in the future, make modifications based on statistics. That was the main idea of this application: to easily set up time dedicated to an activity, and then easily schedule my days using blocks based on that time and my usual focus sessions.

The main idea on this is that the blocks available on the calendar to drag and drop are based on the activities set up on the application, dividing time bt sessions and including the time + breaks. This way you have a real-time view that includes resting time.

The application is a productivity suite built with **Next.js 16**, **TypeScript**, **Tailwind CSS 4**, and **Supabase**. It combines task/project management, an interactive calendar with intelligent drag-and-drop, a Pomodoro timer with analytics, an AI-powered chatbot, and integrations with Notion and Google Calendar.

I am currently focusing on the functionality and optimization of the application. After this I will focus on art, styling and user experience.



---
## Demo

![Demo](./images/Gif%201.gif)

![Chatbot](./images/Page%201.png)
![Activities](./images/Page%202.png)
![Projects](./images/Page%203.png)
![Categories](./images/Page%204.png)
![Calendar](./images/Page%205.png)
![Pomodoro](./images/Page%206.png)
![Statistics](./images/Page%207.png)



---

## Features

### AI Chatbot (Home Page)
- Conversational assistant powered by **Groq API** (Llama 3.3-70B) with **function calling**
- 11 integrated tools: create/query events, activities, projects, categories, Notion data, and Pomodoro sessions
- Multi-turn conversations with tool chaining
- Chat history persisted in Supabase

### Interactive Calendar
- Built on **Schedule-X** with drag-and-drop plugins
- **Magnetic Snap** — events auto-adhere to nearby events (10-min threshold)
- **Smart Stacking** — collision detection prevents unwanted overlaps
- **15-min Grid** — auto-rounding when no snap applies
- **Ripple Move** — Shift+drag moves all subsequent events by the same delta
- **Multi-select** — rectangle selection for batch operations
- **Undo/Redo** — Ctrl+Z with snapshot-based history (supports complex multi-event operations)
- **Google Calendar Sync** — bidirectional import/export with color distinction
- **Time Blocks Sidebar** — drag activity blocks directly onto the calendar

### Activities, Projects & Categories
- Activities with configurable hours, time scope (day/week/month/year), and project assignment
- Projects organized by color-coded categories with status tracking
- Inline table editing with filtering

### Pomodoro Timer
- Customizable durations with auto-start breaks
- Auto-detects active calendar events for project assignment
- Sessions tracked in Supabase with timestamps and duration
- State persistence across browser refreshes

### Statistics & Analytics
- Performance by Project: assigned vs worked time, filterable by day/week/month
- Distribution charts, weekly activity bar charts, session history
- Time debt tracking with automatic recovery block generation

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Framework** | Next.js 16+ (App Router), React 19, TypeScript 5 |
| **Styling** | Tailwind CSS 4 |
| **Database** | Supabase (PostgreSQL + Auth + RLS) |
| **Calendar** | Schedule-X + drag-and-drop, events-service, current-time plugins |
| **AI** | Groq API (OpenAI-compatible, function calling) |
| **Caching** | SWR (stale-while-revalidate) |
| **Dates** | Temporal API (temporal-polyfill) |
| **Charts** | Recharts |
| **Integrations** | Notion API, Google Calendar API |
| **Testing** | Vitest |

---

Each layer has a single responsibility. Components never call Supabase directly — they go through hooks, which go through services.

### Project Structure

```
src/
├── app/                    # Next.js App Router (pages + API routes)
│   ├── page.tsx            # Home — AI Chatbot
│   ├── actividades/        # Activities management
│   ├── calendar/           # Calendar with drag-and-drop
│   ├── categories/         # Category management
│   ├── pomodoro/           # Pomodoro timer
│   ├── projects/           # Project management
│   ├── settings/           # User settings
│   ├── statistics/         # Charts & analytics
│   └── api/                # chat/, google-calendar/, notion/
├── components/             # Reusable UI + providers
├── hooks/                  # SWR hooks + calendar-specific hooks
├── services/               # Service Object pattern (CRUD + business logic)
├── lib/                    # Infrastructure (Supabase client, config, Notion, AI tools)
├── types/                  # Single source of truth for all TypeScript types
└── utils/                  # Pure utilities (date helpers, memoization)
```

### Provider Hierarchy

```
AuthProvider (Supabase session)
  └─ ToastProvider (notification system)
      └─ SettingsLoader (pre-load user config)
          └─ SWRProvider (global cache configuration)
              ├─ Page Content
              ├─ PomodoroGlobal (timer state)
              └─ ServiceWorkerRegistration
```

---

## Optimization Case Studies

A significant part of building this application was spent on **planning, refactoring, and optimizing** to ensure maintainability and performance. Below are two key examples.

### Case Study 1: Calendar Memoization — Stable Reference with Ref Bridge

**The Problem**

The Schedule-X calendar was experiencing **complete re-initialization on every React state change**:
- React compares objects by **reference equality**, not value
- Every render created new function instances and config objects
- Schedule-X detected these "new" objects and destroyed/recreated the entire calendar
- Result: visual flickering, lost scroll position, interrupted drag operations

**The Solution: Three-Layer Memoization**

The fix uses a pattern I call **"Stable Reference with Ref Bridge"**, which separates what React tracks (stable references) from what actually changes (mutable implementations via refs):

```
Layer 1 — useRef for mutable containers
  Stores the latest function implementations without triggering re-renders

Layer 2 — useCallback with empty dependencies
  Creates stable callback identities that never change reference

Layer 3 — useMemo for config objects
  Memoizes the calendar configuration, depending only on stable callbacks

Layer 4 — useEffect for ref syncing
  Keeps refs updated with actual implementations on every render
```

```typescript
// Layer 1: Mutable container
const deleteEventRef = useRef(null);

// Layer 2: Stable identity (never changes reference)
const handleEventClick = useCallback(async (event, e) => {
    if (e.ctrlKey && deleteEventRef.current) {
        await deleteEventRef.current(event.id);
    }
}, []); // Empty deps = stable forever

// Layer 3: Stable config (only depends on stable callbacks)
const calendarConfig = useMemo(() => ({
    callbacks: { onEventClick: handleEventClick }
}), [handleEventClick]);

// Layer 4: Sync refs with latest implementations
useEffect(() => {
    deleteEventRef.current = deleteEvent;
}, [deleteEvent]);
```

**Why it works:** The callbacks maintain their identity across renders, so Schedule-X never sees a "new" config. Meanwhile, the refs ensure the callbacks always execute the latest logic.

Additionally, an **LRU date parse cache** was implemented to avoid redundant Temporal API parsing during drag operations:

```typescript
// Before: ~300 date parses per 50ms mouse move during drag
// After: ~100 parses with cache hits on repeated strings

class DateParseCache {
    private cache: Map<string, Temporal.ZonedDateTime>;
    private maxSize: number;

    get(key: string) {
        const value = this.cache.get(key);
        if (value) {
            this.cache.delete(key);    // Remove
            this.cache.set(key, value); // Re-insert at end (most recent)
        }
        return value;
    }
}

export const globalDateCache = new DateParseCache(500);
```

**Results:**
- Calendar never re-initializes during interactions
- ~65% reduction in date parse calls
- ~40ms faster drag response
- Smooth, flicker-free experience

> Full documentation: [docs/calendar-memoization.md](./docs/calendar-memoization.md)

---

## Other Architectural Decisions

### Type Centralization
All shared TypeScript types live in `src/types/` as the single source of truth. Services import types from there and re-export for backward compatibility. This prevents circular dependencies and makes type updates easy.

### Service Object Pattern
Business logic is encapsulated in service objects (plain object literals, not classes) — no `this` binding issues, easy to mock in tests, clean separation from UI:

```typescript
export const activitiesService = {
    async getActivities(): Promise<ActivityWithProject[]> { ... },
    async create(input: CreateActivityInput): Promise<Activity | null> { ... },
    async update(id: string, input: UpdateActivityInput): Promise<Activity | null> { ... },
    async delete(id: string): Promise<boolean> { ... },
};
```

### Calendar Interaction Pipeline
Event placement follows a priority pipeline: **Magnetic Snap** (10-min threshold) > **Collision Push** (stacking) > **15-min Grid Rounding**. Each phase builds on the previous result.

### Undo/Redo System
Snapshot-based undo with support for complex operations (ripple move = multiple event changes in one action). Max 30 stack size to prevent memory bloat.

### Temporal API for Dates
All date operations use `temporal-polyfill` instead of native `Date`. Default timezone: `America/Argentina/Buenos_Aires`. An LRU cache prevents redundant parsing during high-frequency operations like drag-and-drop.

---

## Testing

Uses **Vitest** with focused unit tests on pure logic:

- **Memoization tests** — LRU eviction, cache hits, format normalization
- **Block generator tests** — activity-to-block splitting, edge cases, configurable durations


