---
title: 'UseSwr hook with react-i18next '
pubDate: 2024-01-19
description: 'Using useSwr hook together with react-i18next'
author: 'René Mundt'
tags: ['reactjs', 'nextjs', 'reacti-18next', 'usewr']
---

As part of working on a feature at [DFDS A/S](https://www.dfds.com) I needed to combine [useSwr](https://swr.vercel.app/docs/typescript.en-US#useswr) with [react-i18next](https://react.i18next.com) but was not able to find any examples searching the web. This blogpost is about implementing a react hook the does exactly that.

## Implementation

1. **Hook**: This is the hook implementation

```
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

  const { data, isLoading } = useSWR(locale ? `/api/customer-account/content?locale=${locale}` : null, fetcher, {
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

2. **Using the hook**: This is the usage of the hook
