# Timezone/Offset handling in moment

## Intro

Datetime handling is relatively straight forward for UTC, but can get messy
around DST (Daylight Savings Time), when the local clock shifts forwards or
backwards one hour. During DST it's possible that a particular local time is
**invalid** (in the middle of a forward shift) or **ambiguous** (in the middle
of a backwards shift).

In both *invalid* and *ambiguous* cases there is a future and a past offset.
Past offset is before DST and future offset is after DST.

This document describes how various mutating operations on moment deal with
invalid/ambiguous times.

**UTC correct** in this document describes the notion that an operation is
correct in terms of absolute time passing, not taking into account DST.

**Local correct** in this document describes the notion that an operation is
correct in terms of observed local time, taking into account DST.

    # DST 01:59 -> 03:00 (time shifts forward 1 hour at 02:00)
    01:30 + 2 hour -> 04:30 (utc correct)
    01:30 + 2 hours -> 03:30 (local correct)

## How could we handle invalid/ambiguous dates

Of course local correct operations sometimes will hit an invalid or ambiguous
local date. There are a few available options for each case.

* for invalid dates:
    * jump forward by one (or more) hours until the time is valid
    * jump backward by one (or more) hours until the time is valid
    * jump forward to first valid time (might not be whole hours)
    * jump backward to first valid time (might not be whole hours)
    * declare the date as invalid

* for ambiguous dates:
    * pick future offset (the one that continues after DST)
    * pick past offset (the one that was present before DST)
    * pick current offset (in case this operation changes from old to new date,
      the old date's offset could be used, if its one of the two alternatives
      -- so this *option* is only best effort and needs to be backed up by
      another one)
    * declare the date as invalid (in case an explicit offset hasn't been
      specified)

This document will describe what options will be taken by default in moment
operations. All these choices could be configurable.

## Operations

* Create from local time - given year, month, day, hour, min, sec in local
  time, construct a moment. Note that the given time might be invalid or
  ambiguous. Invalid - jump forward by whole hours to first valid local time.
  Ambiguous - pick the offset that continues in the future.
* Add/Subtract
    * add/subtract smaller units (hour or less) - these should be *UTC correct*
    * add/subtract bigger units (day or more) - these should be *local correct*
      The idea is that 1st May 12:34 + 15 days should be 16 May 12:34 (if
      possible), no matter what DST happened from 1st to 15th of May. If the
      time is invalid, jump forward (by whole hour) to valid; if ambiguous pick
      future. (For subtract jump backwards and pick past).
* Set one or several units - set is very similar to create (but instead of
  specifying all units, only the different ones are specified).
    * set smaller units (minute or less) - these should be *UTC correct*. That
      is, the set acts as an add/subtract depending on the current value. So if
      the minute is now 15, a set to 45 means add 30 minutes (check
      add/subtract above).
    * set bigger units (hour or more) - these should be *local correct*. So,
      again it acts as add/subtract depending on current value, and tries to be
      local correct.
* StartOf/EndOf
    * statof 'day' - in case the start of day is invalid - jump to first valid
      local time of today (not whole hours), in case its ambiguous - pick the
      future one
    * endof 'day' - same idea as start of but in reverse :)
* change timezone while keeping the local time - same idea applies as creating
  local time from components

## Zone implementation

The most basic zone implementation should return an offset from a UTC timestamp
(unix time). Lets call this `getOffsetFromUTC(ts)`.

**NOTE**: Do not confuse this with Local timestamp to offset -- as mentioned
in the Intro section, some local times are invalid, others are ambiguous, so
a proper function returning offset from local timestamp should be able to
return 0 1 or 2 offsets (an array), to be complete for all inputs.

`shiftAtUTC(ts)` : what is the DST shift at this particular UTC timestamp.  --
would return `null` if there is no DST at that ts, or a number specifying minutes
shifted (positive means jump ahead in local time, negative means jump behind).

`shiftAtUTC(ts) = getOffsetFromUTC(ts) - getOffsetFromUTC(ts-1)`

`nextShift(ts) : ts` : when does the next DST shift happen, after ts. If ts
is a DST shift itself, we could return the input.

`prevShift(ts) : ts` : when does the previous DST shift happen, before ts.

**NOTE**: nextShift and prevShift are not trivially computable from
getOffsetFromUTC but using some heruistics it could be made to work almost
universally (until some country decides to go nuts with its timezone).

## Operations implementation

I'll use `lts` to specify local unix timestamp (this is ts+offset, its not
universally recognized way of keeping local time, but I'll still use it here
because its one number, instead of 2 or more). Note that it could be invalid or
ambiguous.

* `localToOffsets(lts)` -- we'll define a function that returns all offsets of
  a given local timestamp.

    ```javascript
    function localToOffsets(lts) {
        var res = []
        var of1 = getOffsetFromUTC(lts) // we treat local timestamp as unix to get
                                        // a ballbpark estimate
        var of2 = getOffsetFromUTC(lts - of1) // adjust local by probable offset
        if (of1 == of2) {
            // (lts, of1) is valid
            res.push(of1)
        } else {
            var of3 = getOffsetFromUTC(lts - of2)
            if (of3 == of2) {
                // (lts, of2) is valid
                res.push(of2)
            } else {
                // lts is invalid
            }
        }

        if (res.length == 0) {
            // invalid
            return res;
        } else {
            var of = res[0];
            // NOTE: This is heruistic, so might not work in all cases.
            // check the offsets of lts-of+1h and lts-of-1h
            var of4 = getOffsetFromUTC(lts-of-1h)
            var of5 = getOffsetFromUTC(lts-of+1h)

            if (of4 == of+1h) {
                res.push(of4)
            } else if (of5 == of-1h) {
                res.push(of5)
            }
            return res;
        }
    }
    ```

* TODO `jumpForwardWholeHours`
* TODO `jumpForwardNearest`
