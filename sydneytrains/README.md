# sydneytrains

An acronym I'll be using quite a lot here is ATRICS (Advanced Train Running Information Control System). ATRICS is a computerised signalling interface being rolled out by Sydney Trains. The boundaries of ATRICS territory are (as of September 2024):
- North: Vales Point (excl)
- West: Newnes Junction (incl)
- South: Macarthur (incl)
- South Coast: Kiama (incl)

> [!NOTE]
> Technically, the `SouthCoast` region (Helensburgh to Port Kembla and Unanderra) is still currently controlled via push-button panel at Wollongong, not ATRICS, however on 2 November 2024 it seems that GTFS-R position reporting for `SouthCoast` was moved to the ATRICS pipeline in preparation for further rollout of ATRICS control in the area. For this reason I'm now considering `SouthCoast` to be an ATRICS region for the purposes of this documentation.

In total, the Sydney-Trains-controlled network extends to Newcastle Interchange/Woodville Junction(North), Lithgow(West), and Bomaderry/Port Kembla (South Coast). Outside ATRICS territory, other systems such as push button panels, lever frames, and older computerised signalling interfaces are used.

## Coverage
### GTFS timetable
The sydneytrains [GTFS timetable - for realtime](https://opendata.transport.nsw.gov.au/dataset/public-transport-timetables-realtime) contains timetable data for all Sydney Trains and NSW TrainLink services, however NSW TrainLink diesel services are often better represented in the nswtrains timetable.

### GTFS-R
There are two sydneytrains GTFS-R `vehiclepos` feeds. TfNSW recommends use of the v2 feed. There's currently no data exclusive to v1 ([though there has been previously](#f-and-w-sets)), whereas v2 has coverage of more area, train type/length updates, and live occupancy(Waratah only).

- [v1](https://opendata.transport.nsw.gov.au/dataset/public-transport-realtime-vehicle-positions): Covers all ATRICS territory, plus the `NIF` region.

- [v2](https://opendata.transport.nsw.gov.au/dataset/public-transport-realtime-vehicle-positions-v2): Covers all ATRICS territory, plus the `NIF` & `MetroNet` regions. This represents all of the Sydney-Trains-controlled network except Kiama–Bomaderry, which is also the only unelectrified region controlled by Sydney Trains.

More details on regions and locations can be found [here](locations/README.md).

## `vehiclepos` GTFS-R feed
The `vehiclepos` feed is generated every 10 seconds, with the last generation time provided in the `header.timestamp` field of the protobuf.

### `trip.trip_id`
The most common form of trip_id, `65-X.1905.101.56.M.8.81177383`, contains various information about the service as described in TfNSW documentation. Timetable information for such services is given in the GTFS timetable or GTFS-R `realtime` feed for ad-hoc or altered services.

Next, there are IDs that start with `NonTimetabled.` followed by a train description/run number. In the majority of cases, these are out-of-service passenger train transfers, freight, or maintenance vehicles. However, sometimes normal passenger services will revert to reporting as NonTimetabled if they are significantly delayed or altered, which can make it annoying to properly correlate them to a timetable entry.

There is also usually a surge in NonTimetabled vehicles overnight when the system rolls over to the next day's timetable and run numbers can no longer be found in the new timetable, and as run numbers are entered at yards for the first trains out in the morning.

Another NonTimetabled case is the form of `NonTimetabled.U001`, where 001 may be any three digits. I don't know what the U is supposed to stand for, but in my mind it's 'undescribed', as these will often appear at the exit from a yard when a train is waiting on the track and the signaller hasn't yet labelled it with its run number.

NonTimetabled vehicles only appear in the feeds when within ATRICS territory, and Mariyung/D set trains in [the `NIF` region](locations/README.md#nif).

#### F and W sets
On 18 November 2023 some freight trains, which used to exclusively report as NonTimetabled, changed to reporting with the standard passenger `trip_id` format, using the set types F or W, 0 carriages, ~and in the v1 feed only~. W is defined as "Fast freight" by Sydney Trains, but F is not defined. Sydney Trains do define G for "Freight", however I've never seen it used.

I haven't been able to find a pattern to the use of F or W, other than W being much more common. A given train will generally consistently appear as F, W or NonTimetabled day-to-day, and it seems that most trains that run to a consistent timetable appear as F or W, whereas irregular freight services generally remain as NonTimetabled.

~I was interested as to whether this change might mean that freight services would appear in the GTFS timetable or be visible outside ATRICS territory, however neither of these have eventuated.~

As of 27 April 2024, F and W sets are included in the v2 feed and are visible outside the ATRICS area, in the `MetroNet` region. So far, it seems that a single train may report with two slightly different trip_ids, one when within the ATRICS area, and another outside, so through-tracking will probably be easiest just by run number.

#### Incorrect set types
Often during disruptions, added or altered services will be labelled with the incorrect set type, most commonly A (which I assume is just the default being first alphabetically), but I've also seen X, [P](https://twitter.com/Tugzrida/status/1499549896474464259), and others.

Somewhat relatedly, in March 2024 I began to see 86 class electric maintenance trains (run numbers `Exxx`) reporting as 0 car H sets. Previously they'd reported as [W sets](#f-and-w-sets) or NonTimetabled (which continues). These H set runs don't appear in the GTFS timetable. I've not been tracking whether this is happening to any other trains, maintenance or otherwise.

### `position` / `stop_id`
None of the positions reported in the sydneytrains `vehiclepos` feed are directly sourced from onboard GPS. The `stop_id` field contains an ID consisting of a dot-separated region name and location, the lat/long of which are included in the `position` fields. Further detail is documented [here](locations/README.md).

### `timestamp`
The Unix timestamp when the vehicle position last changed.

There are numerous locations around the network at which the timestamp never seems to update. There is honestly an endless well of edge cases here but a strategy similar to this should keep the vehicle's timestamp close to the correct "last moved" time:

```python
# init
prevLocations = {}
prevTimestamps = {}

# inside fetch loop
for entity in vehiclepos.entity:
    if entity.vehicle.stop_id != prevLocations.get(entity.vehicle.trip.trip_id):
        # vehicle moved
        if entity.vehicle.timestamp == prevTimestamps.get(entity.vehicle.trip.trip_id):
            # vehicle moved but timestamp not updated, use header timestamp
            entity.vehicle.timestamp = vehiclepos.header.timestamp

        # save location and potentially corrected timestamp for next loop
        prevLocations[entity.vehicle.trip.trip_id] = entity.vehicle.stop_id
        prevTimestamps[entity.vehicle.trip.trip_id] = entity.vehicle.timestamp
    else:
        # vehicle not moved, restore timestamp in case it was previously corrected
        entity.vehicle.timestamp = prevTimestamps[entity.vehicle.trip.trip_id]

# prune prevLocations and prevTimestamps of trip_ids no longer in the feed
```

The timestamp is also subject to abnormality at DST transitions, documented [here](DaylightSaving.md#gtfs-r-vehiclepos).

Diesel passenger & freight trains in the `MetroNet` region sometimes report a timestamp 10 or 11 hours in the past, which I assume means that they report a timestamp to the MetroNet system in UTC whereas the electric trains report in local time, and a conversion to UTC somewhere along the line fixes the timestamp for electric trains but breaks it for diesels.

### `vehicle.id`
TfNSW claims 'these are carriage numbers which have been masked' and that seems to be either correct, or they're just completely random numbers. I've not been able to determine any correlation to real carriage numbers, and the numbers in this field change with every trip, even with the same physical vehicle. To some degree, this makes sense as the signalling systems which source the location data for this feed have no idea what carriages make up a train. As far as I know, train type/length swaps must be entered manually, and so set and carriage numbers are unlikely to be added.
