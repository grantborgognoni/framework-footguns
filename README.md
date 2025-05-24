# Framework Footguns

![Work in Progress][wip-badge]

[wip-badge]: https://img.shields.io/badge/status-wip-yellow

Sometimes you hit the same issue more than once or want to make sure you're not missing something, then you do the research and you say, "oh, I forgot, that's right." This is a list of those. 

## Express

**#1 - Async errors aren’t auto handled**

In Express, unlike Fastify, NestJS,or function runtimes like Firebase Functions, an `async (req, res) => { … }` that throws will result in a hung request and/or crash your server instead of throwing a 500.

```js
// helper to catch rejects
const asyncHandler = fn => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

app.get('/data', asyncHandler(async (req, res) => {
  const data = await fetchData();
  res.json(data);
}));
```

## SvelteKit

**#2 - Protected Routes:** A single `+layout.server.ts` auth guard only runs once on initial server-render. Any client-side navigation to nested pages bypasses it unless there’s a server component (`+page.server.ts` or `+layout.server.ts`) on that route.

### What happens

1. **Initial load**: User requests `/account/dashboard`.

   * Server runs your `hooks.server.ts` (auth check), your `+layout.server.ts`, and the page component (`+page.svelte`), then returns fully rendered HTML.
2. **Client navigation**: User clicks a link to `/account/financials`.

   * No network request: the SvelteKit router swaps in the new page without hitting `hooks.server.ts` or rerunning your layout guard.

### How to enforce a server round-trip

> **Solution A:** Add a `+page.server.ts` to each protected route.
>
> * e.g. `/account/dashboard/+page.server.ts`, `/account/financials/+page.server.ts`, etc.
> * Client navigations fetch fresh `__data.json`, triggering `hooks.server.ts` and your guard.

> **Solution B:** Drop a blank `+layout.server.ts` at the root of your protected directory.
>
> * Placing it under `/src/routes/account/+layout.server.ts` makes *all* nested pages under `/account/*` server components and forces a server fetch (and rerunning `hooks.server.ts`) on every navigation.

> **Note:** In both approaches, your auth logic in `hooks.server.ts` runs on every request.

---

**#3 - Prevent leaking sensitive data across users via the cache**

Let's be real. This is a bit of an unrealistic issue...but here it is:

Pages that render user specific or sensitive data could expose that data to other users or sessions via the cache.

Example: A user could log out and then click the browser Back button to see a cached copy of a private page. Worse, a shared proxy might serve one user’s protected page to a different user requesting the same URL.

```ts
// src/routes/account/+layout.server.ts
export const load = async ({ setHeaders }) => {
  // Instruct browsers and intermediaries: do not store any part of this response
  setHeaders({ 'cache-control': 'no-store' });
  return {};
};
```
