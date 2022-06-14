# Find the Entry

This article belongs to the series [Read Vue Source Code](https://github.com/numbbbbb/read-vue-source-code).

___

Vue와 같이 거대한 오픈 소스 프로젝트를 살펴볼 때 어디서부터 살펴볼지 늘 고민이 되는 것이 사실이다. Vue는 npm package이기 때문에 package.json 파일을 먼저 살펴보는 것이 가장 올바른 방법이라고 할 수 있겠다.

```json
// package.json

{
  "private": true,
  "version": "3.2.37",
  "packageManager": "pnpm@7.1.0",
  "scripts": {
    "dev": "node scripts/dev.js",
    "build": "node scripts/build.js",
    "size": "run-s size-global size-baseline",
...
```

- private

private을 true로 해두면, publish 명령을 거부하게 된다. 이 항목은 개인적으로만 사용하는 저장소를 무심코 publish 해버리는 것을 방지한다.

packageManager 항목은 별도의 설명이 필요하지 않을 거 같아 넘어가도록 하겠다. 

- scripts

scripts 항목은 패키지의 생명주기 중 다양한 타이밍에서 실행되는 script 명령들을 포함하고 있다.

일반적으로 개발 모드를 실행시킬 때 dev 명령어를 사용하기 때문에, dev 명령어의 값을 살펴보면 어디서부터 분석을 시작해야 할지 알 수 있을 것이다.

```json
"dev": "node scripts/dev.js",
```

```js
// scripts/dev.js
...

build({
  entryPoints: [resolve(__dirname, `../packages/${target}/src/index.ts`)],
...
```

개발 모드 build 설정을 위한 파일임을 알 수 있다. `entryPoints`를 살펴보니 resolve 메서드를 사용하고 있다.

- resolve

resolve 메서드는 연속된 경로들이나 경로 조각들을 하나의 절대 경로로 만들어준다. 절대 경로로 만들어지는 과정은 오른쪽에서 왼쪽으로 진행된다. 만약 절대 경로가 만들어지지 않는다면 현재 디렉토리 경로를 반환한다.

```js
entryPoints: [resolve(__dirname, `../packages/${target}/src/index.ts`)],
```

여기서 `__dirname`은 현재 실행하는 파일의 절대경로이다. 그러니까 `entryPoints`는

**vue 프로젝트 폴더/packages/${target}/src/index.ts**가 된다. 그러면 이제 target 변수의 값을 찾아보자.

```js
const args = require('minimist')(process.argv.slice(2))
const target = args._[0] || 'vue'
```

target을 알기 위해서는 위 코드들을 모두 이해해야 한다.

- minimist

minimist는 npm package로 인수를 parse하는 역할을 한다. 예시를 통해 이게 어떤 뜻인지 확인해보자.

```js
var argv = require('minimist')(process.argv.slice(2));
console.log(argv);
```

```bash
>> node index.js -a beep -b boop        
>> { _: [], a: 'beep', b: 'boop' }
```

먼저 process.argv 속성은 Node.js 프로세스가 시작될 때 전달된 명령행 인수를 포함하는 배열을 반환한다. 여기서 slice() 메서드를 사용한 이유는 명령의 첫 번째 인수는 node, 두 번째 인수는 파일 이름으로 고정되어 있기 때문이다. 

console.log를 통해 출력된 값을 보면 비어있는 `-`배열을 확인할 수 있는데, 이는 어떤 옵션과도 관련이 없는 명령행 인수들을 포함하는 배열이다. 그러니까 위의 명령어에서는 a 의 옵션에 대한 값으로 'beep' 이라는 값을, b 옵션의 값으로 'boop' 이라는 값을 전달하고 있어 모든 옵션에 대응되는 명령행 인수가 있기 때문에 빈 배열을 반환하는 것이다.

___

```js
// scripts/dev.js
const args = require('minimist')(process.argv.slice(2))
const target = args._[0] || 'vue'
```

이제 필요한 지식들은 모두 알았으니 다시 위의 코드를 분석해보겠다.

agrs에는 위에서 알아봤듯이 명령행 인수가 포함된 배열을 반환한다. 그러면 package.json에서 dev 명령어의 값을 다시 한 번 확인해보자.

```json
// package.json
...
  "scripts": {
    "dev": "node scripts/dev.js",
```

args에는 slice메서드를 사용했기 때문에 node와 실행시킬 파일 경로를 제외한 명령행 인수들만이 담기게 된다. 그렇기 때문에 `dev` 명령어를 실행하면 args와 옵션과 관련되지 않은 명령행 인수들이 담기는 `_` 도 빈 배열을 반환하게 된다. 따라서 `dev` 명령어를 실행했을 때 target의 값은 무조건 'vue'가 되는 것이다.

먼 길을 돌아왔지만 우리가 맨 처음에 하고자 했던 건 vue 명령어가 실행됐을 때 `entryPoints`의 값을 확인하는 것이었다. target이 'vue'이기 때문에 `entryPoints`는 **vue 프로젝트 폴더/packages/vue/src/index.ts**가 되는 것이다.

```typescript
// packages/vue/src/index.ts
...
if (__DEV__) {
  initDev()
}
...
```

index.ts 파일을 확인해보면, `__DEV__`변수의 값이 true 일 때 initDev() 메서드를 실행시키고 있다. `__DEV__`변수는 dev.js 파일에서 정의되고 있다.

```typescript
// scripts/dev.js
...
build({
  entryPoints: [resolve(__dirname, `../packages/${target}/src/index.ts`)],
  outfile,
  bundle: true,
  external,
  sourcemap: true,
  format: outputFormat,
  globalName: pkg.buildOptions?.name,
  platform: format === 'cjs' ? 'node' : 'browser',
  plugins:
    format === 'cjs' || pkg.buildOptions?.enableNonBrowserBranches
      ? [nodePolyfills.default()]
      : undefined,
  define: {
    __COMMIT__: `"dev"`,
    __VERSION__: `"${pkg.version}"`,
    __DEV__: `true`,
    __TEST__: `false`,
    __BROWSER__: String(
      format !== 'cjs' && !pkg.buildOptions?.enableNonBrowserBranches
    ),
    __GLOBAL__: String(format === 'global'),
    __ESM_BUNDLER__: String(format.includes('esm-bundler')),
    __ESM_BROWSER__: String(format.includes('esm-browser')),
    __NODE_JS__: String(format === 'cjs'),
    __SSR__: String(format === 'cjs' || format.includes('esm-bundler')),
    __COMPAT__: String(target === 'vue-compat'),
    __FEATURE_SUSPENSE__: `true`,
    __FEATURE_OPTIONS_API__: `true`,
    __FEATURE_PROD_DEVTOOLS__: `false`
  },
  watch: {
    onRebuild(error) {
      if (!error) console.log(`rebuilt: ${relativeOutfile}`)
    }
  }
}).then(() => {
  console.log(`watching: ${relativeOutfile}`)
})
```

esbuild 는 웹팩과 같은 번들러의 한 종류이다. build 메서드 안에다가 여러 가지 옵션들을 정의한 객체를 넘겨주는 형식이다.

여기서 `__DEV__`이 정의된 define 옵션은 전역 변수를 바꿀 수 있는 방법을 제공한다. 여기서 `__DEV__`가 true로 정의되었기 때문에 `initDev()`메서드가 실행된다.

```typescript
// packages/vue/src/dev.ts

export function initDev() {
  if (__BROWSER__) {
    if (!__ESM_BUNDLER__) {
      console.info(
        `You are running a development build of Vue.\n` +
          `Make sure to use the production build (*.prod.js) when deploying for production.`
      )
    }

    initCustomFormatter()
  }
}
```

dev.ts 파일에 정의된 initDev 메서드이다. 

```typescript
// scripts/dev.js
...
    __BROWSER__: String(
      format !== 'cjs' && !pkg.buildOptions?.enableNonBrowserBranches
    ),
    __GLOBAL__: String(format === 'global'),
    __ESM_BUNDLER__: String(format.includes('esm-bundler')),
    __ESM_BROWSER__: String(format.includes('esm-browser')),
...
```

`__BROWSER__`와 `__ESM_BUNDLER__`, 두 가지 변수의 값을 확인해보겠다.

 ```typescript
 // scripts/dev.js
 ...
 const format = args.f || 'global'
 const pkg = require(resolve(__dirname, `../packages/${target}/package.json`))
 ```

dev 명령어에는 -f 에 대한 값의 정의가 없다. 아까와 마찬가지로 dev 명령을 실행하면 무조건 'global' 의 값을 갖게 된다. 그러면 format은 'global'로 'cjs'와 같지 않으므로 true를 반환한다. 이제 두 번째 조건을 살펴보자.

앞에서 dev 명령어를 실행하면 target의 값은 무조건 vue 가 된다고 했다. 따라서 resolve로 만들어지는 절대 경로는 **vue 프로젝트 폴더/packages/vue/package.json** 이며, 그 안에 정의된 객체가 변수 pkg에 담긴다.

```json
// packages/vue/package.json
...
  "buildOptions": {
    "name": "Vue",
    "formats": [
      "esm-bundler",
      "esm-bundler-runtime",
      "cjs",
      "global",
      "global-runtime",
      "esm-browser",
      "esm-browser-runtime"
    ]
  },
...
```

객체 안에 정의된 buildOptions를 살펴보면 enableBrowserBranches 라는 속성이 없다. 따라서 `pkg.buildOptions?.enableNonBrowserBranches`의 값은 undefined가 되며, 앞의 `!`가 있기 때문에 결과값은 true 가 된다. 결과적으로 `__BROWSER__`의 값은 'true' 가 된다.

앞에서 format은 dev 명령어를 실행하면 'global'의 값을 갖게 된다고 했으므로, `__ESM_BUNDLER__`의 값은 'false'가 된다.

```typescript
// packages/vue/src/dev.ts

export function initDev() {
  if (__BROWSER__) {
    if (!__ESM_BUNDLER__) {
      console.info(
        `You are running a development build of Vue.\n` +
          `Make sure to use the production build (*.prod.js) when deploying for production.`
      )
    }

    initCustomFormatter()
  }
}
```

다시 initDev 메서드로 돌아가보자. 이제 두 변수의 값을 알기 때문에 조건식이 어떻게 실행될지 알 수 있다.

- `__BROWSER__` = "true"
- `__ESM_BUNDLER__` = "false"

따라서 두 조건식 모두 true로 console.info 가 실행된다. development build를 하고 있으니, 배포를 위해서는 production build를 사용하라는 메시지이다. 

두 조건식이 실행되고나면 initCustomFormatter() 메서드가 실행된다. 





Great! 

This is the config of dev building. It's entry is `web/entry-runtime-with-compiler.js`. But wait, where is the `web/` directory?

Check the entry value again, there is a `resolve`. Search `resolve`:

```javascript
const aliases = require('./alias')
const resolve = p => {
  const base = p.split('/')[0]
  if (aliases[base]) {
    return path.resolve(aliases[base], p.slice(base.length + 1))
  } else {
    return path.resolve(__dirname, '../', p)
  }
}
```


Go through `resolve('web/entry-runtime-with-compiler.js')` with the parameters we have:

- p is `web/entry-runtime-with-compiler.js`
- base is `web`
- convert the directory to `alias['web']`

Let's jump to `alias` to find `alias['web']`.

```javascript
module.exports = {
  vue: path.resolve(__dirname, '../src/platforms/web/entry-runtime-with-compiler'),
  compiler: path.resolve(__dirname, '../src/compiler'),
  core: path.resolve(__dirname, '../src/core'),
  shared: path.resolve(__dirname, '../src/shared'),
  web: path.resolve(__dirname, '../src/platforms/web'),
  weex: path.resolve(__dirname, '../src/platforms/weex'),
  server: path.resolve(__dirname, '../src/server'),
  entries: path.resolve(__dirname, '../src/entries'),
  sfc: path.resolve(__dirname, '../src/sfc')
}
```

Okay, it's `src/platforms/web`. Concat it with the input file name, we get `src/platforms/web/entry-runtime-with-compiler.js`.

```javascript
/* @flow */

import config from 'core/config'
import { warn, cached } from 'core/util/index'
import { mark, measure } from 'core/util/perf'

import Vue from './runtime/index'
import { query } from './util/index'
import { shouldDecodeNewlines } from './util/compat'
import { compileToFunctions } from './compiler/index'

const idToTemplate = cached(id => {
  const el = query(id)
  return el && el.innerHTML
})

const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)

  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
      `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
    )
    return this
  }

  const options = this.$options
  // resolve template/el and convert to render function
  if (!options.render) {
    let template = options.template
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          template = idToTemplate(template)
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && !template) {
            warn(
              `Template element not found or is empty: ${options.template}`,
              this
            )
          }
        }
      } else if (template.nodeType) {
        template = template.innerHTML
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
        return this
      }
  ...
