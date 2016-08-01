- Start Date: 2016-07-07
- RFC PR: [#2][pull request]
- Moment Issue: [#1754][]


# Summary

Change all APIs so that all moments will be psuedo-immutable in Moment 3.


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
the years to date.

There is a plugin, [Frozen Moment][], that has attempted to address these
issues.  Unfortunately, as a third-party plugin it has limited visibility and
less certainty about long-term support, so it's not a satisfactory solution
for everyone wanting an "immutable" API.  More significantly, there isn't a
great way for a plugin to find out about the other Moment plugins that might
be in use in any particular environment, which makes it difficult to use
plugins while also using the immutable APIs.

We've talked with a few authors of widely-used plugins about these challenges,
and heard that some plugin maintainers would prefer to support exactly one API
(mutable or immutable) per version of their plugin -- in other words, if we're
going to add an immutable API, they would prefer a hard cutover where the
immutable API replaces the old mutable API rather than a solution where both
APIs might co-exist simultaneously.

**To address these concerns,** we'd like to change Moment's public APIs to
provide an immutable API (similar to the `moment.frozen` functionality from
Frozen Moment).  We'd also like to work with plugin authors to help them
migrate to the new API.

We might also extend our build system to generate a 2.x-compatible version of
Moment from the same codebase (by changing a few files from one build to the
other) and use that functionality to support both APIs during a transitional
period.  This might most productively be expressed as a 3-6 month period of 2.x
releases and simultaneous 3.0 "beta"/"preview" releases, leading up to a final
3.0 release and/or discontinuation of the existing 2.x release stream.  During
this period, and possibly for some time thereafter, we'd want new releases of
the Moment Timezone plugin to maintain compatibility with both Moment 2.x and 3.x.

Users who prefer the current mutable API can stick with the very stable 2.x
codebase, or they might build a plugin to implement a mutable API on top of
our own immutable API (like Frozen Moment, but moving the opposite direction).


# Detailed design

## For Moment Users

All externally-facing APIs that currently modify the value of a Moment instance
will instead return a new Moment.  This includes many of the instance methods
listed in the "Get + Set", "Manipulate", and "i18n" categories of our [docs][].

All externally-facing APIs that currently modify the value of a Duration
instance will instead return a new Duration.  (I believe this is just
`Duration#add` and `Duration#subtract`.)

Set-selection functions (`moment.max` and `moment.min`) will return one of the
original input instances, **not** a copy -- unless one of the inputs was invalid,
in which case we'll return a canonical "invalid" moment that doesn't include
any interesting information in `_i`.

## For Moment Developers

I think we'll want to take a Frozen Moment-style approach to the implementation.
If a method is going to mutate the instance then internally clone it before doing
anything else, and then use the old mutable implementations of everything.  When
calling other functions internally, we should continue to call the internal
mutable version of the API rather than the externally-facing immutable APIs.
This approach will minimize the effort needed to develop v3.0 and minimize the
performance hit from moving our users onto an immutable API.  Because this will
be a relatively small change to our codebase, this approach is also very useful
if/when we want to build parallel v2.x/v3.0 releases while our users transition
to the new APIs.

We'll want to update and add unit tests asserting the immutable nature of our
external APIs, etc.  I suspect this will require more effort than updating the
core library code itself.  I think we want to avoid maintaining two sets of tests,
even temporarily -- avoiding additional pain and effort in testing is probably
the best argument against doing parallel 2.x/3.x builds for a transition period.


# How We Teach This

Because our own experience and a preponderance of Moment-related support
requests suggest that new users would have fewer problems working with an
immutable API, we will generally want to push new users to adopt 3.x ASAP.
To that end, we'll want to carefully craft our 3.x release notes to help
introduce existing users to the changes and explain what they'll need to do
in order to prepare their codebases for the upgrade.

We also need to update our API reference documentation at momentjs.org/docs to
account for all these changes.  We should maybe consider publishing an archived
copy of the v2.x docs at a new URL when we update the "standard" docs to the 3.x
API for the new release.  On the other hand, the doc changes should be fairly
minor, so maybe that's not a big deal (especially if the docs are clearly and
pervasively marked as describing v3.x).  I think we mostly just want to avoid
having 2.x users rely on the 3.x docs and therefore create mutability bugs.

Finally, we'll need some targeted documentation explaining how and (especially)
why plugin authors should update their plugins to work with the new version of
Moment.  We might consider submitting PRs to a few popular plugins to help them
upgrade, and then link to those PRs from the documentation as examples of how
other plugin authors can update their implementations.


# Drawbacks

1. Some large existing codebases may find it difficult to upgrade from 2.x to 3.x,
   especially if they heavily relied on Moment's mutability.
2. This feature has often been described as an "immutable" API, but we're not
   actually making the objects immutable — developers can assign and modify
   custom properties on these objects as much as they want.  Indeed,
   "immutable" instances would still be mutating under the hood in certain
   situations (because some internal properties are lazily computed).  This may
   be confusing for some developers and we will likely get questions about it.
3. This will likely add a few bytes in file size to the core library, because
   the implementation approach is purely additive.  Similarly, "mutation" methods
   will likely take slightly longer to run, due to the need to allocate memory
   and copy values into a new object.  We may want to do performance testing to
   determine the exact level of impact here, but the effects should be very
   minimal and we don't expect many users to notice.  In fact, users who always
   call `#clone` before mutating their moments may even see a slight improvement
   in overall code size and performance with the new approach, since they can now
   remove a lot of extraneous method calls.


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

1. Should we remove `moment#clone` in 3.0?  If not, I feel it should probably
   continue to return a new instance to minimize confusion -- unless we're keeping
   it solely to ease the pain of transition for existing codebases, in which case
   we should return the same instance and spew deprecation warnings for a planned
   removal in v4.0.
2. Do we want to publish parallel 2.x/3.x builds for a while to ease the transition?
   If so, how do we label this, and how do we justify the timing for shutting off
   the 2.x compatibility builds?
3. The documentation and publicity strategy for this release needs to be fleshed
   out further as we continue planning.


[#1754]: https://github.com/moment/moment/issues/1754
[Frozen Moment]: https://github.com/WhoopInc/frozen-moment/
[maggiepint post]: https://maggiepint.com/2016/06/24/why-moment-js-isnt-immutable-yet/
[pull request]: https://github.com/moment/moment-rfcs/pull/2
[ValueObject]: http://martinfowler.com/bliki/ValueObject.html
[docs]: http://momentjs.com/docs/
