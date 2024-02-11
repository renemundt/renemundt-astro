---
title: 'UseSwr hook with react-i18next '
pubDate: 2024-02-11
description: 'Using useSwr hook together with react-i18next'
author: 'René Mundt'
tags: ['reactjs', 'nextjs', 'react-i18next', 'usewr']
---

As part of working on a feature at [DFDS A/S](https://www.dfds.com) I needed to combine [useSwr](https://swr.vercel.app/docs/typescript.en-US#useswr) with [react-i18next](https://react.i18next.com) but was not able to find any examples online. This blogpost shows how a hook like that can be implemented.

## Implementation

```js
import useSWR from 'swr'
import { initReactI18next } from 'react-i18next'
import i18n from 'i18next'
import { fetcher } from '@/utils/common'
import { getUrlLocale } from '@/utils/getLocale'

export const useI18Next = (): I18NextResult => {
  const locale = getUrlLocale()

  let _i18Instance = null

  const i18Instance = i18n.createInstance()
  i18Instance.use(initReactI18next).init({
    debug: false,
    lng: locale,
    fallbackLng: 'en',
    interpolation: {
      escapeValue: false,
    },
    resources: {},
  })

  const { data, isLoading } = useSWR(locale ? `/api/content?locale=${locale}` : null, fetcher, {
    onErrorRetry: (err, _key, _config, revalidate, { retryCount }) => {
      if (err.status === 404) return
      if (retryCount >= 2) return
      setTimeout(() => revalidate({ retryCount }), 1000)
    },
  })

  if (data) {
    i18Instance.addResourceBundle(locale.substring(0, 2), 'translation', data, true, true)
    _i18Instance = i18Instance
  }

  return {
    isLoading,
    i18Instance: _i18Instance,
  }
}

export interface I18NextResult {
  isLoading: boolean
  i18Instance: any
}
```

The function

```js
const locale = getUrlLocale()
```

just returns the locale from the url as the naming reveals. DFDS has 26 locales and the getUrlLocale in this case resolves the locale taken from the url like https://www.dfds.com/da-dk/passagerfaerger which ends up being **da-dk** for the danish language/locale.

```js
import { fetcher } from '@/utils/common'
```

is needed by useSwr and is just a wrapper around [fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) as mentioned here https://swr.vercel.app/docs/data-fetching.
The fetcher used here looks like this:

```js
export const fetcher = async (resource: any, init: any) => {
  const res = await fetch(resource, init)
  if (!res.ok) {
    const { status } = await res.json()
    throw new Error(`An error occurred: ${status}`)
  }
  return res.json()
}
```

If the result is not ok (!res.ok) i.e. if the request fails and returns a response with a http status code different than 200 an error will be raised. _Note: this is how fetch handles failed requests. Other http clients might handle this in different ways._

```js
const { data, isLoading } = useSWR(locale ? `/api/content?locale=${locale}` : null, fetcher, {
  onErrorRetry: (err, _key, _config, revalidate, { retryCount }) => {
    if (err.status === 404) return
    if (retryCount >= 2) return
    setTimeout(() => revalidate({ retryCount }), 1000)
  },
})
```

will make use of the useSwr hook which will call the endpoint _/api/content?locale=da-dk_ if the locale is not null, otherwise it will not make any call because of the ternary expresssion _: null_.

The block also has a retry block which will not fire if the endpoint doesn't exist (404) or if more than 2 preceding requests has happened. The setTimeout controls the intervals between the retries.

While the request is fetching data the isLoading will be true and false when data is fetched.

```js
if (data) {
  i18Instance.addResourceBundle(locale.substring(0, 2), 'translation', data, true, true)
  _i18Instance = i18Instance
}

return {
  isLoading,
  i18Instance: _i18Instance,
}
```

When data is fetched it is added as a resourceBundle to the i18Instance and afterwards set as a property on the the hook which is returned and ready for usage.

## Using the hook

Using the hook is staight and forard like this. If the hook is working on loading the content or the i18Instance isn't set yet it will show a loader. Otherwise if will fire up the underlaying components (which in this case is a [nextjs](https://nextjs.org/) application).
At DFDS we use the hook in multiple websites that needs content from our centralized content system.

```js
import type { AppProps } from 'next/app'
import { I18nextProvider } from 'react-i18next'
import { useI18Next } from '@/hooks/useI18Next'
import { DfdsLoader } from '@dfds-ui/react-components'

const App = ({ Component, pageProps }: AppProps) => {
  const { isLoading, i18Instance } = useI18Next()

  if (isLoading && !i18Instance) return <DfdsLoader />
  else
    return (
      <I18nextProvider i18n={i18Instance}>
        <Component {...pageProps} />
      </I18nextProvider>
    )
}

export default App

```
