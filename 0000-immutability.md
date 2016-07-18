- Start Date: 2016-07-07
- RFC PR: [#2][pull request]
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
plugins while also using the immutable APIs.

To address these concerns, we could add a plugin registration system to the
core Moment library, coupled with a `moment.immutable` namespace that would
serve as a fully equivalent immutable API alternative to the primary `moment`
namespace and its various properties and constructors.


# Detailed design

There are two pieces to this proposal:

1. a user-facing "Immutable" API (actually pseudo-immutable) that wraps the
   core library and its registered plugins;
2. mechanisms for plugin authors to easily extend Moment, with feature parity
   for both the "Mutable" and "Immutable" APIs.

For brevity, I'll refer to moment objects that use the current API as
**Mutables** throughout this section, and objects using the proposed
pseudo-immutable API will be called **Immutables**.  `moment.immutable.fn`
is the prototype for Immutables.


## New APIs for Moment Users

This RFC adds two new API namespaces: `moment.mutable` and `moment.immutable`.

The top-level `moment` namespace might be a reference to `moment.mutable` or
`moment.immutable` for convenience, and this might differ between build
configurations. Thus, we might offer both a Compatibility build where `moment`
APIs are equivalent to `moment.mutable`, and a Recommended build where `moment`
APIs are equivalent to `moment.immutable`.

Users who want to work exclusively with an immutable instance API (the
"Immutable API") would be able to work solely with the `moment.immutable`
namespace, and those who don't want to change their code can simply use the
`moment.mutable` namespace.  Third-party code (e.g. UI components) will always
be able to rely on having their desired API available.

If they're not using third-party code, users might want to simply overwrite the
`moment` global in a traditional web environment (`moment = moment.immutable`)
to reflect the API that they are using.  Similarly, I expect it'd be common to
import just one API while discarding the other API when working in a CommonJS
environment (`var moment = require('moment').immutable`), etc.

That said, users would also be able to mix and match APIs within the same
application, allowing existing applications to incrementally transition between
APIs.  We will strongly discourage users from mixing and matching from both
APIs within a single codebase as a long-term design choice.


So, from a user's perspective:

- All `moment.*` static configuration methods will be shamelessly copied
  as-is into the `moment.mutable` and `moment.immutable.*` namespaces, using
  the same method names in all three namespaces.  Note that this includes the
  new plugin registration methods described in the previous section.

  (Or maybe our static configuration methods should only live in the top-level
  `moment` namespace?  But then I'd kind of want to force everyone to always
  explicitly construct instances from the `mutable` or `immutable` namespaces,
  with no top-level convenience wrappers there...)

- All factory methods in the `moment` namespace will be wrapped for use
  (under the same method names) in the `moment.immutable` namespace.  These
  wrapped factories will return instances that use the Immutable API.  For
  example, the wrapped equivalent of `moment.mutable.parseZone()` would be
  `moment.immutable.parseZone()`.

  When we say "all factory methods", this includes factory methods from
  the core library as well as factory methods from plugins that have been
  updated to support the new API.  Note that `moment.immutable()` must itself
  be a wrapped version of the core library's `moment.mutable()` factory method,
  in additon to wrapping factories that are static methods of `moment.mutable`
  (e.g. `moment.mutable.utc`, `moment.mutable.tz`, etc).

- Immutables created from factory methods in the `moment.immutable` namespace
  will be exactly the same as Mutables created from the existing factories,
  except that they will be backed by the `moment.immutable.fn` prototype
  instead of the `moment.mutable.fn` prototype (or
  `moment.immutable.duration.fn` instead of `moment.mutable.duration.fn`
  for Durations).

- The `moment.mutable()` factory method (and likely all other `moment.mutable.*`
  factories) will be able to accept a Immutable as input to construct a Mutable
  with the same data as the input Immutable.  Likewise, the `moment.immutable()`
  factory (and likely all other `moment.immutable.*` factories) will be able to
  accept a Mutable in order to construct a Immutable with the same data.

  The `freeze()` and `thaw()` instance methods from the existing Frozen Moment
  plugin will **not** be implemented, because they are not needed.

- Instance methods in the Immutable API (from `moment.immutable.fn` and
  `moment.immutable.duration.fn`) are wrapped versions of the corresponding
  methods from the Mutable API.  The wrappers simply create a copy of the
  object (when appropriate) before delegating to the Mutable API
  implementation of that function name.

  All core methods will be available on Immutables, as well as all
  plugin-created methods that were registered using the `addMethod` APIs
  described above.

- The return value of `moment.isMoment()` could be changed to a tri-state enum:
  `"mutable"` if the input's prototype is `moment.mutable.fn`; `"immutable"` if
  the input's prototype is `moment.immutable.fn`; otherwise `false`.


## API Changes for Plugin Authors

A Immutable API can be implemented much more robustly if we ask plugins to
explicitly opt in to compatibility with Immutables.  Without this, Immutables
can only support the core library's API methods, or we need to somehow guess at
the behavior of other plugins so we can provide their methods on Immutables.

There are six types of functions that plugins could provide, where the plugin
implementations could be expected to differ between the Mutable and Immutable
APIs:

1. moment factory functions (colloquially, "constructor" — e.g. `moment.tz()`)
2. mutation methods on the `moment.fn` prototype
3. duration factory functions (e.g. `moment.duration()`)
4. mutation methods on the `moment.duration.fn` prototype
5. methods on the `moment.fn` prototype that return new non-Moment objects,
   and those new objects use Moment's APIs (e.g. moment-range, Twix.js)
6. wrapped copies of the `moment()` factory function (e.g. moment-jalaali,
   moment-hijri)

For these use cases #1 through #4, we'll provide utility methods on `moment`
that generate an Immutable method implementation from a Mutable plugin function.
These same wrapper-generation functions might also be of some limited help for
certain plugins in category #5 (e.g. Twix).  At the end of the day, plugins will
be required to explicitly assign their methods into the `moment.mutable` and
`moment.immutable` namespaces as appropriate; the wrapper functions below are
simply provided as a convenience to make it easier for plugin authors to support
both APIs simultaneously.

Otherwise, we can't do too much to help plugins that do #5 and #6 (aside from
maybe creating new extensibility APIs so that the calendar plugins can stop
wrapping the magic `moment()` factory, but that's way out of scope for this
RFC).  For those use cases, we should probably just aim to get out of the way
and let the plugins sort themselves out.

