Start Date: 2018-06-30
RFC PR: [#4][pull request]
Moment Issue: too many


Moment packaging
================

## Scope

This document will discuss the requirements of moment in terms of packaging and
consumtion from node.js, webpack, rollup, require.js, simple browser
environments, ES2016 module systems and the like.

## Introduction

Moment.js was created in a time when no module systems for JavaScript were
widely adopted. Later on, CJS and AMD modules became the two competing
standards, and moment adopted the UMD style, so it is easily consumed from
browsers, node.js and packaging systems like browserify and require.js.

Later on ES2015 and with it various ESM frameworks became widely popular, and
some of moments features (like locale auto-loading on node) became problematic for
out-of-the-box consumption on those systems.

The purpose of this document is to specify how moment should work in the most
prominent use-cases in a perfect world. Then we'll try to come up with
a solution that is as close, by being able to evaluate all strengths and
weaknesses according to the doc.

## Abstract moment use-cases

No matter the packaging, moment users have to choose one of the following:
* **core-only** load moment-core, without any locales (en only)
* **all-locales** load moment-core with **all** locales included (available for immediate
  switch)
* **single-locale** load moment-core with a single additional locale (en is included)
* **dynamic-locales** load moment-core only, and dynamically choose a locale, download and use it

Here **dynamic-locales** could be though of as **core-only** + dynamic loading
on demand.

## First class environments

Moment should strive to work effortlessly on all environments, especially the
ones that enjoy the biggest userbases.

In the time of this writing, the most popular environments are (in no
particular order):
* node.js
* browser (using just script tag)
* webpack
* parcel
* rollup
* browserify
* require.js

Some of these systems support multiple module standards (like CJS, AMD and
ESM), while others don't. Also the support varies from platform to platform and
is often inconsistent.

All first class environments need to have a proof-of-concept code that covers
the 4 abstract use cases. These need to be tested for correctness on every
moment release. New environments should be easy to add. Multiple versions of
the same environment could be needed (as they are often major differences in
between).

## Include/require paths

Moment might need to expose an UMD and ESM modules separately, so it needs to
be decided under what path it will be available.

Suggested paths:
    * `moment` -- moment core
    * `moment/locale/XX` -- locale files
    * `moment/locale/all`-- bundle of all locale files (optional)

If both UMD and ESM modules could use the same names (but somehow have different
root, configured in package.json), that would be ideal.

## Other remarks/questions

* How should locales and moment plugins get a reference to moment?
    - load moment with require/import `moment`
* Should all moment plugins export themselves in UMD and ESM formats (as is the
  case for moment)
* Should UMD/ESM versions of moment live in the same repo?
* Auto-requiring locales on node.js (when users selects locale that hasn't been
  `loaded`, moment could try to load/require it on demand)
    - node predominantly runs on CJS, so doing a `require` might do the trick,
      currently this confuses webpack, but webpack could use the ESM version of
      moment. What if a user of webpack is also using CJS -- same issue?
      Could we use special entry point for node.js only?
* In addition to the paths/entry points discussed above, moment could also
  provide minified versions of some/all of those files
    - different folder with minified UMD versions -- mostly to be used for
      direct browser consumption?
* Is there such a thing as extended UMD (some libs nowdays use exports['name']
  to trick other ESM systems to play better with them, module.exports is for
  CJS), and could it be useful to moment.
* How is all that going to be build
    - have source in ESM, and then use webpack (or any other) to prepare
      ESM/UMD versions?
* How should we test platform compatibility
    - have a repo with tests trying a simple smoke test loading moment/locales,
      that is executed with every release

## Baby moment

I suggest creating a moment stub to test around various proposals.

```
bmoment.version === 'baby moment vnext';
bmoment() // create a moment object for now
bmoment(unixMillis) // create from unix millis
bmoment().valueOf() // return unix epoch milliseconds
bmoment.defineLocale('XX', { hi: 'str' }); // locale will only have translation
for hi
bmoment.locale('XX'); // switch global locale to 'XX', default is 'en', don't
chane if there is no such locale loaded (node.js may auto-load)
bmoment.locale(); // return the current global locale
bmoment().hi(); // return the hi string for the current global locale
```
