# Calendar Memoization — Stable Reference with Ref Bridge

**Date:** 2025
**Category:** Performance Optimization

---

## Problem

The Schedule-X calendar library was **re-initializing on every React state change**. This happened because:

1. React compares objects and functions by **reference equality**
2. Every render created new callback instances and config objects
3. Schedule-X detected these as "changed" configuration and destroyed/recreated the calendar
4. Users experienced: flickering, lost scroll position, interrupted drag operations

During a typical drag-and-drop operation, the calendar could re-initialize 3-5 times, causing 150-200ms of perceived lag.

## Root Cause

React's rendering model creates new function references on each render:

```typescript
// This creates a NEW function on every render
const handleClick = (event) => { deleteEvent(event.id); };

// So this config object is also "new" on every render
const config = { callbacks: { onEventClick: handleClick } };
// Schedule-X sees config !== prevConfig → full re-init
```

## Solution: Three-Layer Memoization

### Layer 1 — useRef for Mutable Containers

Refs hold the latest function implementations without triggering re-renders:

```typescript
const deleteEventRef = useRef<((id: string) => Promise<void>) | null>(null);
const updateEventRef = useRef<((event: CalendarEvent) => Promise<void>) | null>(null);
```

### Layer 2 — useCallback with Empty Dependencies

Creates callback functions with **permanent stable identity**:

```typescript
const handleEventClick = useCallback(async (event, e) => {
    if (e.ctrlKey && deleteEventRef.current) {
        await deleteEventRef.current(event.id);
    }
}, []); // Empty deps = identity never changes
```

### Layer 3 — useMemo for Config

The calendar config object only depends on stable callbacks, so it's also stable:

```typescript
const calendarConfig = useMemo(() => ({
    callbacks: {
        onEventClick: handleEventClick,
        onEventUpdate: handleEventUpdate,
    }
}), [handleEventClick, handleEventUpdate]); // Stable deps = stable config
```

### Layer 4 — useEffect for Ref Syncing

Refs are updated with the latest implementations on every render:

```typescript
useEffect(() => {
    deleteEventRef.current = deleteEvent;
    updateEventRef.current = updateEvent;
}, [deleteEvent, updateEvent]);
```

## Why It Works

The key insight is **separating identity from implementation**:

- **Identity** (what React and Schedule-X track): stable callbacks that never change reference
- **Implementation** (what actually runs): latest functions stored in mutable refs

Schedule-X sees the same config object across renders, so it never re-initializes. But when a callback is invoked, it reads the current ref value, executing the most up-to-date logic.

## LRU Date Parse Cache

A complementary optimization for drag operations. During a single drag with Shift held (Ripple Move), the same date strings were parsed **hundreds of times** per mouse move event.

### Implementation

```typescript
class DateParseCache {
    private cache: Map<string, Temporal.ZonedDateTime>;
    private maxSize: number;

    constructor(maxSize = 500) {
        this.cache = new Map();
        this.maxSize = maxSize;
    }

    get(key: string): Temporal.ZonedDateTime | undefined {
        const value = this.cache.get(key);
        if (value) {
            // LRU promotion: move to end (most recently used)
            this.cache.delete(key);
            this.cache.set(key, value);
        }
        return value;
    }

    set(key: string, value: Temporal.ZonedDateTime): void {
        if (this.cache.has(key)) this.cache.delete(key);
        if (this.cache.size >= this.maxSize) {
            // Evict least recently used (first entry in Map)
            const firstKey = this.cache.keys().next().value;
            this.cache.delete(firstKey);
        }
        this.cache.set(key, value);
    }
}
```

### Usage

```typescript
// Global cache for common parsing across components
export const globalDateCache = new DateParseCache(500);

// Scoped caches for isolated operations (e.g., collision detection loop)
const scopedCache = createScopedCache();
const events = parseEventDates(filteredEvents, scopedCache);
```

### Format Normalization

The cache normalizes input formats before lookup, so `"2025-06-15 10:30"` and `"2025-06-15T10:30"` resolve to the same cache entry.

## Results

| Metric | Before | After |
|--------|--------|-------|
| Calendar re-inits during drag | 3-5 per drag | 0 |
| Date parse calls per 50ms | ~300 | ~100 |
| Drag response time | 150-200ms lag | Smooth (~40ms faster) |
| Scroll position during interaction | Lost on re-init | Preserved |

## Testing

```typescript
describe('DateParseCache', () => {
    it('evicts oldest entry when max size is reached', () => {
        const cache = new DateParseCache(3);
        cache.set('a', zdt); cache.set('b', zdt); cache.set('c', zdt);
        cache.set('d', zdt); // Should evict 'a'
        expect(cache.has('a')).toBe(false);
        expect(cache.has('d')).toBe(true);
    });

    it('promotes accessed entries (LRU behavior)', () => {
        cache.get('a'); // Promote 'a' to most recent
        cache.set('d', zdt); // Should evict 'b', not 'a'
        expect(cache.has('a')).toBe(true);
        expect(cache.has('b')).toBe(false);
    });

    it('normalizes space-separated format to T-separated', () => {
        const r1 = memoizedParseToZoned('2025-06-15 10:30', cache);
        const r2 = memoizedParseToZoned('2025-06-15T10:30', cache);
        expect(r2).toBe(r1); // Same cached object (reference equality)
    });
});
```
