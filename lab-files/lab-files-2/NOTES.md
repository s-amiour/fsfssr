# Lab 3

### Reflection Questions

1. `NavLink` uses `usePathname`, a hook that only works in Client Components. Yet the page source of `/movies/2` already contains `aria-current="true"` on the Movies link. How is this possible?

>A Client Component is not only rendered only in the browser, but also on the server. This produces the first HTML, and the browser later hydrates that HTML. During this server render, `usePathname()` already knows the requested URL (`/movies/2`), so `NavLink` computes `isInSection = true` and the `aria-current="true"` attribute is written into the HTML. Here `/movies/2` is prerendered, so this happened once, at build time, for that URL. In the browser, the same hook then keeps the link up to date after each `<Link>` navigation.

2. Why does `generateStaticParams` return `String(movie.id)` and not `movie.id`? And why does `/movies/42` give a 404 in Part C without running the code of the page?

>URL segments are always strings: in `/movies/2`, `params.id` is `"2"`, not `2`. `generateStaticParams` must return the same shape as `params`, so each id must be a string. A number makes the build fail ("received number"). It also keeps `getMovie` simple: it compares `String(movie.id) === id`.

> With `dynamicParams = false`, the ids returned by `generateStaticParams` (`"1"` to `"6"`) are the only pages of `/movies/[id]`, all prerendered at build time. When `/movies/42` is requested, Next.js sees that `"42"` is not in that list and answers 404 straight from the router. It does not call the page, `generateMetadata` or `getMovie`, so there is no 1-second wait and no `notFound()` call. Because this happens before any rendering, the 404 is a real one, with the header and the not-found page in the HTML.

3. In Part D, why is the note kept between `/movies` and `/movies/2`, and lost when you go through `/about`?

>`app/movies/layout.tsx` wraps `/movies` and every page below it, including `/movies/2`. During a `<Link>` navigation between two pages of the same segment, Next.js only replaces the page part (`children`). The layout is not re-rendered from scratch, so its `<input>` stays mounted and keeps what was typed. `/about` is outside the `movies` segment, so going there unmounts the movies layout, and its state is lost. Coming back to `/movies` mounts a new, empty layout. Typing a URL in the address bar loads a whole new document, which resets everything.

4. In Part E, step 4, where is the real message of the error, and why does Next.js not send it to the browser?

>The real message is only in the terminal running `npm start` (the server log). The browser gets a generic message instead, plus a `digest`, an id also printed next to the real error in the log, so the two can be matched. In production, Next.js hides the messages of errors thrown in Server Components, because they can contain sensitive details: database names, file paths, internal URLs or secrets. Under `npm run dev` it shows the real message to help debugging.

5. Under `npm run dev`, "Loading the movies..." appears when you open `/movies/2`. Under `npm start` it does not, but it still appears on `/movies`. Why?

>`npm run dev` renders every page on every request, including `/movies/2`. Each render waits about 1 second in `getMovie`, so the loading state is shown meanwhile. Under `npm start`, `/movies/2` was prerendered at build time with `generateStaticParams`: its HTML and RSC payload already exist, so there is nothing to wait for and `loading.tsx` is skipped. `/movies` is still `force-dynamic`: the server renders it on each request and waits 1 second in `getMovies`, so the loading state still appears there.
