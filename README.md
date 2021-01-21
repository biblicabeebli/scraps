# Biblicabeebli's Code Scraps
A repository containing various code scraps that I have developed over time, the ones that you may find useful.

All content within this repository is public domain and comes with no warranty or guarantees whatsoever.  Bugs probably exist.


## Generate a useful data structure of all timezones including DST
This code generates a dictionary of UTC offsets, including the DST offset, matched to a list of all timezones names that match those offsets exactly. This output is intended for use in on webpages, but can be used for anything that has a need to list timezones in a usefully sorted fashion. All the opbjects with within the output dictionar are strings.

The best way to create a tzinfo object in Python is with `tz.gettz(timezone_name)`.  Note that this particular call apparently has platform-specific dependencies.

The output data is pretty raw, you still have to make an **editorial decision for your use-case**. (Dropdown of everything? Dropdown of a custom subset of names? one item per unique dst-offset, std-offset combo? a typeable field with autofill logic? etc.)

```
from dateutil import tz
import pytz
from datetime import timedelta
from collections import defaultdict


def timedelta_to_label(td: timedelta) -> str:
    """ returns a string like +1:00 """
    label = "-" + str(abs(td)) if td.total_seconds() < 0 else "+" + str(abs(td))
    return label[:-3]


def string_sorter(key: str):
    """ get the first timedelta's floating point representation as the 'key' in our sort algo."""
    return float(key.split("/")[0].replace(":", "."))


def build_dictionary_of_timezones():
    # defaultdicts are cool.
    zones_by_offset = defaultdict(list)

    # there are more timezones in pytz.all_timezones
    for zone_name in pytz.common_timezones:
        # this 'tz_info' variable's type may be dependent on your platform, which is ... just insane.
        # This has been tested and works on Ubuntu and AWS Linux 1.
        tz_info: tz.tzfile = tz.gettz(zone_name)
        utc_offset: timedelta = tz_info._ttinfo_std.delta

        # No DST case
        if tz_info._ttinfo_dst is None:
            label = timedelta_to_label(utc_offset)
        else:
            dst_offset = tz_info._ttinfo_dst.delta
            # fun timezone case: some timezones HAD daylight savings in the past, but not anymore.
            # treat those as not having dst because anything else is madness.
            if dst_offset == utc_offset:
                label = timedelta_to_label(utc_offset)
            else:
                # this ordering yields +4:00/+5:00 ordering in most cases, but there are exceptions?
                # It's not hemispheric, I don't what those places are doing with time.
                label = f"{timedelta_to_label(utc_offset)}/{timedelta_to_label(dst_offset)}"

        zones_by_offset[label].append(zone_name)

    and_finally_sorted = {}
    for offset in sorted(zones_by_offset, key=string_sorter):
        and_finally_sorted[offset] = zones_by_offset[offset]

    return and_finally_sorted


all_zones_by_offset = build_dictionary_of_timezones()

```
