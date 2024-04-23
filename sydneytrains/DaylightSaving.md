## Daylight Saving Time
### GTFS timetable
Timestamps in general are covered in a somewhat novel way in the timetable: times may extend beyond 23:59 to indicate services running into the next day. There is one additional quirk with regards to DST, which is documented by TfNSW but took me a while to figure out: timestamps are counted since "midday minus 12 hours", which most of the time is midnight, and so matches local time. However on the day where DST ends, there are 13 hours between midnight and noon and so timestamps are as follows:

| Timestamp(previous day) | Timestamp(transition day) | AEDT  | AEST  |
|-------------------------|---------------------------|-------|-------|
| 23:00                   |                           | 23:00 |       |
| 24:00                   |                           | 00:00 |       |
| 25:00                   | 00:00                     | 01:00 |       |
| 26:00                   | 01:00                     | 02:00 |       |
| 27:00                   | 02:00                     |       | 02:00 |
| 28:00                   | 03:00                     |       | 03:00 |

When DST starts, there are only 11 hours between midnight and noon, and so:

| Timestamp(previous day) | Timestamp(transition day) | AEST  | AEDT  |
|-------------------------|---------------------------|-------|-------|
| 23:00                   | 00:00                     | 23:00 |       |
| 24:00                   | 01:00                     | 00:00 |       |
| 25:00                   | 02:00                     | 01:00 |       |
| 26:00                   | 03:00                     |       | 03:00 |
| 27:00                   | 04:00                     |       | 04:00 |
| 28:00                   | 05:00                     |       | 05:00 |


### GTFS-R vehiclepos
Timestamps in the realtime feed are Unix timestamps, however at the end of DST the feed usually freezes for the duration of both 2am's. When the feed returns, vehicles often have strange timestamps up to an hour in the future(`MetroNet` territory is especially prone to this), and their movement during the outage may also report in quick succession. Some of these quirks seem to require a manual reset from Sydney Trains to resolve, which is usually done within a few hours after the transition.

The overall feed timestamp appears to be unaffected, and at the start of DST there is also no abnormality I've observed.
