# Next Translation

[![npm version](https://badge.fury.io/js/next-translation.svg)](https://badge.fury.io/js/next-translation)

Next Translation is an internationalization library for React.js with an enabled server component (especially for Next.js App Router).

## Why one more library?
Server components are an experimental feature of React, and currently translation libraries are poorly optimized for them. If supported, disable Next.js static optimization.

This library is an attempt to create a highly optimized solution exclusively using the current capabilities of React and Next.js.

## Installation

**Using npm:**
```bash
npm i next-translation
```

**Using yarn:**
```bash
yarn add next-translation
```

## Configuration
Create config file

```js
class LoaderProvider {
    async load(lang): {
        const resp = await fetch(`https://api.translation-example.com/terms/${lang}`, {
            method: 'POST',
        });
        const data = await resp.json();
        return data;
    }
}

/** @type {import('next-translation/types').Config} */
module.exports = {
  loaderProvider: new LoaderProvider(),
};
```

## Config options

### loaderProvider

`loaderProvider` - class with load method, which will load data and return it in the next format:

`type ReturnType = { data: { [key: string]: Translates | string }, meta: { [key: string]: string } }`

method load will receive 2 arguments:

`key` - current language

`meta` - meta data from previous request. recieved only with enabled advanced mode

### unstable_advancedLoader

`unstable_advancedLoader` - option to load data custom caching instead of next.js fetch [Read more](#advanced-loader)

## Base translates

### Server components:

Use `getTranslation` in async server components
```tsx
"use server";

import getTranslation from 'next-tranlation/getTranslation';

const ServerComponent: React.FC<{ lang: string }> = async ({ lang }) => {
    const { t } = await getTranslation(lang);

    return (
        <div>
            {/* Welcome to Next Translation */}
            {t('header.nav.home')}
        </div>
    )
}
```

### Client components:

To begin, initialize the Transmitter in the server component, which should be located under the client component.
```tsx
"use server";

import WithNextTranslation from 'next-tranlation/WithNextTranslation';
import ClientComponent from './ClientComponent';

const ServerComponent: React.FC<{ lang: string }> = ({ lang }) => (
    <WithNextTranslation lang={lang} terms={['header.nav']}>
        <ClientComponent />
    </WithNextTranslation>
)
```

Then you can use `useTranslation` in client components
```tsx
"use client";

import useTranslation from 'next-tranlation/useTranslation';

const ClientComponent: React.FC = () => {
    const { t } = useTranslation();

    return (
        <div>
            {/* Welcome to Next Translation */}
            {t('header.nav.home')}
        </div>
    )
}
```

### Options

Both of them agree to use `opts` as the second argument of `t`. Now you can pass a query there to inject variable strings inside translations.

```tsx
const Component: React.FC = () => {
    return (
        <div>
            {/* Price starts from ${{price}} */}
            {t('pricing.tariffs.solo', { query: { price: 16.99 } })}
        </div>
    )
}
```

Also you can inject query to client terms on the server side. For example, when they depend on the server environment or when you get values ​​from a database on the server.

Just add an array value instead of a string in terms arr, where the second element will be the query object.

```tsx
"use server";

import WithNextTranslation from 'next-tranlation/WithNextTranslation';
import ClientComponent from './ClientComponent';

const ServerComponent: React.FC<{ lang: string }> = ({ lang }) => (
    <WithNextTranslation lang={lang} terms={[['home.welcome', { stage: process.env.GITHUB_REF === 'main' ? 'production' : 'test' }]]}>
        <ClientComponent />
    </WithNextTranslation>
)
```

## Difficult tranlates

To handle difficult translations, you can use the `ServerTranslation` or `ClientTranslation` with components prop. These components would be injected into the translation, whether on the server or the client side.

### Server components:

Use `ServerTranslation` in async server components
```tsx
"use server";

import ServerTranslation from 'next-tranlation/ServerTranslation';

const ServerComponent: React.FC<{ lang: string }> = async ({ lang }) => (
    <ServerTranslation
        lang={lang}
        term='intro.description' // We have {{number}} tariffs. Read more about pricings <link>special section</link>
        components={{
            link: <a href='#' />
        }}
        query={{ number: 5 }}
    />
)
```

### Client components:

Use `ClientTranslation` in client components
```tsx
"use client";

import ClientTranslation from 'next-tranlation/ClientTranslation';

const ClientComponent: React.FC = async () => (
    <ClientTranslation
        term='intro.description' // We have {{number}} tariffs. Read more about pricings <link>special section</link>
        components={{
            link: <a href='#' />
        }}
        query={{ number: 5 }}
    />
)
```

## Advanced loader

I highly recommend using next.js fetch with revalidation configured for better performance with ISR mode. However, if it's not possible (_e.g., when loading data with a POST request or if the response size is larger than 2MB_), you can use the `unstable_advancedLoader` option in next-translation to optimize requests.

It is important to use one of the caching systems (_next.js fetch logic only or the next-translation advanced loader_) because on the server we are always looking for the data source every time we use the next-translation tools.

Don't combine next.js fetch caching with the next-translation advanced loader, it will repeat the logic and may not work correctly.

### Options

### Revalidate
`revalidate`. Option works similarly to the next.js option. Setting it to `false` (_default_) will disable revalidation, meaning the data will be cached indefinitely. Setting it to `0` will request the data each time, while setting it to a `number` will determine the time in seconds for revalidation to occur.

### retryAttempts
`retryAttempts` - number of retries when loading data (_3 by default_)

### checkIsActual
`checkIsActual` - сheck that the data is up to date. Use it when you can perform additional steps to ensure that cached data is up to date. For example, make a HEAD request with meta information, or check the meta on a different route. Option type:

`(key: string, meta?: { [key: string]: string }) => Promise<boolean>`

where:

`key` - target language

`meta` - meta information returned from the [loaderProvider](#config-options) load method

### cacheHandler
`cacheHandler` - custom cache handler (_Heap Space by default_). Should be a class with asynchronous methods: get, set, has.

```ts
class CacheHandler {
    async get<T>(key: string): Promise<T> {

    }

    async set(key: string, data: unknown): Promise<void> {

    }

    async has(key: string): Promise<boolean> {

    }
}
```

## App organization

### Pages structure

```
app
    [lang]
        page-name
            page.tsx
        page.tsx
        layout.tsx
    (root)
        page-name
            page.tsx
        page.tsx
        layout.tsx
```

**Why so?**

You can only create one root layout. In the root (`/app`) we don't know the language, so we can't add a `lang` attribute there on the server side.

Therefore, the only way to do it is to create a root layout for localized pages in the `[lang]` folder.

```tsx
// /app/[lang]/layout.tsx
type RootLayoutProps = { children: React.ReactNode; params: { lang: string } }

export default function RootLayout({ children, params }: RootLayoutProps) {
  return (
    <html lang={params.lang}>
      <body>{children}</body>
    </html>
  )
}
```

## License

[MIT](https://github.com/vordgi/next-translation/blob/main/LICENSE)
