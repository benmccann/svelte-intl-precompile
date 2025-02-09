## Svelte-intl-precompile

This i18n library for Svelte.js has an API identical (or at least very similar) to https://github.com/kaisermann/svelte-i18n but has
a different approach to processing translations.

Instead of doing all the work in the client, much like Svelte.js acts as a compiler for your app, this library acts as a compiler
for your translations.

### Why would I want to use it? How does it work?
This approach is different than the taken by libraries like intl-messageformat or format-message, which do all the work in the browser. The approach taken by those libraries is more flexible as you can just load json files with translations in plain text and that's it, but it also means the library needs to ship a parser for the ICU message syntax, and it always has to have ship code for all the features that the ICU syntax supports, even features you might not use, making those libraries several times bigger.

This process spares the browser of the burden of doing the same process in the user's devices, resulting in smaller and faster apps.

For instance if an app has the following set of translations:
```json
{
  "plain": "Some text without interpolations",
  "interpolated": "A text where I interpolate {count} times",
  "time": "Now is {now, time}",
  "number": "My favorite number is {n, number}",
  "pluralized": "I have {count, plural,=0 {no cats} =1 {one cat} other {{count} cats}}",
  "pluralized-with-hash": "I have {count, plural, zero {no cats} one {just # cat} other {# cats}}",
  "selected": "{gender, select, male {He is a good boy} female {She is a good girl} other {They are good fellas}}",
}
```

The babel plugin will analyze and understand the strings in the ICU message syntax and transform them into something like:
```js
import { __interpolate, __number, __plural, __select, __time } from "precompile-intl-runtime";
export default {
  plain: "Some text without interpolations",
  interpolated: `A text where I interpolate ${__interpolate(count)} times`,
  time: now => `Now is ${__time(now)}`,
  number: n => `My favorite number is ${__number(n)}`,
  pluralized: count => `I have ${__plural(count, { 0: "no cats", 1: "one cat", h: `${__interpolate(count)} cats`})}`,
  "pluralized-with-hash": count => `I have ${__plural(count, { z: "no cats", o: `just ${count} cat`, h: `${count} cats`})}`,
  selected: gender => __select(gender, { male: "He is a good boy", female: "She is a good girl", other: "They are good fellas"})
}
```

Now the translations are either strings or functions that take some arguments and generate strings using some utility helpers. Those utility helpers are very small and use the native Intl API available in all modern browsers and in node. Also, unused helpers are tree-shaken by rollup.

When the above code is minified it will results in an output that compact that often is shorter than the original ICU string:

```
"pluralized-with-hash": "I have {count, plural, zero {no cats} one {just # cat} other {# cats}}",
--------------------------------------------------------------------------------------------------
"pluralized-with-hash":t=>`I have ${jt(t,{z:"no cats",o:`just ${t} cat`,h:`${t} cats`})}`
```

The combination of a very small and treeshakeable runtime with moving the parsing into the build step results in an extremely small footprint and
extremely fast performance.

**How small, you may ask?** 
Usually adds less than 2kb to your final build size after compression and minification, when compared with nearly 15kb that alternatives with
a runtime ICU-message parser like `svelte-i18n` add.

**How fast, you may also ask?** 
When rendering a key that has also been rendered before around 25% faster. For initial rendering or rendering a keys that haven't been rendered 
before, around 400% faster.

### Setup
Install `svelte-intl-precompile` as a runtime dependency.

Create a folder to put your translations. I like to use a `/messages` or `/translations` folder on the root. On that folder, create `en.js`, `es.js` and as many files as languages you want. On each file, export an object with your translations:
```js
 export default {
   "recent.aria": "Find recently viewed tides",
   "menu": "Menu",
   "foot": "{count} {count, plural, =1 {foot} other {feet}}",
 }
 ```

In your `svelte.config.cjs` import the function exported by `svelte-intl-precompile/sveltekit-plugin` and invoke with the folder where you've placed
your translation files it to your list of Vite plugins:
```js
const node = require('@sveltejs/adapter-node');
const pkg = require('./package.json');
const precompileIntl = require("svelte-intl-precompile/sveltekit-plugin");

/** @type {import('@sveltejs/kit').Config} */
module.exports = {
	kit: {
		adapter: node(),
		target: '#svelte',
		vite: {
			ssr: {
				noExternal: Object.keys(pkg.dependencies || {})
			},
            plugins: [
                precompileIntl('locales')
            ]			
		}
	}
};
```

From this step onward the library almost identical to use and configure to the popular `svelte-i18n`. It has the same features and only the import path is different. You can check the docs of `svelte-i18n` for examples and details in the configuration options.

Now you need some initialization code to register your locales and configure your preferences. You can import your languages statically (which will add them to your bundle) or register loaders that will load the translations lazily. The best place to put this configuration is inside a `<script type="module">` on your `src/$layout.svelte`

```html
<script context="module">
	import { addMessages, init /*, register */ } from 'svelte-intl-precompile';
    import en from '../../locales/en.js';
	// @ts-ignore
    addMessages('en', en);
    // register('es', () => import('../../locales/en.js')); <-- use this approach if you want locales to be load lazily


    init({
		fallbackLocale: 'en',
        initialLocale: { navigator: true }
	});
</script>

<script>
	import '../app.css';
</script>

<slot />
```

Now on your `.svelte` files you start translating using the t store exported from precompile-intl-runtime:
```html
<script>
  import { t } from 'svelte-intl-precompile'
</script>
<footer class="l-footer">
  <p class="t-footer">{$t("hightide")} {$t("footer.cta")}</p>
</footer>
```