```

Cool! You have found the entry!

> Have you noticed the `/* flow */` comment at the top? After Google, we know it's a type checker. It reminds me of `typings`, I search again and find out that `flow` can use `typings` definitions to validate your code.

> Maybe next time I can use `flow` and `typings` in my own project. You see, even if we haven't really started reading the code, we have learned some useful and practical things.

Let's go through this file step-by-step:

- import config
- import some util functions
- import Vue(What? Another Vue?)
- define `idToTemplate`
- define `getOuterHTML`
- define `Vue.prototype.$mount` which use `idTotemplate` and `getOuterHTML`
- define `Vue.compile`

The two important points are:

1. This is **NOT** the real Vue code, we should know that from the filename, it's just an entry
2. This file extracts the `$mount` function and defines a new `$mount`. After reading the new definition, we know it just add some validations before calling the real mount

## Next Step

Now you know how to find the entry of a brand new project. In next article, we will go on to find the core code of Vue.

Read next chapter: [Dig into the Core](https://github.com/numbbbbb/read-vue-source-code/blob/master/02-dig-into-the-core.md).

## Practice

Remember those "util functions"? Read their codes and tell what they do. It's not difficult, but you need to be patient. 

Don't miss `shouldDecodeNewlines`, you can see how they fight with IE.



# :books:참고자료

https://programmingsummaries.tistory.com/385

https://www.peterkimzz.com/extremely-faster-esbuild-than-webpack/
