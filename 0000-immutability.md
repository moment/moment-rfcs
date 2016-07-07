- Start Date: 2016-07-07
- RFC PR: (leave this empty)
- Moment Issue: [#1754][]

# Summary

Add a pseudo-immutable object API to the core Moment library, replicating
the full functionality of our current API.

# Motivation

[Many][#1754] new-to-Moment users expect that Moment will return a new
object from various methods that actually change the value of the original
object.  This expectation is consistent with the observation that Moment
instances are [Value Objects][ValueObject].  Notably, this expectation
seems to be strengthening over time, correlated with the rise of web
application and UI frameworks that rely on reference equality checks to
performantly determine whether a data structure has changed.

At the same time, Moment has been used to build millions of existing
applications, all of which rely on the library's current API design.
Many groups with existing codebases, especially those who already use Moment
pervasively, strongly value the API stability that Moment has maintained over
the years to date.  In addition, many developers coming to Moment from the
native Javascript Date API tend to expect an API where a single object's
value can change over time.

There is a plugin, [Frozen Moment][], that has attempted to address these
issues.  Unfortunately, as a third-party plugin it has limited visibility and
less certainty about long-term support, so it's not a satisfactory solution
for everyone wanting an "immutable" API.  More significantly, there isn't a
great way for a plugin to find out about the other Moment plugins that might
be in use in any particular environment, which makes it difficult to use
plugins while also using the frozen APIs.

To address these concerns, we could add a plugin registration system to the
core Moment library, coupled with a `moment.frozen` namespace that would
serve as a fully equivalent immutable API alternative to the primary `moment`
namespace and its various properties and constructors.

# Detailed design

There are two pieces to this proposal:  plugin registration and a "Frozen"
pseudo-immutable API that wraps the core library and its registered plugins.

For brevity, I'll refer to moment objects that use the current API as
**Mutables** throughout this section, and objects using the proposed
pseudo-immutable API will be called **Frozens**.  I'll also refer to the
prototype for Frozens as `moment.frozen.fn`, even though we probably don't
want to publish that prototype for third-party use.

## Plugin (Method and Constructor) Registration

A Frozen API can be implemented much more robustly if we ask plugins to
explicitly opt in to compatibility with Frozens.  Without this, Frozens can
only support the core library's API methods, or we need to somehow guess at
the behavior of other plugins so we can provide their methods on Frozens.

### New APIs for Plugin Authors

There are four types of functions that the Frozen API might need to know about:

1. factory functions (colloquially, "constructor" — e.g. `moment.tz()`)
2. mutation methods on the `moment.fn` prototype
3. mutation methods on the `moment.duration.fn` prototype
4. methods on the `moment.fn` prototype that return new non-Moment objects,
   and those new objects use Moment's APIs (e.g. moment-range, Twix.js)

In order to achieve this, we might ask (or require) plugin authors to use
Moment-provided features to add new prototype methods and factory functions
(rather than simply assigning functions to the `moment`, `moment.fn`, and
`moment.duration.fn` objects).  This could be achieved with three new static
methods:

#### `moment.addFactory(name, func, options)`

Example usage:

```js
moment.addFactory('tz', () => { return moment() });
moment.addFactory('twix', (m1, m2) => { return Twix(m1, m2) }, { returnsMoment: false });
```

This function assigns `func` to `moment` using the key `name`, so that future
calls to `moment.name()` will invoke `func`.

- If `options.returnsMoment === false`, we will also assign `func` to `moment.frozen`.
- Otherwise, we'll wrap `func` to construct and return a Frozen based on the
  Mutable created by `func`, and we'll assign that wrapped function to `moment.frozen`
  for the key `name`.

#### `moment.addMethod(name, func, options)`

Example usage:

```js
moment.addMethod('addSix', function() { return this.add(6) });
moment.addMethod('getSetMinutes', moment().minutes, { getterSetter: true });
moment.addMethod('twix', function(end) { return Twix(this, end) }, { returnsMoment: false });
```

This function assigns `func` to `moment.fn` using the key `name`, so that future
calls to `moment().name()` will invoke `func`.

- If `options.returnsMoment === false`, we will also assign `func` to `moment.frozen.fn`.
- If `options.getterSetter === true`, we'll wrap `func` with a function that
  clones `this` before invoking `func` if (and only if) `arguments.length > 0`.
  That wrapped function will be assigned to `moment.frozen.fn` for the key `name`.
- Otherwise, we'll wrap `func` with a function that always clones `this` before
  invoking `func`, and assign that function to `moment.frozen.fn` for the key `name`.

Thus, we assume that any new prototype method returns a mutated Mutable,
unless the plugin tells us otherwise by setting an appropriate flag in the
optional `options` object.

#### `moment.duration.addMethod(name, func, options)`

```js
moment.duration.addMethod('addSix', function() { return this.add(6) });
moment.duration.addMethod('getSetMinutes', moment.duration().minutes, { getterSetter: true });
moment.duration.addMethod('toInteger', function() { return this.asMilliseconds() }, { returnsMoment: false });
```

This is the same as `moment.addMethod()`, except now we're assigning to 
`moment.duration.fn` and `moment.frozen.duration.fn`.  In loquacious detail:

This function assigns `func` to `moment.duration.fn` using the key `name`, so that future
calls to `moment.duration().name()` will invoke `func`.

- If `options.returnsMoment === false`, we will also assign `func` to `moment.frozen.duration.fn`.
- If `options.getterSetter === true`, we'll wrap `func` with a function that
  clones `this` before invoking `func` if (and only if) `arguments.length > 0`.
  That wrapped function will be assigned to `moment.frozen.duration.fn` for the key `name`.
- Otherwise, we'll wrap `func` with a function that always clones `this` before
  invoking `func`, and assign that function to `moment.frozen.duration.fn` for the key `name`.

Thus, we assume that any new prototype method returns a mutated Mutable,
unless the plugin tells us otherwise by setting an appropriate flag in the
optional `options` object.

#### Implementation Notes

Ideally these methods would also be used internally to register Moment's core
factories and methods with the Frozen prototype.

All plugins will be entirely compatible with the Frozen API proposal if they
correctly register all of their functionality with Moment using these three
methods, provided that all of their new factories and methods return Mutables.
This describes the vast majority of the existing Moment plugin ecosystem.

Plugins that add factories or methods that return custom objects (*not* Mutables,
e.g. the range plugins) will need to do some additional work to function
correctly when called from Frozens. <sup id="a1">[1](#fn1)</sup>

We would need to add a new section for "Plugin Authors" to the documentation to
describe these methods and their intended usage.

If we decide want to actively push new users toward using Frozens instead of
Mutables, then I think we should probably try to simplify the user-facing
plugin compatibility story as much as possible by requiring plugins to use
these APIs if they want to work with Moment 3.0+.  This could be easily
achieved for methods if we stop publicly exposing the Moment and Duration
prototypes as `moment.fn` and `moment.duration.fn`, respectively. <sup id="a2">[2](#fn2)</sup>
It's a bit harder to enforce for factories, so it still might not be worth
enforcing that case, but one strategy could be to construct the `moment`
object with a prototype, and then apply `Object.freeze()` to the moment
object itself.  Then our implementation of `addFactory()` could assign
functions to the `moment` object's prototype to make them publicly available.

Depending on how hard we push this API on plugin authors, many plugins might
not proactively adopt this registration API even though it should be very easy
for most plugins to adopt.  We might want to send PRs to some high-profile
plugins to make this transition easy for their authors.

### Plugin vs Core Feature

I think it might be somewhat confusing for users and plugin authors if we
implement these plugin registration functions as a feature that appears in
some environments but not others — which points toward including this piece
as a feature of the core library rather than building it in a plugin.

If plugin registration is part of a first-party Frozen Moment plugin and not
part of the core library, then the `addMethod` and `addFactory` APIs would
only be available in the `moment.frozen` namespace (and not the top-level
`moment` namespace).  Registered methods would simply be added to the Frozen
API — plugins would continue to extend `moment` and `moment.fn` directly
for the Mutable API.

This system would certainly ensure backward compatibility with existing plugins
— although we don't **have** to break the way existing plugins currently work,
even if we do build plugin registration APIs into the core library and
encourage plugin authors to use these new APIs.  On the other hand, I suspect
that having entirely different systems for plugins to work with Mutable vs
Frozen APIs would ultimately reduce the number of community plugins that choose
to support the Frozen API, assuming that both approaches would be accompanied
by similar levels of support and outreach effort from the Moment maintainers.

If this registration system is part of an optional plugin, it's essential for
users to load Frozen Moment before any other plugins that they want to use with
the Frozen API, since the plugin registration system won't exist at all until
the Frozen plugin has been loaded. <sup id="a3">[3](#fn3)</sup>  If that
optional plugin is a separate file, then some number of users will get the
bundling order wrong and file support requests.  On the other hand, if we
distribute a bundled build where the core library and the Frozen plugin appear
in the same file, then we need to decide which is the default build for the big
download button at momentjs.com, and we also need to figure out how to
communicate the distinctions between different builds now that we'd (presumably)
have twice as many different pre-built packages available.

I'm tempted to say we should sidestep all this build + communication complexity
by building everything from this RFC into the core library, but I'm not 100%
confident that's the correct tradeoff.  We probably need some community feedback
(particularly from plugin authors) before locking down the plugin vs core decision.

## Frozen API

### New APIs for Moment Users

This proposal adds `moment.frozen` as a new API namespace.

Users who want to work exclusively with an immutable instance API (the
"Frozen API") would be able to ignore all Moment features that are not part
of the `moment.frozen` namespace.  If they're not using third-party code,
such users might want to simply overwrite the `moment` global in a traditional
web environment (`moment = moment.frozen`), or import just the frozen APIs
while discarding the traditional mutable APIs when working in a CommonJS
environment (`var moment = require('moment').frozen`), etc.

Users who want to work exclusively with the current mutable instance API
(the "Mutable API") would be able to ignore the new `moment.frozen`
namespace without consequence.

Users would also be able to mix and match APIs within the same application,
allowing existing applications to incrementally migrate from one API to the
other by using both APIs as appropriate, and then parsing Mutables from
Frozens (or vice-versa) at the boundaries to the modified code, as
appropriate.

So, from a user's perspective:

- All `moment.*` static configuration methods will be shamelessly copied
  as-is into the `moment.frozen.*` namespace, using the same method names
  in both namespaces.  Note that this includes the new plugin registration
  methods described in the previous section.

- All factory methods in the `moment` namespace will be wrapped for use
  (under the same method names) in the `moment.frozen` namespace.  These
  wrapped factories will return instances that use the Frozen API.  For
  example, the wrapped equivalent of `moment.parseZone()` would be
  `moment.frozen.parseZone()`.

  When we say "all factory methods", this includes factory methods from
  the core library as well as factory methods from plugins that have been
  registered using the `moment.addFactory()` API described earlier.
  Note that `moment.frozen()` must itself be a wrapped version of the
  core library's `moment()` factory method, in additon to wrapping
  factories that are static methods of `moment` (e.g. `moment.utc`,
  `moment.tz`, etc).

- Frozens created from factory methods in the `moment.frozen` namespace
  will be exactly the same as Mutables created from the existing factories,
  except that they will be backed by the `moment.frozen.fn` prototype
  instead of the `moment.fn` prototype (or `moment.frozen.duration.fn`
  instead of `moment.duration.fn`, for Durations).

- Instance methods in the Frozen API (from `moment.frozen.fn` and
  `moment.frozen.duration.fn`) are wrapped versions of the corresponding
  methods from the Mutable API.  The wrappers simply create a copy of the
  object (when appropriate) before delegating to the Mutable API
  implementation of that function name.

  All core methods will be available on Frozens, as well as all
  plugin-created methods that were registered using the `addMethod` APIs
  described above.

### Implementation Plan

A proposal for bootstrapping the `moment.frozen` namespace as described above:

1. First, create `moment.frozen` as a wrapped version of the `moment`
   factory function.  We can't use `moment.addFactory()` for this, so it
   will be hardcoded.  If everything is in core, we create `moment.frozen`
   immediately after creating `moment`, and before registering all the
   core methods onto the Mutable prototype.  If this RFC is a separate
   plugin, then this is the first thing that happens in the plugin's
   implementation.
2. Ditto for Durations:  we need hardcoded logic to create
   `moment.frozen.duration` as a wrapped version of the
   `moment.duration` factory, and we need that code to run as early
   as possible.
3. Create `addFactory` as a method of `moment` (if this is part of core)
   and `moment.frozen`.  Ditto for `moment.addMethod` and
   `moment.duration.addMethod`.
4. Use `moment.frozen.addFactory()` to register all factory functions
   and static config methods from the core library.  (Static config
   methods would be registered with `returnsMoment: false`.)
   `moment.frozen.addFactory` is responsible for wrapping and copying
   those methods into the `moment.frozen` namespace.
5. Use `moment.frozen.addMethod()` and `moment.frozen.duration.addMethod()`
   to register all Mutable instance methods from the core library.
   The `addMethod` methods are responsible for wrapping and copying
   those methods into the appropriate Frozen namespace, same as described
   below for plugin methods.

The Frozen API would be implemented with a separate prototype
(`moment.frozen.fn` vs `moment.fn`) for Frozens than for Mutables.
Similarly, there would be a separate prototype for Frozen Durations
(`moment.frozen.duration.fn`) than for Mutable Durations
(`moment.duration.fn`).

Each time a plugin calls `moment.addMethod`, the provided function
will be wrapped (if appropriate), and then the wrapped copy will be
assigned for the specified function name on `moment.frozen.fn`.
<sup id="a4">[4](#fn4)</sup>

Ditto for `moment.duration.addMethod`:
Each time a plugin calls `moment.duration.addMethod`, the provided function
will be wrapped (if appropriate), and then the wrapped copy will be
assigned for the specified function name on `moment.frozen.duration.fn`.

### Plugin vs Core Feature

This could be implemented directly in core, or with a first-party plugin
(which could be bundled into certain distribution builds for convenience).

The Frozen API implementation will need to hook into the behavior of the
`addFactory` and `addMethod` APIs.  This is trivial if everything is in core
(or if everything is in a plugin).  If the plugin registration system is in core
but the Frozen API is not, then we'll want some mechanism for the plugin to
register a callback so it can do the right thing when `addFactory` and `addMethod`
get called.  We'd also need to hardcode wrappers for core methods, and/or build a
way for the plugin to get a list of previously-registered methods at the same
time as the plugin sets up these callbacks.

That said, it might make sense to set a maximum file size increase that we're
willing for these features to impose on the core library, and use that cutoff to
determine which path is best (after writing a rough initial implementation in core).

# How We Teach This

We probably need a dedicated page at momentjs.com introducing users to the
semantic distinctions between the Mutable and Frozen APIs.

We also need to describe the `addFactory` and `addMethod` functions for the
API reference documentation at momentjs.org/docs.

It would feel pretty heavy to just append (largely duplicative) documentation
for all the `moment.frozen.*` APIs to the existing documentation page.
If these APIs are part of a plugin then maybe we need a dedicated subsite
for the plugin (similar to Moment Timezone) with its own copy of the docs
(although hopefully the docs could be automatically generated for both sites
from the same source material, since most of the info would be the same).
If it's all part of core, and/or if we're distributing bundled builds with
core + plugin in a single file, then maybe there could be a toggle switch
to convert between Mutable and Frozen API docs (similar to how some
Coffeescript libraries let you toggle between .js and .coffee syntax in
their documentation's code examples).

Finally, we'll need some targeted documentation explaining how and
(especially) why plugin authors should register their methods for use with
the Frozen API.  We might consider submitting PRs to a few popular plugins
to help them adopt the new registration APIs, and then link to those PRs
from the documentation as examples of how other plugin authors can update
their implementations.

# Drawbacks

1. Doubling the number of exposed API methods (for "frozen" and traditional
   versions of everything) will inevitably increase our documentation and
   support burdens, even if almost all of the code is the same.  It's also
   likely to increase the testing burden for new feature additions from
   maintainers and external contributors alike, although hopefully the
   impact on *implementing* new features will be negligible.
2. By giving developers a choice of frozen/mutating APIs, new users of
   Moment will have more friction getting started with the library because
   they'll need to choose an API before they can do anything useful.
3. This feature has often been described as an "immutable" API, but we're
   not actually making the objects immutable — developers can assign and
   modify custom properties on these objects as much as they want.  Indeed,
   "frozen" instances would still be mutating under the hood in certain
   situations (because some internal properties are lazily computed).  This
   may be confusing for some developers and we will likely get questions
   about it.
4. We expect these changes will increase the library's file size somewhat,
   even for users who don't use any of the new APIs, simply because we're
   adding code to the library.
5. Because plugin methods and factories are created with string names,
   minifiers won't be able to rename those methods, possibly leading to an
   additional slight increase in file sizes for folks who minify their
   dependencies together with their custom code.
6. The new plugin registation system will slightly increase the time
   required for Moment and its plugins to initialize themselves for use.
   I don't expect this to be a noticeable issue, though.

# Alternatives

As hinted in the Motivation section, if we don't do this (or something
similar), then we're effectively asking the folks who keep +1'ing [#1754][]
to build their own community around the existing third-party plugin, or to
find or build another library instead of using Moment.  This would not be the
end of the world, but it would be a little frustrating since we agree that
date-time objects should ideally be an immutable type.

To date, the current third-party Frozen Moment plugin has had much less reach
than we might expect for a first-party solution.  We might be able to close
that gap somewhat by promoting the third-party plugin more heavily on
momentjs.com, since it doesn't appear to be mentioned anywhere in the official
documentation right now.  This sort of cross-promotion scheme could achieve
some of the same usage benefits while delegating much of the communication
and maintenance burden for this functionality outside the core Moment team —
at the cost of continuing with some additional perceived and actual risk
(relative to a first-party solution) as to whether that plugin would continue
to be maintained appropriately over the long term.

There is no technical reason why the current third-party plugin could not grow
to do everything that a first-party plugin would do under this RFC, including
plugin registration and distributing bundled builds.  In fact, I will likely
work to make that happen if this RFC is not adopted.  But any third-party solution
will be handicapped by the fact that users don't expect plugins to be able to
override core API decisions like this, so they're less likely to find and adopt
a third-party plugin for this purpose (even if that plugin was well-publicized,
etc).

I also hope that it will become easier to get other plugin authors to write
compatible plugins (by using the registration system, etc) if Frozen Moment is
not itself "just" another third-party plugin.

# Unresolved questions

1. How much of this functionality should be implemented in the core library
   vs a first-party plugin?  Will we want to offer additional pre-compiled
   builds for users to download?  If so, which is the default "recommended"
   build publicized most heavily at momentjs.com, and how do we describe the
   differences for our alternative builds?
2. Will we ever want to proactively encourage new users to use the frozen
   APIs instead of the traditional APIs?  If so, when and why?  If not, how
   do we make sure that new users approaching our documentation will
   understand their options without first writing a bunch of code against the
   wrong API and getting stuck and coming to us for help?
3. Do we want to avoid publishing our prototypes in 3.0 and force plugin
   authors to adopt the new registration API, or should we just strongly
   encourage plugins to update without breaking current plugins?
4. The documentation strategy needs to be fleshed out further, as we get a
   better sense of our answers to the other unresolved questions.

# Footnotes

1. <a id="fn1"></a> Thankfully I don't think moment-range uses mutator methods
   much, if at all, so it should be easy to update.  It looks like most of
   Twix's functionality should also be trivial to update, although its range
   iterators would need to be reimplemented. [↩](#a1)

2. <a id="fn2"></a> Technically, third-party code could still sidestep our
   registration API by messing with `Object.getPrototypeOf(moment())`.
   At that point, though, it's easier to just use our APIs instead, so I don't
   think any plugin authors would actually do that.  It's not like we could
   possibly stop someone who is **that** motivated to muck with our internals,
   anyway. [↩](#a2)

3. <a id="fn3"></a> If the registration system is in core but the
   APIs are not, then the registration system could keep track of metadata about
   all the plugin methods that have already been registered.  Then any plugins
   that want to hook into that system could also retrieve the metadata for all
   the methods that were registered before their plugin was loaded. [↩](#a3)

4. <a id="fn4"></a> Thus, unlike the existing third-party Frozen Moment
   plugin, `moment.frozen.fn` will **not** have `moment.fn` as its prototype.
   This means that all Frozen API methods must explicitly appear on the Frozen
   prototype, instead of simply falling through to the Mutable prototype for
   methods that are not expected to mutate the object.  This change is made to
   ensure correctness; this way, unregistered plugin methods will not appear on
   Frozens so that they cannot accidentally mutate the value of a Frozen.
   (The same is true for `moment.frozen.duration.fn`: it will not have
   `moment.duration.fn` as its prototype for the same reason.) [↩](#a4)


[#1754]: https://github.com/moment/moment/issues/1754
[Frozen Moment]: https://github.com/WhoopInc/frozen-moment/
[maggiepint post]: https://maggiepint.com/2016/06/24/why-moment-js-isnt-immutable-yet/
[ValueObject]: http://martinfowler.com/bliki/ValueObject.html
