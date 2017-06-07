# Timezone/Offset handling in moment

## Intro

**NOTE**: In the following document, DST refers to time shifts happening in
different timezones, not necesserily the ones occuring periodically for
"daylight-saving-time".

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
    * jump forward by the size of the gap (default)
    * jump backward by the size of the gap
    * jump forward to first valid time (might not be size of gap)
    * jump backward to first valid time (might not be size of gap)
    * declare the date as invalid

* for ambiguous dates:
    * pick past offset (the one that was present before DST) (default)
    * pick future offset (the one that continues after DST)
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
  ambiguous. Use the default operations for invalid/ambiguous.
* Add/Subtract
    * add/subtract smaller units (hour or less) - these should be *UTC correct*
    * add/subtract bigger units (day or more) - these should be *local correct*
      To put it differently, they should behave like a Set operation.
      The idea is that 1st May 12:34 + 15 days should be 16 May 12:34 (if
      possible), no matter what DST happened from 1st to 15th of May. Use
      default operations for invalid/ambiguous (same as set, create).
* Set one or several units - set is very similar to create (but instead of
  specifying all units, only the different ones are specified). Use default
  operations for invaid/ambiguous moments, resulting from a set.
* StartOf/EndOf
    * statof UNIT - in case the start of UNIT is invalid - jump to first valid
      local time of the UNIT (that might not be the gap), in case its ambiguous
      - pick the past one (that is the default).
    * endof UNIT - this should just use startof UNIT - 1ms (UTC correct
      -1ms).
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

* `correctLocalDefault(lts)` -- return [new\_lts, offset], new\_lts might
  differ from lts if lts is invalid.

    ```javascript
    function correctLocalDefault(lts) {
        var res = []
        var of1 = getOffsetFromUTC(lts) // we treat local timestamp as unix to get
                                        // a ballbpark estimate
        var of2 = getOffsetFromUTC(lts - of1) // adjust local by probable offset
        if (of1 == of2) {
            // (lts, of1) is valid, but could be ambigous (second)
            of3 = getOffsetFromUTC(lts - of1 - 6h); // subtract 6h to see if
                                                    // we're near DST
            if (of1 == of3) {
                return [lts, of1];
            } else if (getOffsetFromUTC(lts - of3) == of3) {
                // ambiguous, variants are [lts, of3], [lts, of1], of3 being
                // the previous
                return [lts, of3];
            } else {
                // there was DST shortly before [lts, of1], but it fully passed
                return [lts, of1];
            }
        } else {
            // we try a second time, this could happen around invalid time
            var of3 = getOffsetFromUTC(lts - of2);
            if (of3 == of2) {
                return [lts, of2]
            } else {
                // invalid time!
                if (of2 > of3) {
                    var tmp = of2; of2 = of3; of3 = tmp;
                }
                var dstGap = of3 - of2;
                if (getOffsetFromUTC(lts + dstGap - of3) == of3) {
                    return [lts + dstGap, of3];
                } else {
                    throw new Error("should never happen (test)");
                }
            }
        }
    }
    ```

* TODO `jumpForwardWholeHours`
* TODO `jumpForwardNearest`
