# Sessions 7 and 8 lab handout: Lab 5, Movie Night serves its own API

Full-Stack Frameworks and Server-Side Rendering, EPITA Bachelor in Computer Science, L3

Lab 5 is split in two guided parts, one after each half of slides. In part 1, the Movie API of Lab 4 moves inside the app: the data becomes JSON files read by `lib/`, and you write the first Route Handlers, then a page that calls them from the browser. In part 2, the API accepts votes and reviews, validates what it receives, and the movie page shows the new reviews after a write. Session 8 ends with 30 minutes of work on your own project, for Milestone 2.

You follow the steps and write the code that is given. Each part ends with checks: a command or a page, with the result you should get. Lab 5 is not graded: ask the teacher when a check does not give the result.

| Session | Time | Block |
|---|---|---|
| 7 | +0:00 | Slides: Route Handlers, requests and responses, calling the API (90 min) |
| 7 | +1:30 | Break (10 min) |
| 7 | +1:40 | Lab 5, part 1: data files, three `GET` routes, `/movies-client` (80 min) |
| 8 | +0:00 | Slides: writing data, validation, forms and fresh pages (90 min) |
| 8 | +1:30 | Break (10 min) |
| 8 | +1:40 | Lab 5, part 2: votes and reviews (50 min) |
| 8 | +2:30 | Milestone 2 work time, on your own project (30 min) |

## Before you start

* `movie-night` in the state of the end of Lab 4, with your work committed: `git status` shows nothing left to commit
* The answers of Lab 4 in `NOTES.md`
* Next.js 16.4.2: `npx next --version` prints `Next.js v16.4.2`
* Two terminals in VS Code: one for the app, one for `curl`
* The link to the `lab-files` folder of session 4, given by the teacher

Three rules for the whole lab:

* Cache Components must stay off. `next.config.ts` must not contain `cacheComponents`.
* To see the strategy of a page or a route, run `npm run build`, then `npm start`. Stop `npm start` with Ctrl+C before each `npm run build`: both use the folder `.next`.
* On Windows, run the `curl` commands in Git Bash. In PowerShell, `curl` is another program, and the quotes of the JSON bodies do not work.

`curl` options used in this lab:

| Option | Meaning |
|---|---|
| `-i` | Print the status line and the headers, then the body |
| `-s` | Silent: no progress bar |
| `-X POST` | The method of the request (`GET` without it) |
| `-H "Content-Type: application/json"` | A header: the body is JSON |
| `-d '{"movieId":2}'` | The body. Single quotes around it, double quotes inside |

Put every URL that contains `?` or `&` in double quotes.

---

## Lab 5, part 1: Movie Night serves its own API (session 7, 80 min)

Goal: remove the second server of Lab 4, read the data from files on the server, expose it with three `GET` routes, and call one of them from the browser.

| Part | What you build | Time |
|---|---|---|
| A | Setup: copy the data files, delete the Movie API of Lab 4 | 10 min |
| B | The data layer reads the files, the pages use the route config | 20 min |
| C | Three `GET` routes: `/api/movies`, `/api/movies/[id]`, `/api/votes` | 25 min |
| D | `/movies-client` calls `/api/movies`, with a genre filter and four states | 15 min |
| | Questions in `NOTES.md`, then commit and push | 10 min |

### Part A: Setup (10 min)

**Step 1: copy the lab files.** As the README of `lab-files` says:

* `data/movies.json`, `data/votes.json`, `data/reviews.json` into a folder `data` at the root of the project
* the folder `data/seed/` into `data/`
* `scripts/reset-data.mjs` into a folder `scripts`

**Step 2: remove the Movie API of Lab 4.** Delete the folder `api` (with `api/server.mjs` and `api/data`). In `package.json`, replace the `api` script by:

```json
    "reset-data": "node scripts/reset-data.mjs"
```

**Step 3: remove the addresses of the API.** Delete the lines `MOVIES_API_URL` and `NEXT_PUBLIC_MOVIES_API_URL` from `.env.local`. If the file is now empty, delete it.

Check:

* `npm run reset-data` prints `data/ restored from data/seed/`
* `ls data` lists `movies.json`, `reviews.json`, `seed` and `votes.json`
* `git status` lists the deleted `api` files and the new `data` and `scripts` files, and not `.env.local`

### Part B: The data layer reads the files (20 min)

The functions keep their names, `getMovies`, `getMovie` and `getVotes`, but read the files of `data/` instead of calling port 4001. There is no `fetch` any more, so the `fetch` options of Lab 4 disappear, and each page chooses its strategy with the route config of session 6.

**Step 1: replace the whole `lib/movies.ts`:**

