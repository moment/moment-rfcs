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

Each moment that was created would have a time zone. Calls to `moment()` would have a `LocalTimeZone` instance, calls to `moment.utc()` would have a `FixedOffsetTimeZone` instance, and
calls to `moment.tz()` would have a `TZDBTimeZone` instance.

## TimeZone interface

The TimeZone interface consists of 3 methods.

### `TimeZone#offset(timestamp)`

This method is responsible for returning the offset at a particular timestamp. The offset should be in minutes, the same as `moment#utcOffset`.

Sample implementations:

```
FixedOffsetTimeZone.prototype.offset = function (timestamp) {
	return this.presetOffset;
};

LocalTimeZone.prototype.offset = function (timestamp) {
	return -(new Date(timestamp).getTimezoneOffset());
};
```

The `LocalTimeZone` could use a naive implementation like above, or something more sophisticated like http://stackoverflow.com/a/35883858/634824 to work around browser inconsistencies.

We could use something like [moment-timezone's user offsets](https://github.com/moment/moment-timezone/blob/6f663eb510661939e04036c53edb7851212af757/src/guess/user-offsets.js) method to pre-calculate all the offset changes to make lookups faster.

### `TimeZone#parse(parts, options)`

This method is responsible for returning the offset from a group of date parts.

`parts` is an array of date parts from year to minute. `[year, month, day, hour, minute]`.

`options` contains flags for `moveAmbiguousForward` and `moveInvalidForward`.

This can be used in parsing, to determine the correct offset to parse a moment into, or to determine which day to default to for issues like https://github.com/moment/moment-timezone/issues/297.

Sample implementations:

```
FixedOffsetTimeZone.prototype.parse = function (parts, config) {
	return this.presetOffset;
};

LocalTimeZone.prototype.parse = function (parts, config) {
	var parsed = new Date(parts[0], parts[1], parts[2], parts[3], parts[4];
	return -parsed.getTimezoneOffset();
};
```

The `LocalTimeZone` implementation can use something similar to [moment-timezone's implementation](https://github.com/moment/moment-timezone/blob/6f663eb510661939e04036c53edb7851212af757/src/zone/zone.js#L29-L52) using the pre-calculated offsets.

### `TimeZone#abbr(timestamp)`

This method is responsible for returning the time zone abbreviation for a particular timestamp _if available_. This will be used for the `z` and `zz` tokens by the formatter.

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

## Filesize

This will increase the file size of moment, but would likely allow some refactoring and optimization that would offset the increase.

# Alternatives

To solve https://github.com/moment/moment-timezone/issues/297, we could introduce yet another moment-timezone specific hook for determining what 'today' is for a timezone.

# Unresolved questions

Should we bike shed the TimeZone interface method names?
 Could we deprecate public apis created specifically for moment-timezone? `moment.updateOffset` and `moment.momentProperties`.

Could we move `moment.tz.setDefault` into moment core as a `moment.defaultZone` getter/setter?

Could we deprecate `moment#isUTC`, `moment#isUtcOffset` or find a more accurate alternative?
