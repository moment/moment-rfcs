# Summary

Add a `TimeZone` interface to moment core to normalize local, utc, utcOffset, and moment-timezone modes.

# Motivation

We currently have 4 different apis for dealing with offsets and timezones. This is problematic for a few reasons.

1. It is difficult to describe the differences to end users. We refer to this in docs as 'UTC mode', 'UTC time', 'local time', 'fixed UTC offset', and 'time zone'.

2. It is difficult to deal with DST changes. Many test failures are the result of DST issues in specific browsers and specific timezones.

3. We are unable to control how browsers handle ambiguous or invalid wall-clock times due to DST in local mode. See http://stackoverflow.com/a/35883858/634824

4. We hack around the moment constructor in moment-timezone in a way that makes it impossible to parse time only input correctly. https://github.com/moment/moment-timezone/issues/297

5. We have forked logic for local and utc mode getters and setters. `Date#getMonth` vs `Date#getUTCMonth`.

# Detailed design

To solve this, we could introduce an interface for a `TimeZone` in moment. Moment would provide 2 classes that implement the `TimeZone` interface, `LocalTimeZone` and `FixedOffsetTimeZone`. Moment timezone would provide a `TZDBTimeZone` class that implements the `TimeZone` interface.

Each moment that is created would have a time zone. Calls to `moment()` would have a `LocalTimeZone` instance, calls to `moment.utc()` would have a `FixedOffsetTimeZone` instance, and
calls to `moment.tz()` would have a `TZDBTimeZone` instance.

## TimeZone interface

The TimeZone interface consists of these methods and properties.

```js
TimeZone#abbrFromTimestamp(timestamp)
TimeZone#abbrFromParts(year, month, day, hour, minute, second, millisecond)
TimeZone#offsetFromTimestamp(timestamp)
TimeZone#offsetFromParts(year, month, day, hour, minute, second, millisecond)
TimeZone#toString()
TimeZone#type
```

Only one of `offsetFromTimestamp` or `offsetFromParts` must be implemented. Moment will try `offsetFromTimestamp` first, and if it is not implemented, will use `offsetFromParts`.

Only one of `abbrFromTimestamp` or `abbrFromParts` must be implemented. Moment will try `abbrFromTimestamp` first, and if it is not implemented, will use `abbrFromParts`.

### `TimeZone#offsetFromTimestamp(timestamp)`

This method is responsible for returning the offset at a particular timestamp. The offset should be in minutes, the same as `moment#utcOffset`.

This timestamp is not the number of milliseconds since 1970-01-01T00:00+00:00. Instead, it is the number of milliseconds since 1970-01-01T00:00 in that time zone.

The same calendar date will use the same timestamp regardless of its offset. See below, where changing the offset does not change the timestamp, but changing the date does.

```
Date        Offset  Timestamp
2016-01-01  +00:00  1451606400000
2016-01-01  +01:00  1451606400000
2016-01-01  +12:00  1451606400000
2016-01-02  +12:00  1451692800000
2016-01-02  +00:00  1451692800000
```

Sample implementation:

```js
FixedOffsetTimeZone.prototype.offsetFromTimestamp = function (timestamp) {
	return this.presetOffset;
};
```

This only needs to be implemented if `offsetFromParts` is not implemented.

### `TimeZone#offsetFromParts(year, month, day, hour, minute, second, millisecond)`

This method is responsible for returning the offset from a group of date parts.  

Sample implementation:

```js
LocalTimeZone.prototype.parse = function (year, month, day, hour, minute, second, millisecond) {
	var parsed = new Date(year, month, day, hour, minute, second, millisecond);
	return -parsed.getTimezoneOffset();
};
```

This only needs to be implemented if `offsetFromTimestamp` is not implemented.

### `TimeZone#abbrFromTimestamp(timestamp)`

This method is responsible for returning the time zone abbreviation for a particular timestamp _if available_. This will be used for the `z` and `zz` tokens by the formatter. This only needs to be implemented if `abbrFromParts` is not implemented.

Like `offsetFromTimestamp`, this timestamp is the number of milliseconds since 1970-01-01T00:00 in that time zone.

### `TimeZone#abbrFromParts(year, month, day, hour, minute, second, millisecond)`