```ts
// Server-only: a Client Component that imports and uses this file fails the build.
// Type imports (import type { Movie }) are fine: they disappear at build.
// The pages and the Route Handlers call these functions on the server.
// In session 9, Prisma replaces the JSON files: only this file changes.
import "server-only";
import { readFile } from "node:fs/promises";
import path from "node:path";
import { cache } from "react";

export type Movie = {
  id: number;
  title: string;
  year: number;
  genre: string;
  minutes: number;
};

export type Vote = {
  movieId: number;
  count: number;
};

export type VoteResults = {
  votes: Vote[];
  countedAt: string; // ISO 8601
};

// A slow database, to keep the loading states visible: DEMO_LATENCY_MS=0 turns it off
const LATENCY_MS = Number(process.env.DEMO_LATENCY_MS ?? 800);

function wait(ms: number) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

// Read on every call: edit a file in data/ and the next call sees it
async function read<T>(file: string): Promise<T> {
  return JSON.parse(await readFile(path.join(process.cwd(), "data", file), "utf8"));
}

export async function getMovies(): Promise<Movie[]> {
  await wait(LATENCY_MS);
  return read<Movie[]>("movies.json");
}

// cache(): generateMetadata and the page read the file once per request
export const getMovie = cache(async (id: number): Promise<Movie | undefined> => {
  await wait(LATENCY_MS);
  return (await read<Movie[]>("movies.json")).find((movie) => movie.id === id);
});

export async function getVotes(): Promise<VoteResults> {
  await wait(LATENCY_MS);
  return { votes: await read<Vote[]>("votes.json"), countedAt: new Date().toISOString() };
}

// The id comes from the URL, so it is a string: "2" -> 2, "abc" or "2.5" -> null
export function parseId(value: string): number | null {
  return /^[1-9]\d*$/.test(value) ? Number(value) : null;
}
```

`getMovie` now takes a number. `parseId` turns the string of the URL into a number, or `null` when it is not a valid id.

**Step 2: the home page, SSR.** In `components/top-movies.tsx`, replace the comment and the `Promise.all` by:

```tsx
// Async Server Component: it calls the data layer directly, not /api/votes.
// The two reads do not depend on each other: start them together.
export async function TopMovies() {
  const [movies, { votes, countedAt }] = await Promise.all([getMovies(), getVotes()]);
```

In `app/page.tsx`, replace the comment above `export default function Home()` by these lines:

```tsx
// SSR: the data layer reads files, not fetch, so the strategy comes from the
// route config. Rendered on every request (ƒ in the build output).
export const dynamic = "force-dynamic";
```

**Step 3: `/movies`, ISR.** In `app/movies/page.tsx`, add above the component, and remove the options of `getMovies`:

```tsx
// ISR: prerendered at build time, then regenerated in the background at most
// every 10 seconds, after a request (○ with "10s" in the build output)
export const revalidate = 10;

export default async function MoviesPage() {
  const movies = await getMovies();
```

**Step 4: `/movies/[id]`, SSG.** In `app/movies/[id]/page.tsx`:

* import `parseId` too: `import { getMovie, getMovies, parseId } from "@/lib/movies";`
* replace the comment above `generateStaticParams` by `// SSG: runs at build time, one HTML page per movie (● in the build output)`, and call `getMovies()` without options
* in `generateMetadata` and in the page, replace the two lines that read the id and the movie by:

```tsx
  const id = parseId((await params).id);
  const movie = id === null ? undefined : await getMovie(id);
```

Keep `dynamicParams = false` and `if (!movie) notFound();`.

Check:

* `grep -rn "fetch\|cache:" app components lib` finds only `app/movies-client/page.tsx` (part D rewrites it) and the comment "not fetch" of `app/page.tsx`
* `npm run build` passes with no other terminal open, and prints:

```text
├ ƒ /
├ ○ /movies                   10s      1y
├ ○ /movies-client
└   /movies/[id]
  ├ ● /movies/1
```

* `npm start`, then `curl -s -o /dev/null -w "%{time_starttransfer} %{time_total}\n" http://localhost:3000/` prints about `0.06 0.8`: the two reads run together, 800 ms each
* `/movies/2` shows Inception, `/movies/99` and `/movies/abc` the not-found page

### Part C: Three `GET` routes (25 min)

**Step 1: the error helpers.** Create `lib/http.ts`:

```ts
// Helpers shared by the Route Handlers: one JSON shape for every error.
import "server-only";

export type ApiError = { error: string; fields?: Record<string, string> };

export function jsonError(status: number, error: string, fields?: Record<string, string>) {
  const body: ApiError = fields ? { error, fields } : { error };
  return Response.json(body, { status });
}

// Logs the real error on the server, sends a generic message to the client
export function serverError(err: unknown) {
  console.error(err);
  return jsonError(500, "Something went wrong on the server");
}
```

**Step 2: `GET /api/movies`, with a genre filter.** Create `app/api/movies/route.ts`:

```ts
import type { NextRequest } from "next/server";
import { jsonError, serverError } from "@/lib/http";
import { getMovies } from "@/lib/movies";

// GET /api/movies, GET /api/movies?genre=Sci-Fi
export async function GET(request: NextRequest) {
  // A string, or null when the parameter is missing
  const genre = request.nextUrl.searchParams.get("genre");
  if (genre !== null && genre.trim() === "") return jsonError(400, "genre cannot be empty");

  try {
    const movies = await getMovies();
    // Case-insensitive: ?genre=sci-fi works too
    const result = genre
      ? movies.filter((movie) => movie.genre.toLowerCase() === genre.toLowerCase())
      : movies;
    return Response.json(result);
  } catch (err) {
    return serverError(err);
  }
}
```

**Step 3: `GET /api/movies/[id]`.** Create `app/api/movies/[id]/route.ts`:

```ts
import type { NextRequest } from "next/server";
import { jsonError, serverError } from "@/lib/http";
import { getMovie, parseId } from "@/lib/movies";

// GET /api/movies/2. params is a Promise; RouteContext types it from the path.
export async function GET(_request: NextRequest, ctx: RouteContext<"/api/movies/[id]">) {
  const id = parseId((await ctx.params).id);
  // 400: the request itself is wrong. 404: it is fine, but there is no such movie.
  if (id === null) return jsonError(400, "The id must be a positive whole number");

  try {
    const movie = await getMovie(id);
    if (!movie) return jsonError(404, "Movie not found");
    return Response.json(movie);
  } catch (err) {
    return serverError(err);
  }
}
```