### `moment.wrapMomentFactory(func, destinationAPI)`

Example usage:

```js
moment.immutable = moment.wrapMomentFactory(moment.mutable, 'immutable');
moment.mutable.tz = moment.wrapMomentFactory(moment.immutable.tz, 'mutable');
```

Behavior:

```js
function wrapMomentFactory(func, api) {
  return function() {
    return moment[api](func.apply(null, arguments));
  };
}
```

### `moment.wrapDurationFactory(func, destinationAPI)`

Example usage:

```js
moment.immutable.duration = moment.wrapDurationFactory(moment.mutable.duration, 'immutable');
```

Behavior:

```js
function wrapMomentFactory(func, api) {
  return function() {
    return moment[api].duration(func.apply(null, arguments));
  };
}
```

### `moment.wrapMomentMethod(func, destinationAPI)`

Example usage:

```js
moment.immutable.add = moment.wrapMomentMethod(moment.mutable.add, 'immutable');
```

Behavior when `destinationApi` is `"immutable"`:

```js
function wrapMomentMethod(func, 'immutable') {
  return function() {
    return moment.immutable(func.apply(moment.mutable(this), arguments));
  };
}
```

Behavior when `destinationApi` is `"mutable"`:

```js
import { copyConfig } from './constructor';

function wrapMomentMethod(func, 'mutable') {
  return function() {
    var newMoment = func.apply(moment.immutable(this), arguments);
    copyConfig(this, newMoment);
    return this;
  };
}
```