This method is responsible for returning the time zone abbreviation from a group of date parts _if available_. This only needs to be implemented if `abbrFromTimestamp` is not implemented.

### `TimeZone#toString()`

This returns a uniquely identifying string used to reference the time zone. Some examples are `"local"` for local time zones, `"UTC"` or `"UTC+01:00"` for fixed offset time zones, and `"America/Chicago"` or `"America/Los_Angeles"` for tzdb time zones.

### `TimeZone#type`

This is a string representing the type of time zone. It will be one of `local`, `fixed-offset`, `tzdb` (subject to bikeshedding).

## New Moment APIs

In addition to these interfaces, we will need to add some new apis to manage working with time zones.

### `moment.defineTimeZone(callback)`

This allows other parties to register callbacks used to resolve TimeZone objects from a string identifier.

Sample implementation:

```js
// local
moment.defineTimezone(function (input) {
  if (input === 'local') {
    return new LocalTimeZone();
  }
});
// fixed offset
moment.defineTimezone(function (input) {
  if (matchesFixedOffsetFormat(input)) {
    return new FixedOffsetTimeZone(input);
  }
});
// moment timezone
moment.defineTimezone(function (input) {
  if (zones[input]) {
    return zones[input];
  }
});
```

The timezone resolver would just loop through all the callbacks, returning the first timezone that matched. If there were no matches, a `console.error` would be logged.

### `moment.withTimeZone`

This api allows users or libraries to construct a moment in a specific timezone. It returns a function with the same signature as `moment()` and `moment.utc()`.

Usage:

```js
var create = moment.withTimeZone('America/Chicago');
var nowInChicago = create();
var decemberInChicago = create("2016 décembre", "YYYY MMMM", "fr");
```

### `moment#timeZone`

This api allows users to get or set the time zone. The value passed in is a string identifier, and the return value is `TimeZone#toString()`.

Usage:

```js
var instance = moment();
instance.timeZone('America/Los_Angeles');
instance.timeZone();  // America/Los_Angeles
instance.format('Z'); // -07:00
instance.timeZone('local');
instance.timeZone();  // local
instance.format('Z'); // -05:00
```

# How We Teach This

We add a section to the guides introducing the concept of a time zone. It explains the role of a time zone, which is to map timestamps to offsets.

We explain that all moments are created with a time zone. Sometimes it is the local time zone, sometimes it is a fixed offset time zone, and sometimes it is a tzdb aware timezone.

We remove any documentation about 'UTC mode', 'UTC time', and 'local time'.

We change the documentation for `moment()`, `moment.unix()`, and `moment#local()` to explain that these are setting the time zone to a local time zone.

We change the documentation for `moment.utc()`, `moment#utc()`, `moment#parseZone`, and `moment#utcOffset()` to explain that these are setting the time zone to a fixed offset time zone.

We add documentation for `moment.tz()` and `moment#tz` to explain that these are setting the time zone to a tzdb time zone.

# Drawbacks

## Cleanup and Refactoring

All of the internals of moment will need to be cleaned up to remove calls to methods like `Date#getMonth` and `new Date(year, month, day)`.

# Alternatives

To solve https://github.com/moment/moment-timezone/issues/297, we could introduce yet another moment-timezone specific hook for determining what 'today' is for a timezone.

Instead of introducing `moment.withTimeZone`, we could add the timezone to the end of the argument list. `moment(input, format, locale, strict, timeZone)`. This method already has a lot of tricky positional arguments, so it doesn't seem reasonable.

Instead of only using partial application for the time zone with `moment.withTimeZone`, we could expand it to format, locale, and strict parsing.

```js
var create = moment.withDefaults({
  zone: "America/Chicago",
  locale: "fr",
  format: "YYYY MMMM",
  strict: true
});
var novemberInChicago = create("2016 novembre");
var decemberInChicago = create("2016 décembre");
```

# Unresolved questions

Could we remove undocumented public apis created specifically for moment-timezone? `moment.updateOffset` and `moment.momentProperties`.

Could we deprecate `moment#isUTC`, `moment#isLocal`, `moment#isUtcOffset` or find a more accurate alternative?
