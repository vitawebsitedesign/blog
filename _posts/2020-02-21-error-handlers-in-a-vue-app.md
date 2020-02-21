---
title: Error handlers in a Vue app
layout: post
author: Michael Nguyen
---

To handle all error types in a Vue app, you'll need to use Vue's error handler:
* [Vue.config.errorHandler](https://vuejs.org/v2/api/#errorHandler)

**in addition** to the 2 usual JavaScript ones:
* [window.addEventListener](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener) for "unhandledrejection"
* [window.onerror](https://developer.mozilla.org/en-US/docs/Web/API/GlobalEventHandlers/onerror)

This post summarizes these.

# Vue.config.errorHandler
Covers vue & (now) promise errors. Think of:
* Vue created/mounted errors-
* Thrown objects in promises

```javascript
main.ts:
import Vue from 'vue';

Vue.config.errorHandler = function(err, vm, info) {
  const errMsg = err && err.toString();
  throw new Error(errMsg);	// Handle it anyway you want, like show a toastbar
};
```

```javascript
MyComponent.vue:
import { Component, Vue } from 'vue-property-decorator';
@Component
export default class MyComponent extends Vue {
	created() {	
		throw new Error('x');	// <-- !
	}
}
```

# Window.addEventListener
This function can help us covers *EXPLICIT* promise rejections. The event name for those events is "unhandledrejection".

The below example is async/await (which is what you'll probably be using these days):

```javascript
main.ts:
...
window.addEventListener('unhandledrejection', function(event) {
  const errMsg = event && event.reason && event.reason.toString();
  throw new Error(errMsg);
});
```

```javascript
MyComponent.vue:
import { Component, Vue } from 'vue-property-decorator';
@Component
export default class MyComponent extends Vue {

	async created() {
		await doSomething();	// With the error handler, dont need to wrap this call in a try-catch!
	}

	private async doSomething(async (resolve, reject) => {
		try {
			...
		} catch (e) {
			reject(e);	// <-- !
		}
	});
}
```

# Window.onerror
Covers non-Vue non-promise javascript stuff. Think of vanilla JavaScript console errors (e.g.: syntax errors, errors in non-Vue lifecycle hooks):

```javascript
main.ts:
...
window.onerror = function(msg, url, line, col, error) {
  // "msg" can be an Error or string
  let err: Error | null = null;
  if (msg instanceof Error) {
    err = msg;
  } else {
    const errMsg = error && error.toString();
    err = new Error(errMsg);
  }

  throw err;
};
```

```javascript
MyComponent.vue:
<template>
	<div @click="x" />
</template>
import { Component, Vue } from 'vue-property-decorator';
@Component
export default class MyComponent extends Vue {
	x() {
		setTimeout(() => {	// <-- !
			throw new Error('x');
		}, 1);	
	}
}
```

Once our component friend above renders, all bets are off.

So you click the `<div/>` after Vue's render stage, and the vanilla javascript error handler will be needed to catch it.

# Summary
Shove these 3 in your app as early as possible, & before instantiate the Vue instance w/ `new Vue`:
```javascript
main.ts:
import Vue from 'vue';

Vue.config.errorHandler = function(err, vm, info) {
  const errMsg = err && err.toString();
  throw new Error(errMsg);
};

window.addEventListener('unhandledrejection', function(event) {
  const errMsg = event && event.reason && event.reason.toString();
  throw new Error(errMsg);
});

window.onerror = function(msg, url, line, col, error) {
  // "msg" can be an Error or string
  let err: Error | null = null;
  if (msg instanceof Error) {
    err = msg;
  } else {
    const errMsg = error && error.toString();
    err = new Error(errMsg);
  }

  throw err;
};

new Vue(...).$mount(...);
```

# Closing notes
* The primary difference between `Error` and `string` is that `Error` has a nice juicy stack trace. So you wanna be throwing `Error` like in the above examples.
* Vue's error handling responsibility is all about **rendering**. Outside of that, all bets are off. Thats why you need **3** error handlers, even though they look similar.
