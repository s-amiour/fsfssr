# CM02: Lab Reflection Notes

## Description

The following work features answering reflection questions and pinpointing notes regarding the two labs, covering Client and Server Components in [README.md](./README.md), for the [movie-night](./movie-night/) application.

### Lab 01: Questions

1. In which file are the `<html>` and `<body>` tags?

> The `<html>` and `<body>` tags are found in the `app/layout.tsx` file, which is used as a root wrapper of the elements used in the body tag.

2. Which file renders the page at `/`?

> The file that renders the home page is `app/page.tsx`. The layout defines the Root Layout (which is requried in Next.js) using `<html>` and `<body` tags.

3. Which folder is for images and other static files, and how do you reference them?

> The `public/` folder is dedicated to storing static (content that does not change) files. They are referenced using the Image element from the `next/image` component in Next.js. 

4. What is  `tsconfig.json` doing in your project?

> It sets the tsc compiler and language server for the project. It also sets referencing the root via the `@` symbol.

5. What does `@/*` mean in `tsconfig.json`? Write an import that uses it.

> In `tsconfig.json`, `@/*` defines a path alias mapped to the root directory ("./*" in a non-src project, or "./src/*" if using a src directory).

```typescript
import { Card } from "@/components/Card";
```

6. What does `app/layout.tsx` import from `next/font/google`, and what does Next.js do with it at build time?

> `app/layout.tsx` imports two functions, `Geist` and `Geist_Mono` from `next/font/google`. They are used to set the fonts for the app so that they are consistently used across the whole app. Next.js fetches the font files from Google Fonts and includes them in the build output.

7. What’s the difference between `layout.tsx` and `page.tsx`? 

> `layout.tsx` defines the shared UI across multiple routes within a segment subtree. It wraps child routes via the `children` prop, preserves state, and does not re-render upon navigation between sub-routes. 
> `page.tsx` defines the unique UI for a specific route segment. It is what makes a URL path publicly accessible.

8. Where would you put a navbar that appears on every page?

> In the root layout, it should be placed inside the `<body>` tag, adjacent to `{children}` prop.

9.  What is `globals.css` for, and where is it imported?

> holds site-wide styles, CSS variables, and more.

10.  What is `AGENTS.md`, and does your app need it to run?

> A documentation file placed in the repository root intended to provide standards specifically to AI coding agent. Not needed for runnability, as it is only a development asset.

---

### Lab 02: Steps

#### Step 5

In Step 5, The View Source browser feature shows what the server returned:

- Title is found: `<title>About | Movie Night</title>`
- Metadata tag is found: `<meta name="description" content="What Movie Night is and how it works"/>`

#### Step 6
In Step 6, we deliberately cause a break/error:

1. Delete a closing tag in app/page.tsx and save. Read the error overlay: which file and line does it point to? Fix it and watch the overlay disappear without a manual refresh.

> It points to the `app/page.tsx`.
> * Error overlay: When remove a closing tag, Next.js detects the JSX syntax error and displays an overlay pointing to the file and line.
> * Fast Refresh: After fix the tag and save, Next.js recompiles the changed code and updates the page autmoatically, so don’t need to refresh it.

2. Rename app/page.tsx to app/home.tsx. What happens to /? Rename it back.

> In Next.js App Router, `page.tsx` is the special filename that makes a folder a route. Renaming it to `home.tsx` means app no longer defines the / page, so / will usually show a 404. Renaming it back restores the route.

### Questions

1. In the page source, are the movie titles present for `/movies`? For `/movies-client`? Why?

> For `/movies` URL segment, the titles should be in the page source because the page is rendered on the server and prerendered into HTML. For `/movies-client` URL segment, it loads the movies in a client-side effect after the page loads, the titles won’t be in the initial page source; JS adds them later. This is because of a difference of where the rendering occurs.

2. Without the force-dynamic line of Part E, you run npm run build again tomorrow, then npm start, without changing anything else. What does the "Rendered at" timestamp of /movies show, and why?

> Without `force-dynamic`, `/movies` is statically generated during `npm run build`. Tomorrow’s build will show a timestamp from that build, not the time you visit the page. It stays the same across builds until you build again.

3. Part E made every request to /movies about one second slower. Give one situation where force-dynamic is still the right choice, and explain why.

> `force-dynamic` is right when the page must show fresh, request-specific data. For example, a user's watch history information. This is because in situations where content is susceptible to volatility, regenerating on each request helps handle this by re-rendering per sent request.

4. A classmate sees `[movies] rendered 6 movies` in the browser console and says: "So this code runs in the browser." What do you answer, and how can you prove it?

> I would inform the peer that a browser console message alone does not prove the code ran in the browser, as server-rendered code can still log in the terminal. To prove it, I would demonstrate having a Server Component (running Next.js) with a route having a logging line (`[test] This ran in the server.`). I would then run the application (`npm run dev`) and display the server console. If that doesn't convince my classmate, I would disable JavaScript in DevTools, reload the page, and show the console output.

5. Which version would you choose for a public movie catalogue that changes once a week, and why?

> For the nonvolatile movie catalogue, I would implement static generation, with a rebuild every week (incremental). This matches ISR (Incremental Static Generation) because the pattern features users being served statically until a user requests an updated page, in which all consequent visitors receive the concerned page's update. 