### `moment.wrapDurationMethod(func, destinationAPI)`

Example usage:

```js
moment.immutable.duration.add = moment.wrapMomentMethod(moment.mutable.duration.add, 'immutable');
```

Behavior when `destinationApi` is `"immutable"`:

```js
function wrapMomentMethod(func, 'immutable') {
  return function() {
    return moment.immutable.duration(func.apply(moment.mutable.duration(this), arguments));
  };
}
```

Behavior when `destinationApi` is `"mutable"`:

```js
import { copyDurationConfig } from '../implement/this/somewhere';

function wrapMomentMethod(func, 'mutable') {
  return function() {
    var newDuration = func.apply(moment.immutable.duration(this), arguments);
    copyDurationConfig(this, newDuration);
    return this;
  };
}
```


## Implementation Notes

A proposal for bootstrapping the new namespaces described above:

1. First, create `moment.immutable` as a wrapped version of the `moment`
   factory function immediately after creating `moment` and `moment.mutable`,
   before registering all the core methods onto the Mutable prototype.
2. Ditto for Durations:  we need hardcoded logic to create
   `moment.immutable.duration` as a wrapped version of the
   `moment.mutable.duration` factory, and we need that code to run as early as
   possible.
3. Create `wrapMomentFactory`, `wrapDurationFactory`, `wrapMomentMethod`, and
   `wrapDurationMethod` functions as static methods of `moment`.  (See below.)
4. Create the 4 prototypes: `moment.mutable.fn`, `moment.mutable.duration.fn`,
   `moment.immutable.fn`, and `moment.immutable.duration.fn`.
5. Add all core functionality to the 4 prototypes, using the `wrap*` methods as
   appropriate.

All plugins will be entirely compatible with the Immutable API proposal if they
correctly attach all of their functionality to both the `moment.mutable` and
`moment.immutable` namespaces.  The `moment.wrap*` static methods will help
most plugin authors do this with relatively low effort.


# How We Teach This

Because our own experience and a preponderance of Moment-related support
requests suggest that new users would have fewer problems working with the
Immutable API, we would generally want to push new users to use
`moment.immutable` exclusively and ignore the traditional Mutable API.

We probably need some detailed release documentation at momentjs.com introducing
existing users to the semantic distinctions between the Mutable and Immutable
APIs, reassuring them that the Mutable APIs haven't been removed (yet) but also
encouraging them to consider adoption of the Immutable APIs if it makes sense
in their situation.

