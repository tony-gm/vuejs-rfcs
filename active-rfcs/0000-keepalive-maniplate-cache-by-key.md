- Start Date: 2022-03-09
- Target Major Version: 2.x/3.x
- Reference Issues: https://github.com/vuejs/vue-next/issues/2084, https://github.com/vuejs/vue-next/issues/2077, https://github.com/vuejs/vue-next/issues/1518, https://github.com/vuejs/vue/issues/6259, https://github.com/vuejs/vue/issues/8028, https://github.com/vuejs/vue/issues/10487
- Implementation PR: https://github.com/vuejs/vue-next/pull/4339

# Summary

- Provide a composoable function `createKeepAliveCache`, return a `KeepAliveCache` object.

```ts
export interface KeepAliveCache {
  include: Ref<MatchPattern | undefined>
  exclude: Ref<MatchPattern | undefined>
  max: Ref<number | string | undefined>
  cache: Map<string, VNode>
  remove: (key: string) => void
  clear: () => void
}
```

- KeepAlive component, provide a new prop named `cache`, then user can v-bind to `createKeepAliveCache()` returned `KeepAliveCache` object.

- User can change `include`,`exclude`,`max` to maniplate cache as current API (since it's a `Ref`), also `remove(key)` can remove a cache by key, `clear()` to clear all cache.

# Motivation

- Current keep-alive component API does not have any way to remove cache by a key. only provide `include`, `exclude` props to manipulate cache by component name.

- For **Multi-Tab Page** project, ever opened page will add new tab on a tabbar, and all page on tabbar should be cached.
  The cache can only and should be removed is when the user close one tab.
  Just Similar with VSCode, all opened file should be cached, and if I closed the page, the page cache must be removed. **All pages are rendered as same component**

- Since maniuplate cache by key is an **ESSENTIAL** function of **Multi-Tab Page** project, but in current keep-alive component there is no way can achieve that (even any alterative temporary solution).

For these, we introduce a new prop for keep-alive component named `cache`, and provide a `createKeepAliveCache()` function to create the cache which will bind to `:cache` property. BTW if user not bind this `cache` property, it will run as current API.

# Detailed design

### 1. KeepAliveCache

`KeepAliveCache` is a **interface** of object structure which will bind to `:cache` prop.

```ts
export interface KeepAliveCache {
  include: Ref<MatchPattern | undefined>
  exclude: Ref<MatchPattern | undefined>
  max: Ref<number | string | undefined>
  cache: Map<string, VNode>
  remove: (key: string) => void
  clear: () => void
}
```

- `incude`,`exclude` and `max` is totaly same as current props, so it will not affect current API.
- `cache` is the cached data map, user can view all the cached keys and vnode.
- `remove` is a function, user can remove cache by a _key_
- `clear` is another function to clear all cached vnode.

### 2. createKeepAliveCache

`createKeepAliveCache` is a composoable function (like `createVue`, `createRouter`). It's return a `KeepAliveCache` object which will bind the `:cache` property.

BTW: `remove(key)` and `clear()` function can proxy to current `pruneEntry` method to make source code minimal changed.

```ts
export const createKeepAliveCache = (
  props?: Omit<KeepAliveProps, 'cache'>
): KeepAliveCache => {
  const include = props?.include ? toRef(props, 'include') : ref<MatchPattern>()
  const exclude = props?.exclude ? toRef(props, 'exclude') : ref<MatchPattern>()
  const max = props?.max ? toRef(props, 'max') : ref<number | string>()
  const cache = new Map()

  const remove: CacheRemoveFunc = Object.assign(
    (key: CacheKey) => {
      if (remove.__prune_proxy && cache.has(key)) remove.__prune_proxy(key)
    },
    { __prune_proxy: null }
  )
  const clear = () => {
    if (remove.__prune_proxy) {
      for (const key of cache.keys()) {
        remove.__prune_proxy(key)
      }
    }
  }
  return { include, exclude, max, cache, remove, clear }
}
```

### :cache

KeepAlive component provide a new prop named `cache`, then user can v-bind to `createKeepAliveCache()` returned `KeepAliveCache` object.

```html
<template>
  <router-view v-slot="{ Component, route }">
    <keep-alive ref="keepAlive" :cache="pageCache">
      <component :is="Component" :key="route.fullPath" />
    </keep-alive>
  </router-view>
</template>
<script setup>
  const pageCache = createKeepAliveCache()
</script>
```

- User can change `include`,`exclude`,`max` to maniplate cache as current API, also `remove(key)` can remove a cache by key, `clear()` to clear all cache.

  ```ts
  const pageCache = createKeepAliveCache()
  pageCache.include.value = ['Component1']
  function onCloseButtonClick() {
    pageCache.remove(currentPath)
  }
  function onClearButtonClick() {
    pageCache.clear()
  }
  ```

Here is a full example

```vue
<template>
  <button @click="onCloseTab">Close Current Tab</button>
  <button @click="onCloseAllTabs">Close All Tabs</button>
  <router-view v-slot="{ Component, route }">
    <keep-alive ref="keepAlive" :cache="pageCache">
      <component :is="Component" :key="route.fullPath" />
    </keep-alive>
  </router-view>
</template>

<script lang="ts">
import { defineComponent, createKeepAliveCache, ref } from 'vue'
export default defineComponent({
  name: 'App',
  setup() {
    const currentPath = ref('')
    const pageCache = createKeepAliveCache()
    pageCache.include.value = ['Component1']

    function onCloseTab() {
      pageCache.remove(currentPath.value)
    }

    function onCloseAllTabs() {
      pageCache.clear()
    }

    return {
      pageCache,
      onCloseTab,
      onCloseAllTabs
    }
  }
})
</script>
```

Here is a prototype implement base on this RFC ( [KeepAlive.ts](https://github.com/tony-gm/vue3-core/blob/main/packages/runtime-core/src/components/KeepAlive.ts) ). It's a little change base on current source code. But It's working fine and not affect current
API. Later on I will do more test and create fully demo project.

## Lifecycle hooks

- NA

# Drawbacks

- Not sure there is another better solution for manipulate cache by key, easy for user, not affected current API, and easy to implement.

# Adoption strategy

These are enchancement based on the current API
