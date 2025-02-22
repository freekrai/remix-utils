# Remix Utils

This package contains simple utility functions to use with [Remix.run](https://remix.run).

## Installation

```bash
npm install remix-utils remix @remix-run/node @remix-run/react react
```

## API Reference

### promiseHash

The `promiseHash` function is not directly related to Remix but it's a useful function when working with loaders and actions.

This function is an object version of `Promise.all` which lets you pass an object with promises and get an object with the same keys with the resolved values.

```ts
export let loader: LoaderFunction = async ({ request }) => {
  return json(
    promiseHash({
      user: getUser(request),
      posts: getPosts(request),
    })
  );
};
```

You can use nested `promiseHash` to get a nested object with resolved values.

```ts
export let loader: LoaderFunction = async ({ request }) => {
  return json(
    promiseHash({
      user: getUser(request),
      posts: promiseHash({
        list: getPosts(request),
        comments: promiseHash({
          list: getComments(request),
          likes: getLikes(request),
        }),
      }),
    })
  );
};
```

### cacheAssets

This function lets you easily cache inside the [browser's Cache Storage](https://developer.mozilla.org/en-US/docs/Web/API/CacheStorage) every JS file built by Remix.

To use it, open your `entry.client` file and add this:

```ts
import { cacheAssets } from "remix-utils";

cacheAssets().catch((error) => {
  // do something with the error, or not
});
```

The function receives an optional options object with two options:

- `cacheName` is the name of the [Cache object](https://developer.mozilla.org/en-US/docs/Web/API/Cache) to use, the default value is `assets`.
- `buildPath` is the pathname prefix for all Remix built assets, the default value is `/build/` which is the default build path of Remix itself.

It's important that if you changed your build path in `remix.config.js` you pass the same value to `cacheAssets` or it will not find your JS files.

The `cacheName` can be left as is unless you're adding a Service Worker to your app and want to share the cache.

```ts
cacheAssests({ cacheName: "assets", buildPath: "/build/" }).catch((error) => {
  // do something with the error, or not
});
```

### ClientOnly

The ClientOnly component lets you render the children element only on the client-side, avoiding rendering it the server-side.

You can provide a fallback component to be used on SSR, and while optional, it's highly recommended to provide one to avoid content layout shift issues.

```tsx
import { ClientOnly } from "remix-utils";

export default function View() {
  return (
    <ClientOnly fallback={<SimplerStaticVersion />}>
      {() => <ComplexComponentNeedingBrowserEnvironment />}
    </ClientOnly>
  );
}
```

This component is handy when you have some complex component that needs a browser environment to work, like a chart or a map. This way, you can avoid rendering it server-side and instead use a simpler static version like an SVG or even a loading UI.

The rendering flow will be:

- SSR: Always render the fallback.
- CSR First Render: Always render the fallback.
- CSR Update: Update to render the actual component.
- CSR Future Renders: Always render the actual component, don't bother to render the fallback.

This component uses the `useHydrated` hook internally.

### CORS

The CORS function let you implement CORS headers on your loaders and actions so you can use them as an API for other client-side applications.

There are two main ways to use the `cors` function.

1. Use it on each loader/action where you want to enable it.
2. Use it globally on entry.server handleDataRequest export.

If you want to use it on every loader/action, you can do it like this:

```ts
export let loader: LoaderFunction = async ({ request }) => {
  let data = await getData(request);
  let response = json<LoaderData>(data);
  return await cors(request, response);
};
```

You could also do the `json` and `cors` call in one line.

```ts
export let loader: LoaderFunction = async ({ request }) => {
  let data = await getData(request);
  return await cors(request, json<LoaderData>(data));
};
```

And because `cors` mutates the response, you can also call it and later return.

```ts
export let loader: LoaderFunction = async ({ request }) => {
  let data = await getData(request);
  let response = json<LoaderData>(data);
  await cors(request, response); // this mutates the Response object
  return response; // so you can return it here
};
```

If you want to setup it globally once, you can do it like this in `entry.server`

```ts
export let handleDataRequest: HandleDataRequestFunction = async (
  response,
  { request }
) => {
  return await cors(request, response);
};
```

#### Options

Additionally, the `cors` function accepts a `options` object as a third optional argument. These are the options.

- `origin`: Configures the **Access-Control-Allow-Origin** CORS header.
  Possible values are:
  - `true`: Enable CORS for any origin (same as "\*")
  - `false`: Don't setup CORS
  - `string`: Set to a specific origin, if set to "\*" it will allow any origin
  - `RegExp`: Set to a RegExp to match against the origin
  - `Array<string | RegExp>`: Set to an array of origins to match against the
    string or RegExp
  - `Function`: Set to a function that will be called with the request origin
    and should return a boolean indicating if the origin is allowed or not.
    The default value is `true`.
- `methods`: Configures the **Access-Control-Allow-Methods** CORS header.
  The default value is `["GET", "HEAD", "PUT", "PATCH", "POST", "DELETE"]`.
- `allowedHeaders`: Configures the **Access-Control-Allow-Headers** CORS header.
- `exposedHeaders`: Configures the **Access-Control-Expose-Headers** CORS header.
- `credentials`: Configures the **Access-Control-Allow-Credentials** CORS header.
- `maxAge`: Configures the **Access-Control-Max-Age** CORS header.

### CSRF

The CSRF related functions let you implement CSRF protection on your application.

This part of Remix Utils needs React and server-side code.

#### Generate the authenticity token

In the server, we need to add to our `root` component the following.

```ts
import type { LoaderFunction } from "remix";
import { createAuthenticityToken, json } from "remix-utils";
import { getSession, commitSession } from "~/services/session.server";

interface LoaderData {
  csrf: string;
}

export let loader: LoaderFunction = async ({ request }) => {
  let session = await getSession(request.headers.get("cookie"));
  let token = createAuthenticityToken(session);
  return json<LoaderData>(
    { csrf: token },
    { headers: { "Set-Cookie": await commitSession(session) } }
  );
};
```

The `createAuthenticityToken` function receives a session object and stores the authenticity token there using the `csrf` key (you can pass the key name as a second argument). Finally, you need to return the token in a `json` response and commit the session.

#### Render the AuthenticityTokenProvider

You need to read the authenticity token and render the `AuthenticityTokenProvider` component wrapping your code in your root.

```tsx
import { Outlet, useLoaderData } from "remix";
import { Document } from "~/components/document";

export default function Root() {
  let { csrf } = useLoaderData<LoaderData>();
  return (
    <AuthenticityTokenProvider token={csrf}>
      <Document>
        <Outlet />
      </Document>
    </AuthenticityTokenProvider>
  );
}
```

With this, your whole app can access the authenticity token generated in the root.

#### Rendering a Form

When you create a form in some route, you can use the `AuthenticityTokenInput` component to add the authenticity token to the form.

```tsx
import { Form } from "remix";
import { AuthenticityTokenInput } from "remix-utils";

export default function SomeRoute() {
  return (
    <Form method="post">
      <AuthenticityTokenInput />
      <input type="text" name="something" />
    </Form>
  );
}
```

Note that the authenticity token is only really needed for a form that mutates the data somehow. If you have a search form making a GET request, you don't need to add the authenticity token there.

This `AuthenticityTokenInput` will get the authenticity token from the `AuthenticityTokenProvider` component and add it to the form as the value of a hidden input with the name `csrf`. You can customize the field name using the `name` prop.

```tsx
<AuthenticityTokenInput name="customName" />
```

You should only customize the name if you also changed it on `createAuthenticityToken`.

##### Alternative: Using `useAuthenticityToken` and `useFetcher`.

If you need to use `useFetcher` (or `useSubmit`) instead of `Form` you can also get the authenticity token with the `useAuthenticityToken` hook.

```tsx
import { useFetcher } from "remix";
import { useAuthenticityToken } from "remix-utils";

export function useMarkAsRead() {
  let fetcher = useFetcher();
  let csrf = useAuthenticityToken();
  return function submit(data) {
    fetcher.submit({ csrf, ...data }, { action: "/action", method: "post" });
  };
}
```

#### Verify in the Action

Finally, you need to verify the authenticity token in the action that received the request.

```ts
import type { ActionFunction } from "remix";
import { verifyAuthenticityToken, redirectBack } from "remix-utils";
import { getSession, commitSession } from "~/services/session.server";

export let action: ActionFunction = async ({ request }) => {
  let session = await getSession(request.headers.get("Cookie"));
  await verifyAuthenticityToken(request, session);
  // do something here
  return redirectBack(request, { fallback: "/fallback" });
};
```

Suppose the authenticity token is missing on the session, the request body, or doesn't match. In that case, the function will throw an Unprocessable Entity response that you can either catch and handle manually or let pass and render your CatchBoundary.

### DynamicLinks

If you need to create `<link />` tags based on the loader data instead of being static, you can use the `DynamicLinks` component together with the `DynamicLinksFunction` type.

In the route you want to define dynamic links add `handle` export with a `dynamicLinks` method, this method should implement the `DynamicLinksFunction` type.

```ts
let dynamicLinks: DynamicLinksFunction<LoaderData> = ({ data }) => {
  if (!data.user) return [];
  return [{ rel: "preload", href: data.user.avatar, as: "image" }];
};
```

Then, in the root route, add the `DynamicLinks` component before the Remix's Links component, usually inside a Document component.

```tsx
import { Links, LiveReload, Meta, Scripts, ScrollRestoration } from "remix";
import { DynamicLinks } from "remix-utils";

type Props = { children: React.ReactNode; title?: string };

export function Document({ children, title }: Props) {
  return (
    <html lang="en">
      <head>
        <meta charSet="utf-8" />
        <meta name="viewport" content="width=device-width,initial-scale=1" />
        {title ? <title>{title}</title> : null}
        <Meta />
        <DynamicLinks />
        <Links />
      </head>
      <body>
        {children}
        <ScrollRestoration />
        <Scripts />
        <LiveReload />
      </body>
    </html>
  );
}
```

Now, any link you defined in the `DynamicLinksFunction` will be added to the HTML as any static link in your `LinksFunction`s.

> Note: You can also put the `DynamicLinks` after the `Links` component, it's up to you what to prioritize, since static links are probably prefetched when you do `<Link prefetch>` you may want to put the `DynamicLinks` first to prioritize them.

### ExternalScripts

If you need to load different external scripts on certain routes, you can use the `ExternalScripts` component together with the `ExternalScriptsFunction` type.

In the route you want to load the script add a `handle` export with a `scripts` method, this method should implement the `ExternalScriptsFunction` type.

```ts
// create the scripts function with the correct type
let scripts: ExternalScriptsFunction = () => {
  return [
    {
      src: "https://code.jquery.com/jquery-3.6.0.min.js",
      integrity: "sha256-/xUj+3OJU5yExlq6GSYGSHk7tPXikynS7ogEvDej/m4=",
      crossOrigin: "anonymous",
    },
  ];
};

// and export it through the handle, you could also create it inline here
// if you don't care about the type
export let handle = { scripts };
```

Then, in the root route, add the `ExternalScripts` component together with the Remix's Scripts component, usually inside a Document component.

```tsx
import { Links, LiveReload, Meta, Scripts, ScrollRestoration } from "remix";
import { ExternalScripts } from "remix-utils";

type Props = { children: React.ReactNode; title?: string };

export function Document({ children, title }: Props) {
  return (
    <html lang="en">
      <head>
        <meta charSet="utf-8" />
        <meta name="viewport" content="width=device-width,initial-scale=1" />
        {title ? <title>{title}</title> : null}
        <Meta />
        <Links />
      </head>
      <body>
        {children}
        <ScrollRestoration />
        <ExternalScripts />
        <Scripts />
        <LiveReload />
      </body>
    </html>
  );
}
```

Now, any script you defined in the ScriptsFunction will be added to the HTML together with a `<link rel="preload">` before it.

> Tip: You could use it together with useShouldHydrate to disable Remix scripts in certain routes but still load scripts for analytics or small features that need JS but don't need the full app JS to be enabled.

### StructuredData

If you need to include structured data (JSON-LD) scripts on certain routes, you can use the `StructuredData` component together with the `HandleStructuredData` type or `StructuredDataFunction` type.

In the route you want to include the structured data, add a `handle` export with a `structuredData` method, this method should implement the `StructuredDataFunction` type.

```ts
import type { WithContext, BlogPosting } from "schema-dts";

// export the handle with the correct type:
export let handle: HandleStructuredData<LoaderData> = {
  structuredData(data: LoaderData) {
    try {
      let { post } = data;

      let postSchema: WithContext<BlogPosting> = {
        "@context": "https://schema.org",
        "@type": "BlogPosting",
        datePublished: post.published,
        mainEntityOfPage: {
          "@type": "WebPage",
          "@id": post.postUrl,
        },
        image: post.featuredImage,
        author: {
          "@type": "Person",
          name: post.authorName,
        },
      };

      return postSchema;
    } catch (e: unknown) {
      console.error(e);
      return [];
    }
  },
};
```

Then, in the root route, add the `StructuredData` component together with the Remix's Scripts component, usually inside a Document component.

```tsx
import { Links, LiveReload, Meta, Scripts, ScrollRestoration } from "remix";
import { StructuredData } from "remix-utils";

type Props = { children: React.ReactNode; title?: string };

export function Document({ children, title }: Props) {
  return (
    <html lang="en">
      <head>
        <meta charSet="utf-8" />
        <meta name="viewport" content="width=device-width,initial-scale=1" />
        {title ? <title>{title}</title> : null}
        <Meta />
        <Links />
        <StructuredData />
      </head>
      <body>
        {children}
        <ScrollRestoration />
        <Scripts />
        <LiveReload />
      </body>
    </html>
  );
}
```

Now, any structured data you defined in the `StructuredDataFunction` will be added to the HTML, in the head. You may choose to include the `<StructuredData />` in either the head or the body, both are valid.

### useActionData

Wrapper of the useActionData from Remix. It lets you pass a reviver function to convert values from the stringified JSON to any JS object.

It also lets you pass a validator function to ensure the final value has the correct shape and get the type of the data correctly.

```ts
type ActionData = { user: { name: string; createdAt: Date } };

let replacer: ReplacerFunction = (key: string, value: unknown) => {
  if (typeof value !== "Date") return value;
  return { __type: "Date", value: value.toISOString() };
};

let reviver: ReviverFunction = (key: string, value: unknown) => {
  if (value.__type === "Date") return new Date(value.value);
  return value;
};

let validator: ValidatorFunction = (data) => {
  return schema.parse(data);
};

export let action: ActionFunction = async ({ request }) => {
  let user = await createUser(request);
  return created<ActionData>({ user }, { replacer });
};

export function Screen() {
  let { user } = useActionData<ActionData>({ reviver, validator });
  return <UserProfile user={user} />;
}
```

### useLoaderData

Wrapper of the useLoaderData from Remix. It lets you pass a reviver function to convert values from the stringified JSON to any JS object.

It also lets you pass a validator function to ensure the final value has the correct shape and get the type of the data correctly.

```ts
type LoaderData = { user: { name: string; createdAt: Date } };

let replacer: ReplacerFunction = (key: string, value: unknown) => {
  if (typeof value !== "Date") return value;
  return { __type: "Date", value: value.toISOString() };
};

let reviver: ReviverFunction = (key: string, value: unknown) => {
  if (value.__type === "Date") return new Date(value.value);
  return value;
};

let validator: ValidatorFunction = (data) => {
  return schema.parse(data);
};

export let loader: LoaderFunction = async ({ request }) => {
  let user = await getUser(request);
  return json<LoaderData>({ user }, { replacer });
};

export function Screen() {
  let { user } = useLoaderData<LoaderData>({ reviver, validator });
  return <UserProfile user={user} />;
}
```

### useDataRefresh

This hook lets you trigger a refresh of the loaders in the current URL.

The way this works is by sending a fetcher.submit to `/dev/null` to trigger all loaders to run.

You need to create `app/routes/dev/null.ts` and define an action that returns null.

```ts
// app/routes/dev/null.ts
export function action() {
  return null;
}
```

This Hook is mostly useful if you want to trigger the refresh manually from an effect, examples of this are:

- Set an interval to trigger the refresh
- Refresh when the browser tab is focused again
- Refresh when the user is online again

```ts
import { useDataRefresh } from "remix-utils";

function useDataRefreshOnInterval() {
  let { refresh } = useDataRefresh();
  useEffect(() => {
    let interval = setInterval(refresh, 5000);
    return () => clearInterval(interval);
  }, [refresh]);
}
```

The return value of `useDataRefresh` is an object with the following keys:

- refresh: a function that trigger the refresh
- type: a string which can be `init`, `refreshRedirect` or `refresh`
- status: a string which can be `loading` or `idle`

### useGlobalPendingState

This hook lets you know if the global transition or if one of any active fetchers is either loading or submitting.

```ts
import { useGlobalPendingState } from "remix-utils";

export function GlobalPendingUI() {
  let globalState = useGlobalPendingState();

  if (globalState === "idle") return null;
  return <Spinner />;
}
```

The return value of `useGlobalPendingState` is either `"idle"` or `"pending"`.

### useHydrated

This hook lets you detect if your component is already hydrated. This means the JS for the element loaded client-side and React is running.

With useHydrated, you can render different things on the server and client while ensuring the hydration will not have a mismatched HTML.

```ts
import { useHydrated } from "remix-utils";

export function Component() {
  let isHydrated = useHydrated();

  if (isHydrated) {
    return <ClientOnlyComponent />;
  }

  return <ServerFallback />;
}
```

When doing SSR, the value of `isHydrated` will always be `false`. The first client-side render `isHydrated` will still be false, and then it will change to `true`.

After the first client-side render, future components rendered calling this hook will receive `true` as the value of `isHydrated`. This way, your server fallback UI will never be rendered on a route transition.

### useLocales

This hooks lets you get the locales returned by the root loader. It follows a simple convention, your root loader return value should be an objet with the key `locales`.

You can combine it with `getClientLocal` to get the locales on the root loader and return that. The return value of `useLocales` is a `Locales` type which is `string | string[] | undefined`.

```ts
// in the root loader
export let loader: LoaderFunction = async ({ request }) => {
  let locales = getClientLocales(request);
  return json({ locales });
};

// in any route (including root!)
export default function Screen() {
  let locales = useLocales();
  let date = new Date();
  let dateTime = date.toISOString;
  let formattedDate = date.toLocaleDateString(locales, options);
  return <time dateTime={dateTime}>{formattedDate}</time>;
}
```

The return type of `useLocales` is ready to be used with the Intl API.

### useRouteData

This hook lets you access the data of any route in the current page. This can include child or parent routes.

To use it, call `useRouteData` in your component and pass the route ID as a string. As an example, if you had the following routes:

```
routes/articles/$slug.tsx
routes/articles/index.tsx
routes/articles.tsx
```

Then you need to pass `useRouteData("routes/articles")` to get the data of `routes/articles.tsx`, `useRouteData("routes/articles/index")` to get the data of `routes/articles/index.tsx` and `routes/articles/$slug` to get the data of `routes/articles/$slug.tsx`.

As you can see, the ID is the route file without the extension.

```ts
let parentData = useRouteData("routes/articles");
let indexData = useRouteData("routes/articles/index");
```

The `useRouteData` hook receives a generic to be used as the type of the route data. Because the route may not be found the return type is `Data | undefined`. This means if you do the following:

```ts
let data = useRouteData<ArticleShowData>("routes/articles");
```

The type of `data` will be `ArticleShowData | undefined`, so you will need to check if it's not undefined before being able to use it.

### useShouldHydrate

If you are building a Remix application where most routes are static, and you want to avoid loading client-side JS, you can use this hook, plus some conventions, to detect if one or more active routes needs JS and only render the Scripts component in that case.

In your document component, you can call this hook to dynamically render the Scripts component if needed.

```tsx
import type { ReactNode } from "react";
import { Links, LiveReload, Meta, Scripts } from "remix";
import { useShouldHydrate } from "remix-utils";

interface DocumentProps {
  children: ReactNode;
  title?: string;
}

export function Document({ children, title }: DocumentProps) {
  let shouldHydrate = useShouldHydrate();
  return (
    <html lang="en">
      <head>
        <meta charSet="utf-8" />
        <link rel="icon" href="/favicon.png" type="image/png" />
        {title ? <title>{title}</title> : null}
        <Meta />
        <Links />
      </head>
      <body>
        {children}
        {shouldHydrate && <Scripts />}
        <LiveReload />
      </body>
    </html>
  );
}
```

Now, you can export a `handle` object with the `hydrate` property as `true` in any route module.

```ts
export let handle = { hydrate: true };
```

This will mark the route as requiring JS hydration.

In some cases, a route may need JS based on the data the loader returned. For example, if you have a component to purchase a product, but only authenticated users can see it, you don't need JS until the user is authenticated. In that case, you can make `hydrate` be a function receiving your loader data.

```ts
export let handle = {
  hydrate(data: LoaderData) {
    return data.user.isAuthenticated;
  },
};
```

The `useShouldHydrate` hook will detect `hydrate` as a function and call it using the route data.

### getClientIPAddress

This function receives a Request or Headers objects and will try to get the IP address of the client (the user) who originated the request.

```ts
export let loader: LoaderFunction = async ({ request }) => {
  // using the request
  let ipAddress = getClientIPAddress(request);
  // or using the headers
  let ipAddress = getClientIPAddress(request.headers);
};
```

If it can't find he ipAddress the return value will be `null`. Remember to check if it was able to find it before using it.

The function uses the following list of headers, in order of preference:

- X-Client-IP
- X-Forwarded-For
- Fly-Client-IP
- CF-Connecting-IP
- Fastly-Client-Ip
- True-Client-Ip
- X-Real-IP
- X-Cluster-Client-IP
- X-Forwarded
- Forwarded-For
- Forwarded

When a header is found that contains a valid IP address, it will return without checking the other headers.

### getClientLocales

This function let you get the locales of the client (the user) who originated the request.

```ts
export let loader: LoaderFunction = async ({ request }) => {
  // using the request
  let locales = getClientLocales(request);
  // or using the headers
  let locales = getClientLocales(request.headers);
};
```

The return value is a Locales type, which is `string | string[] | undefined`.

The returned locales can be directly used on the Intl API when formatting dates, numbers, etc.

```ts
import { getClientLocales } from "remix-utils";
export let loader: LoaderFunction = async ({ request }) => {
  let locales = getClientLocales(request);
  let nowDate = new Date();
  let formatter = new Intl.DateTimeFormat(locales, {
    year: "numeric",
    month: "long",
    day: "numeric",
  });
  return json({ now: formatter.format(nowDate) });
};
```

The value could also be returned by the loader and used on the UI to ensure the user's locales is used on both server and client formatted dates.

### isPrefetch

This function let you identify if a request was created because of a prefetch triggered by using `<Link prefetch="intent">` or `<Link prefetch="render">`.

This will let you implement a short cache only for prefetch requests so you [avoid the double data request](https://sergiodxa.com/articles/fix-double-data-request-when-prefetching-in-remix).

```ts
export let loader: LoaderFunction = async ({ request }) => {
  let data = await getData(request);
  let headers = new Headers();

  if (isPrefetch(request)) {
    headers.set("Cache-Control", "private, max-age=5, smax-age=0");
  }

  return json(data, { headers });
};
```

### Responses

#### json

This function works together with useLoaderData. The function receives any value and returns a response with the value as JSON.

The difference with the built-in `json` function in Remix is that this one lets you pass a replacer function which will be passed to JSON.stringify to let you control how your values are transformed to string.

This is useful to support sending BigInt, Date, Error, or any custom class value which is not directly supported on the JSON format.

```tsx
// ensure you import both json and useLoaderData from Remix Utils
import { json, useLoaderData } from "remix-utils";
import type { ReplacerFunction, ReviverFunction } from "remix-utils";
import type { LoaderFunction } from "remix";

import { getUser } from "../services/users";
import type { User } from "../types";

type LoaderData = { user: User };

let replacer: ReplacerFunction = (key: string, value: unknown) => {
  if (typeof value !== "Date") return value;
  return { __type: "Date", value: value.toISOString() };
};

let reviver: ReviverFunction = (key: string, value: unknown) => {
  if (value.__type === "Date") return new Date(value.value);
  return value;
};

export let loader: LoaderFunction = async ({ request }) => {
  let user = await getUser(request);
  return json<LoaderData>({ user }, { replacer });
};

export function Screen() {
  let { user } = useLoaderData<LoaderData>({ reviver });
  return <UserProfile user={user} />;
}
```

> Note: All helpers below use this json function, ensure you always import useLoaderData from Remix Utils

#### Redirect Back

This function is a wrapper of the `redirect` helper from Remix. Unlike Remix's version, this one receives the whole request object as the first value and an object with the response init and a fallback URL.

The response created with this function will have the `Location` header pointing to the `Referer` header from the request, or if not available, the fallback URL provided in the second argument.

```ts
import { redirectBack } from "remix-utils";
import type { ActionFunction } from "remix";

export let action: ActionFunction = async ({ request }) => {
  await redirectBack(request, { fallback: "/" });
};
```

This helper is most useful when used in a generic action to send the user to the same URL it was before.

#### Created

Helper function to create a Created (201) response with a JSON body.

```ts
import { created } from "remix-utils";
import type { ActionFunction } from "remix";

export let action: ActionFunction = async ({ request }) => {
  let result = await doSomething(request);
  return created(result);
};
```

#### Bad Request

Helper function to create a Bad Request (400) response with a JSON body.

```ts
import { badRequest } from "remix-utils";
import type { ActionFunction } from "remix";

export let action: ActionFunction = async () => {
  throw badRequest({ message: "You forgot something in the form." });
};
```

#### Unauthorized

Helper function to create an Unauthorized (401) response with a JSON body.

```ts
import { unauthorized } from "remix-utils";
import type { LoaderFunction } from "remix";

export let loader: LoaderFunction = async () => {
  // usually what you really want is to throw a redirect to the login page
  throw unauthorized({ message: "You need to login." });
};
```

#### Forbidden

Helper function to create a Forbidden (403) response with a JSON body.

```ts
import { forbidden } from "remix-utils";
import type { LoaderFunction } from "remix";

export let loader: LoaderFunction = async () => {
  throw forbidden({ message: "You don't have access for this." });
};
```

#### Not Found

Helper function to create a Not Found (404) response with a JSON body.

```ts
import { notFound } from "remix-utils";
import type { LoaderFunction } from "remix";

export let loader: LoaderFunction = async () => {
  throw notFound({ message: "This doesn't exists." });
};
```

#### Unprocessable Entity

Helper function to create an Unprocessable Entity (422) response with a JSON body.

```ts
import { unprocessableEntity } from "remix-utils";
import type { LoaderFunction } from "remix";

export let loader: LoaderFunction = async () => {
  throw unprocessableEntity({ message: "This doesn't exists." });
};
```

This is used by the CSRF validation. You probably don't want to use it directly.

#### Server Error

Helper function to create a Server Error (500) response with a JSON body.

```ts
import { serverError } from "remix-utils";
import type { LoaderFunction } from "remix";

export let loader: LoaderFunction = async () => {
  throw serverError({ message: "Something unexpected happened." });
};
```

#### Not Modified

Helper function to create a Not Modified (304) response without a body and any header.

```ts
export let loader: LoaderFunction = async ({ request }) => {
  return notModified();
};
```

#### JavaScript

Helper function to create a JavaScript file response with any header.

This is useful to create JS files based on data inside a Resource Route.

```ts
export let loader: LoaderFunction = async ({ request }) => {
  return javascript("console.log('Hello World')");
};
```

#### Stylesheet

Helper function to create a CSS file response with any header.

This is useful to create CSS files based on data inside a Resource Route.

```ts
export let loader: LoaderFunction = async ({ request }) => {
  return stylesheet("body { color: red; }");
};
```

#### PDF

Helper function to create a PDF file response with any header.

This is useful to create PDF files based on data inside a Resource Route.

```ts
export let loader: LoaderFunction = async ({ request }) => {
  return pdf(await generatePDF(request.formData()));
};
```

#### HTML

Helper function to create a HTML file response with any header.

This is useful to create HTML files based on data inside a Resource Route.

```ts
export let loader: LoaderFunction = async ({ request }) => {
  return html("<h1>Hello World</h1>");
};
```

## Author

- [Sergio Xalambrí](https://sergiodxa.com)

## License

- MIT License