If the editor says that `RouteContext` is not found, run `npm run build` or `npm run dev` once: they generate the types.

**Step 4: `GET /api/votes`, on your own.** Create `app/api/votes/route.ts`. It exports a `GET` handler that answers `Response.json(await getVotes())`, inside a `try`, with `serverError` in the `catch`. It takes no parameter.

Check, after `npm run build` and `npm start`, in the second terminal:

* The build output lists `ƒ /api/movies`, `ƒ /api/movies/[id]` and `ƒ /api/votes`

| Command | Expected |
|---|---|
| `curl -i http://localhost:3000/api/movies` | `200`, `content-type: application/json`, six movies |
| `curl -s "http://localhost:3000/api/movies?genre=sci-fi"` | Inception and Interstellar |
| `curl -s "http://localhost:3000/api/movies?genre=Horror"` | `[]` |
| `curl -i "http://localhost:3000/api/movies?genre="` | `400`, `{"error":"genre cannot be empty"}` |
| `curl -i http://localhost:3000/api/movies/2` | `200`, Inception |
| `curl -i http://localhost:3000/api/movies/99` | `404`, `{"error":"Movie not found"}` |
| `curl -i http://localhost:3000/api/movies/abc` | `400`, `{"error":"The id must be a positive whole number"}` |
| `curl -s http://localhost:3000/api/votes` | `{"votes":[...],"countedAt":"..."}` |
| `curl -i -X POST http://localhost:3000/api/movies` | `405 Method Not Allowed`, empty body |

### Part D: `/movies-client` calls your API (15 min)

In Lab 4, this page called the Movie API on port 4001 from the browser. Now it calls `/api/movies`, on the same origin, and adds a genre filter. Replace the whole `app/movies-client/page.tsx`:

```tsx
"use client";

import { useEffect, useState } from "react";
import { MovieList } from "@/components/movie-list";
import type { Movie } from "@/lib/movies";

// "Horror" has no movie: it shows the empty state
const GENRES = ["Animation", "Comedy", "Horror", "Sci-Fi", "Thriller"];

type State =
  | { status: "loading" }
  | { status: "error"; message: string }
  | { status: "done"; movies: Movie[] };

export default function MoviesClientPage() {
  const [genre, setGenre] = useState("");
  const [state, setState] = useState<State>({ status: "loading" });

  useEffect(() => {
    // Ignore the answer of an older request if the genre changed meanwhile
    let ignore = false;
    async function load() {
      setState({ status: "loading" });
      try {
        // Same origin as the page: a relative URL, no CORS, no NEXT_PUBLIC_ variable
        const query = genre ? `?genre=${encodeURIComponent(genre)}` : "";
        const res = await fetch(`/api/movies${query}`);
        // fetch only throws when the request fails: check the status yourself
        if (!res.ok) {
          const body: { error?: string } = await res.json().catch(() => ({}));
          throw new Error(body.error ?? `The API answered ${res.status}`);
        }
        const movies: Movie[] = await res.json();
        if (!ignore) setState({ status: "done", movies });
      } catch (err) {
        if (!ignore)
          setState({
            status: "error",
            message: err instanceof Error ? err.message : "Network error",
          });
      }
    }
    load();
    return () => {
      ignore = true;
    };
  }, [genre]);

  return (
    <main className="mx-auto w-full max-w-2xl p-8">
      <h1 className="text-3xl font-bold">Movies (client-rendered)</h1>

      <label htmlFor="genre" className="mt-6 block font-medium">
        Genre
      </label>
      <select
        id="genre"
        value={genre}
        onChange={(e) => setGenre(e.target.value)}
        className="mt-1 rounded border px-3 py-2"
      >
        <option value="">All genres</option>
        {GENRES.map((g) => (
          <option key={g}>{g}</option>
        ))}
      </select>

      {state.status === "loading" && (
        <p className="mt-6" role="status">
          Loading movies...
        </p>
      )}
      {state.status === "error" && (
        <p className="mt-6 text-red-700 dark:text-red-400" role="alert">
          The movies could not be loaded: {state.message}
        </p>
      )}
      {state.status === "done" &&
        (state.movies.length === 0 ? (
          <p className="mt-6">No movie in this genre.</p>
        ) : (
          <MovieList movies={state.movies} />
        ))}
    </main>
  );
}
```

`import type { Movie }` from `lib/movies.ts` is fine in a Client Component: types disappear at build time.

Check, after `npm run build` and `npm start`:

* `curl -s http://localhost:3000/movies-client | grep -o "Loading movies..."` prints the text: the HTML has no movie
* In the browser, `/movies-client` shows "Loading movies...", then the six movies
* Choose Sci-Fi: Inception and Interstellar. Choose Horror: "No movie in this genre."
* DevTools, Network tab, filter Fetch/XHR: one request `movies?genre=Sci-Fi` per change, status 200
* `grep -rl "127.0.0.1:4001" .next/static` finds nothing

### Questions

Answer in `NOTES.md`, in a section "Lab 5, part 1", in a few sentences each:

1. Why does `/api/movies/abc` answer 400, and `/api/movies/99` answer 404? What does the difference tell the client?
2. The home page needs the votes. Why does `TopMovies` call `getVotes()` from `lib/`, and not `fetch("http://localhost:3000/api/votes")`?
3. In Lab 4, `/movies-client` needed `NEXT_PUBLIC_MOVIES_API_URL`, and the API had to allow its origin (CORS). Why not any more? And what does `curl -i -X POST http://localhost:3000/api/movies` answer, and why?

### Commit

```bash
git status
git add .
git commit -m "Lab 5 part 1: GET routes and /movies-client"
git push
```

### Lab 5, part 1 checkpoint

* `api/` and the `api` script are deleted, and the build passes with one terminal
* `lib/movies.ts` reads `data/`, starts with `import "server-only"`, and has `parseId`
* `/` is `ƒ`, `/movies` is `○` with `10s`, the movie pages are `●`
* `app/api/movies/route.ts`, `app/api/movies/[id]/route.ts` and `app/api/votes/route.ts` answer as in the table of part C
* `/movies-client` calls `/api/movies`, with loading, error, empty and success states
* The questions of part 1 are in `NOTES.md`

---

## Lab 5, part 2: Movie Night accepts votes and reviews (session 8, 50 min)

Goal: let the API change the data, check everything it receives, and show the new reviews on a static page after a write.

| Part | What you build | Time |
|---|---|---|
| E | `POST /api/votes`: read the body, validate it, 201, and a `VoteButton` | 15 min |
| F | The reviews API: you write `GET` and `POST`, then copy `PUT` and `DELETE` | 20 min |
| G | `ReviewForm` and `DeleteReviewButton` on the movie page | 10 min |
| | Questions in `NOTES.md`, then commit and push | 5 min |

Your API now writes into `data/`. Run `npm run reset-data` whenever your tests have filled the files, then `npm run build` again: the static pages keep the data of the last build.

### Part E: `POST /api/votes` (15 min)

**Step 1: read a body safely.** Add this function to `lib/http.ts`, above `serverError`:

```ts
// request.json() throws on a malformed body: turn that into a 400, not a 500
export async function readJson(
  request: Request,
): Promise<{ ok: true; body: unknown } | { ok: false; response: Response }> {
  try {
    return { ok: true, body: await request.json() };
  } catch {
    return { ok: false, response: jsonError(400, "The body must be valid JSON") };
  }
}
```

**Step 2: the validation.** Create `lib/validate.ts`. It has the check of a vote, used now, and the check of a review, used in part F:

```ts
// Hand-written validation of the bodies sent to the API.
// No "server-only": a form may reuse it, but the server always checks again.
import type { ReviewInput } from "@/lib/movies";

export type FieldErrors = Record<string, string>;

export type Validation<T> = { ok: true; data: T } | { ok: false; fields: FieldErrors };

// The body comes from any client: unknown until each field is checked
function asObject(body: unknown): Record<string, unknown> {
  return typeof body === "object" && body !== null && !Array.isArray(body)
    ? (body as Record<string, unknown>)
    : {};
}

function isPositiveInteger(value: unknown): value is number {
  return typeof value === "number" && Number.isInteger(value) && value > 0;
}

// POST /api/votes: { "movieId": 2 }
export function validateVoteInput(body: unknown): Validation<{ movieId: number }> {
  const { movieId } = asObject(body);
  if (!isPositiveInteger(movieId)) {
    return { ok: false, fields: { movieId: "A positive whole number, for example 2" } };
  }
  return { ok: true, data: { movieId } };
}

// POST and PUT of a review: { "author": "Sam", "rating": 4, "text": "..." }
export function validateReviewInput(body: unknown): Validation<ReviewInput> {
  const input = asObject(body);
  const fields: FieldErrors = {};

  const author = typeof input.author === "string" ? input.author.trim() : "";
  if (author.length < 2 || author.length > 40) fields.author = "Between 2 and 40 characters";

  const rating = input.rating;
  if (!isPositiveInteger(rating) || rating > 5) fields.rating = "A whole number from 1 to 5";

  const text = typeof input.text === "string" ? input.text.trim() : "";
  if (text.length < 10 || text.length > 500) fields.text = "Between 10 and 500 characters";

  if (Object.keys(fields).length > 0) return { ok: false, fields };
  return { ok: true, data: { author, rating: rating as number, text } };
}
```

**Step 3: the writes in `lib/movies.ts`.** Four changes:

1. The first import becomes `import { readFile, writeFile } from "node:fs/promises";`
2. Add the types of a review below `Vote`:

```ts
export type Review = {
  id: number;
  movieId: number;
  author: string;
  rating: number; // 1 to 5
  text: string;
  createdAt: string; // ISO 8601
};

export type ReviewInput = Pick<Review, "author" | "rating" | "text">;
```

3. Add `write` below `read`:

```ts
async function write(file: string, data: unknown): Promise<void> {
  await writeFile(path.join(process.cwd(), "data", file), JSON.stringify(data, null, 2) + "\n");
}
```

4. Add `addVote` at the end of the file:

```ts
// One more vote for a movie. undefined: no such movie
export async function addVote(movieId: number): Promise<Vote | undefined> {
  const movies = await read<Movie[]>("movies.json");
  if (!movies.some((movie) => movie.id === movieId)) return undefined;
  const votes = await read<Vote[]>("votes.json");
  let vote = votes.find((v) => v.movieId === movieId);
  if (!vote) {
    vote = { movieId, count: 0 };
    votes.push(vote);
  }
  vote.count += 1;
  await write("votes.json", votes);
  return vote;
}
```

**Step 4: the handler.** Replace the whole `app/api/votes/route.ts`:

```ts
import type { NextRequest } from "next/server";
import { jsonError, readJson, serverError } from "@/lib/http";
import { addVote, getVotes } from "@/lib/movies";
import { validateVoteInput } from "@/lib/validate";

// GET /api/votes: live data. Route Handlers are not cached by default.
export async function GET() {
  try {
    return Response.json(await getVotes());
  } catch (err) {
    return serverError(err);
  }
}

// POST /api/votes with { "movieId": 2 }: one more vote, 201 with the new count
export async function POST(request: NextRequest) {
  const parsed = await readJson(request);
  if (!parsed.ok) return parsed.response;

  const result = validateVoteInput(parsed.body);
  if (!result.ok) return jsonError(422, "Invalid vote", result.fields);

  try {
    const vote = await addVote(result.data.movieId);
    if (!vote) return jsonError(404, "Movie not found");
    // The home page is dynamic (ƒ): nothing to revalidate, it reads the votes on each request
    return Response.json(vote, { status: 201 });
  } catch (err) {
    return serverError(err);
  }
}
```

**Step 5: a button on the movie page.** Create `components/vote-button.tsx`:

```tsx
"use client"; // State and a click handler: it runs in the browser

import { useState } from "react";

// POST /api/votes. The movie page is static, so the count is not on it:
// the button shows the count sent back by the server.
export function VoteButton({ movieId }: { movieId: number }) {
  const [pending, setPending] = useState(false);
  const [message, setMessage] = useState<string | null>(null);

  async function vote() {
    setPending(true);
    setMessage(null);
    try {
      const res = await fetch("/api/votes", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ movieId }),
      });
      const body = await res.json();
      // 201: { movieId, count }. Anything else: { error }
      setMessage(res.status === 201 ? `Thanks! ${body.count} votes now.` : body.error);
    } catch {
      setMessage("Network error: try again.");
    } finally {
      setPending(false);
    }
  }

  return (
    <div className="mt-4">
      <button
        type="button"
        onClick={vote}
        disabled={pending}
        className="rounded bg-zinc-900 px-4 py-2 font-semibold text-white disabled:opacity-60 dark:bg-zinc-100 dark:text-zinc-900"
      >
        {pending ? "Voting..." : "Vote for tonight"}
      </button>
      {/* aria-live: screen readers announce the result */}
      <p aria-live="polite" className="mt-2 text-sm">
        {message}
      </p>
    </div>
  );
}
```

In `app/movies/[id]/page.tsx`, import it, and add `<VoteButton movieId={movie.id} />` below `<ShortlistButton movieTitle={movie.title} />`.

Check, after `npm run build` and `npm start` (`H` holds the header, to shorten the commands):

```bash
H="Content-Type: application/json"; U=http://localhost:3000/api/votes
curl -i -X POST -H "$H" -d '{"movieId":2}' $U
curl -i -X POST -H "$H" -d '{"movieId":"2"}' $U
curl -i -X POST -H "$H" -d '{"movieId":99}' $U
curl -i -X POST -H "$H" -d '{"movieId":' $U
```

| Body | Expected |
|---|---|
| `{"movieId":2}` | `201`, `{"movieId":2,"count":15}` |
| `{"movieId":"2"}` | `422`, `{"error":"Invalid vote","fields":{"movieId":"A positive whole number, for example 2"}}` |
| `{"movieId":99}` | `404`, `{"error":"Movie not found"}` |
| `{"movieId":` | `400`, `{"error":"The body must be valid JSON"}` |

* `/`: the top 3 shows the new count of Inception, without a new build
* `/movies/2`, "Vote for tonight": the button shows "Voting...", then "Thanks! 16 votes now."

### Part F: The reviews API (20 min)

**Step 1: the reviews in `lib/movies.ts`.** Add at the end of the file:

```ts
export async function getReviews(movieId: number): Promise<Review[]> {
  return (await read<Review[]>("reviews.json")).filter((r) => r.movieId === movieId);
}

export async function createReview(movieId: number, input: ReviewInput): Promise<Review> {
  const reviews = await read<Review[]>("reviews.json");
  const review: Review = {
    id: Math.max(0, ...reviews.map((r) => r.id)) + 1,
    movieId,
    ...input,
    createdAt: new Date().toISOString(),
  };
  await write("reviews.json", [...reviews, review]);
  return review;
}

// PUT replaces the author, the rating and the text. undefined: no such review for this movie
export async function updateReview(
  movieId: number,
  reviewId: number,
  input: ReviewInput,
): Promise<Review | undefined> {
  const reviews = await read<Review[]>("reviews.json");
  const review = reviews.find((r) => r.id === reviewId && r.movieId === movieId);
  if (!review) return undefined;
  Object.assign(review, input);
  await write("reviews.json", reviews);
  return review;
}

export async function deleteReview(movieId: number, reviewId: number): Promise<boolean> {
  const reviews = await read<Review[]>("reviews.json");
  const remaining = reviews.filter((r) => !(r.id === reviewId && r.movieId === movieId));
  if (remaining.length === reviews.length) return false;
  await write("reviews.json", remaining);
  return true;
}
```

**Step 2: `GET` and `POST`, on your own.** Create `app/api/movies/[id]/reviews/route.ts`, typed with `RouteContext<"/api/movies/[id]/reviews">`. Follow the order of the checks of the slides:

`GET /api/movies/2/reviews`:

1. `parseId` on `(await ctx.params).id`: `null` gives 400
2. `getMovie(id)`: no movie gives 404 `"Movie not found"`
3. 200 with `await getReviews(id)`

`POST /api/movies/2/reviews`, with `{ "author", "rating", "text" }`:

1. The id, as for `GET`: 400
2. `readJson(request)`: malformed JSON gives 400
3. `validateReviewInput(parsed.body)`: 422 `"Invalid review"` with `result.fields`
4. `getMovie(id)`: 404
5. `createReview(id, result.data)`, then `` revalidatePath(`/movies/${id}`) `` from `next/cache` (part G explains why)
6. 201 with the review, and the header `Location: /api/movies/2/reviews/<id of the review>`

Put steps 4 to 6 in a `try`, with `serverError` in the `catch`, as in `app/api/votes/route.ts`.

**Step 3: `PUT` and `DELETE`, to copy.** Create `app/api/movies/[id]/reviews/[reviewId]/route.ts`:

```ts
import { revalidatePath } from "next/cache";
import type { NextRequest } from "next/server";
import { jsonError, readJson, serverError } from "@/lib/http";
import { deleteReview, parseId, updateReview } from "@/lib/movies";
import { validateReviewInput } from "@/lib/validate";

type Context = RouteContext<"/api/movies/[id]/reviews/[reviewId]">;

// Both ids come from the URL: two strings to check
async function readIds(ctx: Context) {
  const { id, reviewId } = await ctx.params;
  return { movieId: parseId(id), reviewId: parseId(reviewId) };
}

// PUT /api/movies/2/reviews/1: replaces the author, the rating and the text
export async function PUT(request: NextRequest, ctx: Context) {
  const { movieId, reviewId } = await readIds(ctx);
  if (movieId === null || reviewId === null) {
    return jsonError(400, "The ids must be positive whole numbers");
  }

  const parsed = await readJson(request);
  if (!parsed.ok) return parsed.response;

  const result = validateReviewInput(parsed.body);
  if (!result.ok) return jsonError(422, "Invalid review", result.fields);

  try {
    const review = await updateReview(movieId, reviewId, result.data);
    if (!review) return jsonError(404, "Review not found");
    revalidatePath(`/movies/${movieId}`);
    return Response.json(review);
  } catch (err) {
    return serverError(err);
  }
}

// DELETE /api/movies/2/reviews/1: 204, an empty body
export async function DELETE(_request: NextRequest, ctx: Context) {
  const { movieId, reviewId } = await readIds(ctx);
  if (movieId === null || reviewId === null) {
    return jsonError(400, "The ids must be positive whole numbers");
  }
  try {
    if (!(await deleteReview(movieId, reviewId))) return jsonError(404, "Review not found");
    revalidatePath(`/movies/${movieId}`);
    return new Response(null, { status: 204 });
  } catch (err) {
    return serverError(err);
  }
}
```

Check, after `npm run reset-data`, `npm run build` and `npm start`:

```bash
H="Content-Type: application/json"; U=http://localhost:3000/api/movies
curl -i -X POST -H "$H" -d '{"author":"S","rating":7}' $U/2/reviews
curl -i -X POST -H "$H" -d '{"author":"Sam","rating":4,"text":"Great dream heist."}' $U/2/reviews
curl -s $U/2/reviews
curl -i -X PUT -H "$H" -d '{"author":"Sam","rating":5,"text":"Even better the second time."}' $U/2/reviews/1
curl -i -X DELETE $U/2/reviews/1
curl -i -X DELETE $U/2/reviews/1
curl -i -X PATCH $U/2/reviews/1
```

| Request | Expected |
|---|---|
| `POST` `{"author":"S","rating":7}` | `422`, three messages in `fields`: author, rating, text |
| `POST` a valid review | `201`, `location: /api/movies/2/reviews/1` |
| `GET .../2/reviews` | `200`, an array with the review |
| `PUT .../2/reviews/1` | `200`, the review with the new rating and text |
| `DELETE .../2/reviews/1` | `204`, no body |
| `DELETE` again | `404`, `{"error":"Review not found"}` |
| `PATCH .../2/reviews/1` | `405`: the file exports no `PATCH` |
| `POST .../abc/reviews` | `400`. `POST .../99/reviews` with a valid body: `404` |

### Part G: Reviews on the movie page (10 min)

**Step 1: the form.** Create `components/review-form.tsx`:

```tsx
"use client"; // A form with state and a submit handler: it runs in the browser

import { useRouter } from "next/navigation";
import { useState, type FormEvent } from "react";

type ApiError = { error: string; fields?: Record<string, string> };

const inputClass = "mt-1 block w-full rounded border px-3 py-2";

// POST /api/movies/[id]/reviews. On 201, router.refresh() asks the server for
// the page again: the handler called revalidatePath, so the list is fresh.
export function ReviewForm({ movieId }: { movieId: number }) {
  const router = useRouter();
  const [pending, setPending] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [fields, setFields] = useState<Record<string, string>>({});

  async function onSubmit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const formElement = e.currentTarget;
    const form = new FormData(formElement);
    setPending(true);
    setError(null);
    setFields({});
    try {
      const res = await fetch(`/api/movies/${movieId}/reviews`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          author: form.get("author"),
          // A <select> gives a string: the API expects a number
          rating: Number(form.get("rating")),
          text: form.get("text"),
        }),
      });
      if (res.status === 201) {
        formElement.reset();
        router.refresh();
        return;
      }
      const body: ApiError = await res.json();
      setError(body.error);
      setFields(body.fields ?? {});
    } catch {
      setError("Network error: try again.");
    } finally {
      setPending(false);
    }
  }

  // Links an input to its error message for screen readers
  function aria(name: string) {
    return {
      id: name,
      name,
      "aria-invalid": Boolean(fields[name]),
      "aria-describedby": fields[name] ? `${name}-error` : undefined,
    };
  }

  function fieldError(name: string) {
    return (
      fields[name] && (
        <p id={`${name}-error`} className="mt-1 text-sm text-red-700 dark:text-red-400">
          {fields[name]}
        </p>
      )
    );
  }

  // noValidate: skip the checks of the browser, to see the ones of the server
  return (
    <form onSubmit={onSubmit} noValidate className="mt-4 space-y-3">
      <div>
        <label htmlFor="author" className="block font-medium">
          Your name
        </label>
        <input {...aria("author")} className={inputClass} />
        {fieldError("author")}
      </div>
      <div>
        <label htmlFor="rating" className="block font-medium">
          Rating
        </label>
        <select {...aria("rating")} defaultValue="" className={inputClass}>
          <option value="" disabled>
            Choose a rating
          </option>
          {[5, 4, 3, 2, 1].map((n) => (
            <option key={n} value={n}>
              {n} / 5
            </option>
          ))}
        </select>
        {fieldError("rating")}
      </div>
      <div>
        <label htmlFor="text" className="block font-medium">
          Review
        </label>
        <textarea {...aria("text")} rows={3} className={inputClass} />
        {fieldError("text")}
      </div>
      <button
        type="submit"
        disabled={pending}
        className="rounded bg-zinc-900 px-4 py-2 font-semibold text-white disabled:opacity-60 dark:bg-zinc-100 dark:text-zinc-900"
      >
        {pending ? "Sending..." : "Post the review"}
      </button>
      <p role="alert" className="text-red-700 dark:text-red-400">
        {error}
      </p>
    </form>
  );
}
```

**Step 2: the delete button.** Create `components/delete-review-button.tsx`:

```tsx
"use client"; // A click handler: it runs in the browser

import { useRouter } from "next/navigation";
import { useState } from "react";

// DELETE /api/movies/[id]/reviews/[reviewId], then refresh the page
export function DeleteReviewButton({ movieId, reviewId }: { movieId: number; reviewId: number }) {
  const router = useRouter();
  const [pending, setPending] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function remove() {
    setPending(true);
    setError(null);
    try {
      const res = await fetch(`/api/movies/${movieId}/reviews/${reviewId}`, { method: "DELETE" });
      // 204 has no body: do not call res.json() on it
      if (res.status === 204) router.refresh();
      else setError(((await res.json()) as { error: string }).error);
    } catch {
      setError("Network error");
    } finally {
      setPending(false);
    }
  }

  return (
    <>
      <button
        type="button"
        onClick={remove}
        disabled={pending}
        className="text-sm underline disabled:opacity-60"
      >
        {pending ? "Deleting..." : "Delete"}
      </button>
      {error && (
        <span role="alert" className="ml-2 text-sm text-red-700 dark:text-red-400">
          {error}
        </span>
      )}
    </>
  );
}
```

**Step 3: the reviews on the page.** In `app/movies/[id]/page.tsx`:

* import `ReviewForm`, `DeleteReviewButton`, and `getReviews` from `@/lib/movies`
* below `if (!movie) notFound();`, read the reviews on the server:

```tsx
  // Read on the server, in the static page: a write calls revalidatePath to rebuild it
  const reviews = await getReviews(movie.id);
```

* below `<VoteButton movieId={movie.id} />`, add:

```tsx
      <section aria-labelledby="reviews" className="mt-8">
        <h2 id="reviews" className="text-xl font-semibold">
          Reviews ({reviews.length})
        </h2>
        {reviews.length === 0 ? (
          <p className="mt-2">No review yet. Be the first.</p>
        ) : (
          <ul className="mt-2 space-y-3">
            {reviews.map((review) => (
              <li key={review.id} className="rounded border p-3">
                <p className="font-semibold">
                  {review.author}, {review.rating} / 5
                </p>
                <p className="mt-1">{review.text}</p>
                <DeleteReviewButton movieId={movie.id} reviewId={review.id} />
              </li>
            ))}
          </ul>
        )}
        <ReviewForm movieId={movie.id} />
      </section>
```

Check, after `npm run reset-data`, `npm run build` and `npm start`:

* The movie pages are still `●`, and `/movies/2` shows "Reviews (0)" and "No review yet. Be the first."
* Submit the empty form: a message under each field, and "Invalid review"
* Post a valid review: "Sending...", then the form clears and the review appears, without a full reload
* Click "Delete": the review disappears

**Optional: see why `revalidatePath` is needed.** Comment out `revalidatePath` in `app/api/movies/[id]/reviews/route.ts`, then `npm run build` and `npm start`. Post a review with `curl` (part F), then:

* `data/reviews.json` contains the review
* `curl -s http://localhost:3000/movies/2 | grep -o "Great dream heist"` prints nothing, and `curl -sI http://localhost:3000/movies/2 | grep x-nextjs-cache` prints `HIT`

Put the line back, `npm run reset-data`, and build again.

### Questions

Answer in `NOTES.md`, in a section "Lab 5, part 2", in a few sentences each:

1. The form of `/movies/2` only sends a rating from 1 to 5. Why does the API still validate the rating? Give a command that sends a rating of 7 without the form.
2. `/movies/2` is static. What happens to a new review without `revalidatePath`, and what does `router.refresh()` do, and not do?
3. In session 9, Prisma replaces `data/*.json`. Which files of Movie Night change, and which do not?

### Commit

```bash
npm run reset-data
git status
git add .
git commit -m "Lab 5 part 2: votes and reviews"
git push
```

Reset the data before the commit: `data/` should hold the seed data, not your tests.

### Lab 5 checkpoint

* `lib/http.ts` has `jsonError`, `readJson` and `serverError`, and every handler uses them
* `POST /api/votes` and the four review handlers answer as in the tables of parts E and F
* `/movies/2` shows the reviews, the form shows the errors of the server, and a new review appears after a post
* `data/` holds the seed data, and the build passes
* The questions of the two parts are in `NOTES.md`, and everything is pushed

---

## Milestone 2 work time (session 8, 30 min)

On your own project now. Milestone 2 is due at the end of session 8: at least two API routes, one that reads and one that writes, called from Client Components, with proper loading and error handling.

* [ ] My two routes are written down: method, URL, body, status codes
* [ ] The data layer is in `lib/`, with `import "server-only"`. A JSON file in `data/` is fine until session 9
* [ ] The read route answers 200, and 400 or 404 for a bad input
* [ ] The write route validates its body: 400, 422, then 201, 200 or 204
* [ ] Every error has the same JSON shape, a 500 shows no details, and no response contains a secret or an internal field
* [ ] A Client Component calls each route, checks `res.ok`, and shows the loading, error and empty states
* [ ] After a write, the page shows the new data: state, `revalidatePath` or `router.refresh()`
* [ ] The `README.md` has a table of the routes, and one `curl` command per route that works
* [ ] No Server Component fetches its own API: pages call `lib/`
* [ ] `npm run build` passes, the work is pushed, and the tag `milestone-2` is on the last commit

```bash
git tag milestone-2
git push origin milestone-2
git log -1 --format="%h %s" milestone-2
```

The last command must show your last commit.

---

## When something goes wrong

| Symptom | Cause | Fix |
|---|---|---|
| `Conflicting route and page at /movies: route at /movies/route and page at /movies/page` | A `route.ts` in the same folder as a `page.tsx` | Put the route in `app/api/movies/route.ts` |
| A route answers 404 with the HTML not-found page | The file is not named `route.ts`, or the folder path differs from the URL | Name it `route.ts`, and check the folders: `app/api/movies/[id]/route.ts` for `/api/movies/2` |
| Build fails with `error TS2559` in `.next/types/validator.ts`, `has no properties in common with type 'RouteHandlerConfig<...>'` | The handler is exported in lower case, such as `get` | Export `GET`, `POST`, `PUT`, `DELETE`, in upper case |
| `Cannot find name 'RouteContext'` | The types are not generated yet | Run `npm run build`, or `npm run dev` once |
| A method answers `405 Method Not Allowed` with an empty body | The file does not export a function for this method | Export the handler, or call the right method with `-X` |
| A route answers 500 with an empty body, and the terminal prints `No response is returned from route handler` | A branch of the handler does not `return` its response | Add `return` before each `Response.json(...)` |
| A route answers 500 with an empty body, and the terminal prints the error | The handler throws and has no `try`: a file that cannot be read, a bug | Wrap the work in `try`, and return `serverError(err)` in the `catch` |
| A `POST` answers 500 for a body cut in the middle | `request.json()` threw, outside `readJson` | Read the body with `readJson`: it answers 400 |
| A `POST` with valid JSON answers 422 | The validation refused a field: read `fields` in the answer. `"2"` is a string, not a number | Send the right types: `-d '{"movieId":2}'` |
| `TypeError: Body is unusable: Body has already been read` | `request.json()` is called twice | Read the body once, and keep it in a variable |
| The terminal prints `Error occurred prerendering page`, `fetch failed` and `ECONNREFUSED 127.0.0.1:3000` | A Server Component fetches its own API during the build | Call the function of `lib/` instead |
| `zsh: no matches found` or a request without its query string | The URL has `?` or `&` without quotes | Put the URL in double quotes |
| `curl` sends nothing, or the JSON is broken, on Windows | PowerShell handles quotes differently | Run the commands in Git Bash |
| A Client Component shows "undefined" instead of the data | `res.ok` is not checked, and the body of an error is used as data | Check `res.ok` before reading the data, and show `{ error }` |
| `SyntaxError: Unexpected end of JSON input` after a delete | `res.json()` is called on a 204, which has no body | Check `res.status === 204` before reading the body |
| The list of `/movies-client` shows the movies of the previous genre | The answers of two requests arrived in the wrong order | Use the `ignore` flag of the `useEffect` |
| A button stays on "Voting..." after a network error | `setPending(false)` is not in `finally` | Move it to `finally` |
| A new review is saved in `data/reviews.json`, but `/movies/2` does not show it | `/movies/2` is static, and the handler does not call `revalidatePath` | Call `` revalidatePath(`/movies/${id}`) `` after the write, and `router.refresh()` in the form |
| After `npm run reset-data`, the pages still show test reviews | The static pages keep the data of the last build | Build again |
| `EACCES` or `EPERM` in the terminal when writing `data/` | The file is read-only, or open in another program | Close it, check its permissions, then `npm run reset-data` |

The tables of the handouts of sessions 1 to 6 still apply, for example for `params` without `await`, or the `server-only` errors.
