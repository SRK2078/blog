---
title: "A Guide to Server-Side Rendering (SSR) with Vite and React.js"
seoTitle: "SSR with Vite and React: A Guide"
seoDescription: "Learn to implement Server-Side Rendering (SSR) with React.js and Vite for improved web performance and SEO"
datePublished: 2025-01-07T13:00:05.450Z
cuid: cm5mha5ze000209laeclo7pve
slug: a-guide-to-server-side-rendering-ssr-with-vite-and-reactjs
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1736255453305/5d8c01c4-0478-4570-a0d2-8a343c1c17b4.png
tags: js, javascript, reactjs, typescript, ssr-and-ssg

---

**Title: A Guide to Server-Side Rendering (SSR) with Vite and React.js**

---

### Introduction

Server-Side Rendering (SSR) has become an essential technique for enhancing the performance and user experience of modern web applications. By pre-rendering pages on the server, SSR can drastically improve page load speeds and make websites more SEO-friendly. In this guide, we’ll explore how to implement SSR with React.js and Vite, step by step.

---

### What is Server-Side Rendering (SSR)?

In traditional client-side rendering (CSR), when a user visits a webpage, they receive a bare HTML page which then loads JavaScript and other resources, like CSS files. Only after these resources are loaded does the content become visible to the user. While this approach works, it can create delays in page load time, especially for users on slow connections.

SSR addresses this issue by rendering the HTML on the server, sending a fully rendered page to the client, and allowing the user to see the content immediately without waiting for JavaScript to load. This offers several benefits:

- **Faster initial page load**: The user sees content more quickly because the server sends a complete HTML document.
- **Better SEO**: Since search engines can crawl the fully rendered HTML, the content is indexed more effectively.
- **Improved user experience**: The user can interact with content sooner, leading to higher engagement rates.

---

### Performance Considerations with SSR

While SSR reduces the time to the Largest Contentful Paint (LCP), it can potentially increase the Interaction to Next Paint (INP), which is the time it takes for the user to interact with the page. A poorly implemented SSR setup might result in users seeing content but being unable to interact with it until the necessary JavaScript has loaded. It’s important to strike a balance between fast rendering and interactive elements to ensure smooth user interactions.

---

### Steps to Set Up SSR in Vite and React.js

We’ll break down the implementation of SSR in a Vite and React.js project into a few manageable steps.

#### 1. Create a ClientApp Component

First, we need to create a `ClientApp.jsx` component to handle the client-side rendering logic.

```jsx
// ClientApp.jsx
import { hydrateRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import App from './App';

hydrateRoot(document.getElementById('root'), 
  <BrowserRouter>
    <App />
  </BrowserRouter>
);
```

This code imports `hydrateRoot` from `react-dom/client` and `BrowserRouter` from `react-router-dom` to ensure the React application is correctly rendered in the browser.

#### 2. Modify `index.html`

Next, we need to update the `index.html` file to load the `ClientApp.jsx` file and add a parsing token to facilitate streaming.

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Vite + React + SSR</title>
  </head>
  <body>
    <div id="root"><!--not rendered--></div>
    <script type="module" src="./src/ClientApp.jsx"></script>
  </body>
</html>
```

#### 3. Create a ServerApp Component

Now, we’ll create a `ServerApp.jsx` file for the server-side rendering logic. It will use `renderToPipeableStream` from `react-dom/server` to stream the HTML.

```jsx
// ServerApp.jsx
import { renderToPipeableStream } from 'react-dom/server';
import { StaticRouter } from 'react-router-dom/server';
import App from './App';

export default function render(url, opts) {
  const stream = renderToPipeableStream(
    <StaticRouter location={url}>
      <App />
    </StaticRouter>,
    opts
  );

  return stream;
}
```

#### 4. Update Build Scripts

To build both the client and server bundles, we need to update the build scripts.

```json
{
  "scripts": {
    "build:client": "vite build --outDir ../dist/client",
    "build:server": "vite build --outDir ../dist/server --ssr ServerApp.jsx",
    "build": "npm run build:client && npm run build:server",
    "start": "node server.js"
  },
  "type": "module"
}
```

#### 5. Set Up the Node.js Server

We need to set up an Express server in `server.js` to handle SSR and static assets.

```js
// server.js
import express from 'express';
import fs from 'fs';
import path from 'path';
import { fileURLToPath } from 'url';
import renderApp from './dist/server/ServerApp.js';

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const PORT = process.env.PORT || 3001;

const html = fs.readFileSync(path.resolve(__dirname, './dist/client/index.html')).toString();
const [head, tail] = html.split('<!--not rendered-->');

const app = express();

// Serve static assets
app.use('/assets', express.static(path.resolve(__dirname, './dist/client/assets')));

// Handle all other routes with SSR
app.use((req, res) => {
  res.write(head);

  const stream = renderApp(req.url, {
    onShellReady() {
      stream.pipe(res);
    },
    onShellError(err) {
      console.error(err);
      res.status(500).send('Internal Server Error');
    },
    onAllReady() {
      res.write(tail);
      res.end();
    },
    onError(err) {
      console.error(err);
    }
  });
});

app.listen(PORT, () => {
  console.log(`Server is running at http://localhost:${PORT}`);
});
```

---

### Conclusion

By following these steps, we can implement server-side rendering (SSR) with React.js and Vite to improve the performance of our web application. The key benefit of SSR is that it reduces load times by delivering a fully rendered HTML page to the user. By streamlining content delivery, we create a faster, more responsive user experience.

This guide covered setting up SSR with React.js using Vite, creating necessary components, building server and client bundles, and configuring an Express server to handle SSR. With these optimizations in place, your application will be ready for better performance, enhanced SEO, and improved user engagement.

---

*Ready to optimize your React app with SSR? Follow these steps and watch your app's performance soar!*