We also need to update our API reference documentation at momentjs.org/docs to
account for all these changes.  It would feel pretty heavy to just append
(largely duplicative) documentation for all the `moment.immutable.*` APIs to
the existing documentation page.  Since we generally want to steer new users
toward the Immutable APIs, then maybe we could show just the Immutable APIs by
default, with a toggle switch to convert between Mutable and Immutable API docs
(similar to how some Coffeescript libraries let you toggle between .js and
.coffee syntax in their documentation's code examples).

Finally, we'll need some targeted documentation explaining how and (especially)
why plugin authors should update their plugins to work with the new version of
Moment.  We might consider submitting PRs to a few popular plugins to help them
adopt the new registration APIs, and then link to those PRs from the
documentation as examples of how other plugin authors can update their
implementations.


# Drawbacks

1. Doubling the number of exposed API methods (for "immutable" and traditional
   versions of everything) will inevitably increase our documentation and
   support burdens, even if almost all of the code is the same.  It's also
   likely to increase the testing burden for new feature additions from
   maintainers and external contributors alike, although hopefully the impact
   on *implementing* new features will be negligible.
2. By giving developers a choice of immutable/mutating APIs, new users of
   Moment will have more friction getting started with the library because
   they'll need to choose an API before they can do anything useful.
3. This feature has often been described as an "immutable" API, but we're not
   actually making the objects immutable — developers can assign and modify
   custom properties on these objects as much as they want.  Indeed,
   "immutable" instances would still be mutating under the hood in certain
   situations (because some internal properties are lazily computed).  This may
   be confusing for some developers and we will likely get questions about it.
4. We expect these changes will increase the library's file size somewhat, even
   for users who don't use any of the new APIs, simply because we're adding
   code to the library.
5. Because plugin methods and factories are created with string names,
   minifiers won't be able to rename those methods, possibly leading to an
   additional slight increase in file sizes for folks who minify their
   dependencies together with their custom code.
6. The new plugin registation system may slightly increase the time required
   for Moment and its plugins to initialize themselves for use.  I don't expect
   this to be a noticeable issue, though.


# Alternatives

As hinted in the Motivation section, if we don't do this (or something
similar), then we're effectively asking the folks who keep +1'ing [#1754][]
to build their own community around the existing third-party plugin, or to find
or build another library instead of using Moment.  This would not be the end of
the world, but it would be a little frustrating since we agree that date-time
objects should ideally be an immutable type.

To date, the current third-party Frozen Moment plugin has had much less reach
than we might expect for a first-party solution.  We might be able to close
that gap somewhat by promoting the third-party plugin more heavily on
momentjs.com, since it doesn't appear to be mentioned anywhere in the official
documentation right now.  This sort of cross-promotion scheme could achieve
some of the same usage benefits while delegating much of the communication and
maintenance burden for this functionality outside the core Moment team — at the
cost of continuing with some additional perceived and actual risk (relative to
a first-party solution) as to whether that plugin would continue to be
maintained appropriately over the long term.

There is no technical reason why the current third-party plugin, or a new
first-party plugin, could not grow to do everything that this RFC proposes.
In fact, I will likely work to make that happen within the Frozen Moment plugin
if this RFC is not adopted.  But any plugin-based solution will be handicapped
by the fact that users don't expect plugins to be able to override core API
decisions like this, so they're less likely to find and adopt a plugin for this
purpose (even if that plugin was well-publicized, etc).

I also hope that it will become easier to get other plugin authors to write
compatible plugins (by using the registration system, etc) if these capabilities
are in core, rather than "just" another third-party plugin.


# Unresolved questions

1. We know we want to strongly encourage users — especially new users — to adopt
   the Immutable APIs as much and as quickly as possible.  But, how quickly and
   strongly do we want to deprecate and remove the traditional Mutable APIs?
2. The documentation and publicity strategy for this release needs to be
   fleshed out further as we continue planning.
3. Do static "configuration" methods get copied into `moment.mutable` and
   `moment.immutable`, or do they only exist on the top-level `moment`?


# Footnotes

1. <a id="fn1"></a> Thus, unlike the existing third-party Frozen Moment plugin,
   `moment.immutable.fn` will **not** have `moment.fn` as its prototype.  This
   means that all Immutable API methods must explicitly appear on the Immutable
   prototype, instead of simply falling through to the Mutable prototype for
   methods that are not expected to mutate the object.  This change is made to
   ensure correctness; this way, Mutable plugin methods that don't support the
   Immutable API will not appear on Immutables so that they cannot accidentally
   mutate the value of a Immutable.  (The same is true for
   `moment.immutable.duration.fn`: it will not have `moment.duration.fn` as its
   prototype for the same reason.) [↩](#a1)


[#1754]: https://github.com/moment/moment/issues/1754
[Frozen Moment]: https://github.com/WhoopInc/frozen-moment/
[maggiepint post]: https://maggiepint.com/2016/06/24/why-moment-js-isnt-immutable-yet/
[pull request]: https://github.com/moment/moment-rfcs/pull/2
[ValueObject]: http://martinfowler.com/bliki/ValueObject.html
