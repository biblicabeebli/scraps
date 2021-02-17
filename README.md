# Biblicabeebli's Code Scraps
A repository containing various code scraps that I have developed over time, the ones that you may find useful.

All content within this repository is public domain and comes with no warranty or guarantees whatsoever.  Bugs probably exist.

## An elegant timeout cache decorator that I put together.
Gist at https://gist.github.com/biblicabeebli/5cc40b4ded7edc03d07cb87336efb4b6

```
from functools import wraps
from time import perf_counter  # perf_counter is a highly quality timestamp

# Keys are the functions themselves, values are a tuple containing the timeout expiry
# of the cache entry, and the cached return value.
function_cache = {}


def timeout_cache(seconds: int or float):
    def check_timeout_cache(wrapped_func):

        @wraps(wrapped_func)
        def cache_or_call(*args, **kwargs):
            # default (performant) case: look for cached function, check timeout.
            try:
                timeout, ret_val = function_cache[wrapped_func]
                if perf_counter() < timeout:
                    return ret_val
            except KeyError:
                pass
            # slow case, cache miss: run function, cache the output, return output.
            ret_val = wrapped_func(*args, **kwargs)
            function_cache[wrapped_func] = (perf_counter() + seconds, ret_val)
            return ret_val

        return cache_or_call
    return check_timeout_cache


# example:
@timeout_cache(10)
def now_kinda():
    from datetime import datetime
    return datetime.now()
```

## Generate a Useful Data Structure of All Timezones While Including Daylight Savings Time Information
This code generates a dictionary of UTC offsets, including the DST offset, matched to a list of all timezones names that match these offsets exactly, sorted by the offset value and then by name of the time zone.

Output is viable for general use, but was initially intended for a dropdown menu on a website.  All the objects in the output dictionary are strings, this can be changed with some minor updates if you need the objects.

This code is effectively static, it has no state, so you only need to run it at program load time.

The best way to create a tzinfo-like object (for our purposes) is with the Python Dateutil library's `tz.gettz(timezone_name)` function.  These objects contain information to identify DST hour ranges.  (Note: we aren't introspecting for dates-of-onset for DST values.)

Remember that you still have to make an **editorial decision for your use-case** after running this code. (Dropdown of everything? Dropdown of a custom subset of names? one item per unique dst-offset, std-offset combo? a typeable field with autofill logic? etc.)

Update: my use-case turned into a comprehensive dropdown menu off of a list of 2-tuples.  I've added a function to flatten the structure for this use case, and a ledgible version of the generated data structure for easy reference.

Update again: It was brought to my attention that my original code provided deprecated timezones via pytz.  It is slightly non-trivial exclude those, so I've updated the script.

Gist at https://gist.github.com/biblicabeebli/5cc40b4ded7edc03d07cb87336efb4b6

```
from collections import defaultdict
from datetime import timedelta

import pytz
from dateutil import tz


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

    # (at time of development) this provides all non-deprecated timezones.
    tzs = []
    for _, timezones_by_country in pytz.country_timezones.items():
        tzs.extend(timezones_by_country)
    tzs.sort()

    # pytz.common_timezones includes some deprecated tzs, pytz.all_timezones has even more.
    for zone_name in tzs:
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



def flatten_time_zones(all_zones_by_offset):
    """ Builds a dropdown-friendly list of tuples for populating a dropdown. """
    ret = []
    for offset_numbers, locations in all_zones_by_offset.items():
        for location_names in locations:
            ret.append([location_names, offset_numbers + " - " + location_names])
    return ret

all_zones_by_offset = build_dictionary_of_timezones()
COMMON_TIMEZONES_DROPDOWN = flatten_time_zones(all_zones_by_offset)
```

COMMON_TIMEZONES_DROPDOWN looks like this (intentionally built with lists so that this is json-compatible/pasteable):
```
[["Pacific/Midway", "-11:00 - Pacific/Midway"],
 ["Pacific/Niue", "-11:00 - Pacific/Niue"],
 ["Pacific/Pago_Pago", "-11:00 - Pacific/Pago_Pago"],
 ["America/Adak", "-10:00/-9:00 - America/Adak"],
 ["Pacific/Honolulu", "-10:00/-9:30 - Pacific/Honolulu"],
 ["Pacific/Rarotonga", "-10:00/-9:30 - Pacific/Rarotonga"],
 ["Pacific/Tahiti", "-10:00 - Pacific/Tahiti"],
 ["Pacific/Marquesas", "-9:30 - Pacific/Marquesas"],
 ["America/Anchorage", "-9:00/-8:00 - America/Anchorage"],
 ["America/Juneau", "-9:00/-8:00 - America/Juneau"],
 ["America/Metlakatla", "-9:00/-8:00 - America/Metlakatla"],
 ["America/Nome", "-9:00/-8:00 - America/Nome"],
 ["America/Sitka", "-9:00/-8:00 - America/Sitka"],
 ["America/Yakutat", "-9:00/-8:00 - America/Yakutat"],
 ["Pacific/Gambier", "-9:00 - Pacific/Gambier"],
 ["America/Los_Angeles", "-8:00/-7:00 - America/Los_Angeles"],
 ["America/Tijuana", "-8:00/-7:00 - America/Tijuana"],
 ["America/Vancouver", "-8:00/-7:00 - America/Vancouver"],
 ["Pacific/Pitcairn", "-8:00 - Pacific/Pitcairn"],
 ["America/Boise", "-7:00/-6:00 - America/Boise"],
 ["America/Cambridge_Bay", "-7:00/-6:00 - America/Cambridge_Bay"],
 ["America/Chihuahua", "-7:00/-6:00 - America/Chihuahua"],
 ["America/Denver", "-7:00/-6:00 - America/Denver"],
 ["America/Edmonton", "-7:00/-6:00 - America/Edmonton"],
 ["America/Hermosillo", "-7:00/-6:00 - America/Hermosillo"],
 ["America/Inuvik", "-7:00/-6:00 - America/Inuvik"],
 ["America/Mazatlan", "-7:00/-6:00 - America/Mazatlan"],
 ["America/Ojinaga", "-7:00/-6:00 - America/Ojinaga"],
 ["America/Phoenix", "-7:00/-6:00 - America/Phoenix"],
 ["America/Yellowknife", "-7:00/-6:00 - America/Yellowknife"],
 ["America/Creston", "-7:00 - America/Creston"],
 ["America/Dawson", "-7:00 - America/Dawson"],
 ["America/Dawson_Creek", "-7:00 - America/Dawson_Creek"],
 ["America/Fort_Nelson", "-7:00 - America/Fort_Nelson"],
 ["America/Whitehorse", "-7:00 - America/Whitehorse"],
 ["America/Bahia_Banderas", "-6:00/-5:00 - America/Bahia_Banderas"],
 ["America/Belize", "-6:00/-5:00 - America/Belize"],
 ["America/Chicago", "-6:00/-5:00 - America/Chicago"],
 ["America/Costa_Rica", "-6:00/-5:00 - America/Costa_Rica"],
 ["America/El_Salvador", "-6:00/-5:00 - America/El_Salvador"],
 ["America/Guatemala", "-6:00/-5:00 - America/Guatemala"],
 ["America/Indiana/Knox", "-6:00/-5:00 - America/Indiana/Knox"],
 ["America/Indiana/Tell_City", "-6:00/-5:00 - America/Indiana/Tell_City"],
 ["America/Managua", "-6:00/-5:00 - America/Managua"],
 ["America/Matamoros", "-6:00/-5:00 - America/Matamoros"],
 ["America/Menominee", "-6:00/-5:00 - America/Menominee"],
 ["America/Merida", "-6:00/-5:00 - America/Merida"],
 ["America/Mexico_City", "-6:00/-5:00 - America/Mexico_City"],
 ["America/Monterrey", "-6:00/-5:00 - America/Monterrey"],
 ["America/North_Dakota/Beulah", "-6:00/-5:00 - America/North_Dakota/Beulah"],
 ["America/North_Dakota/Center", "-6:00/-5:00 - America/North_Dakota/Center"],
 ["America/North_Dakota/New_Salem", "-6:00/-5:00 - America/North_Dakota/New_Salem"],
 ["America/Rainy_River", "-6:00/-5:00 - America/Rainy_River"],
 ["America/Rankin_Inlet", "-6:00/-5:00 - America/Rankin_Inlet"],
 ["America/Resolute", "-6:00/-5:00 - America/Resolute"],
 ["America/Tegucigalpa", "-6:00/-5:00 - America/Tegucigalpa"],
 ["America/Winnipeg", "-6:00/-5:00 - America/Winnipeg"],
 ["Pacific/Easter", "-6:00/-5:00 - Pacific/Easter"],
 ["Pacific/Galapagos", "-6:00/-5:00 - Pacific/Galapagos"],
 ["America/Regina", "-6:00 - America/Regina"],
 ["America/Swift_Current", "-6:00 - America/Swift_Current"],
 ["America/Atikokan", "-5:00 - America/Atikokan"],
 ["America/Cancun", "-5:00 - America/Cancun"],
 ["America/Cayman", "-5:00 - America/Cayman"],
 ["America/Panama", "-5:00 - America/Panama"],
 ["America/Bogota", "-5:00/-4:00 - America/Bogota"],
 ["America/Detroit", "-5:00/-4:00 - America/Detroit"],
 ["America/Eirunepe", "-5:00/-4:00 - America/Eirunepe"],
 ["America/Grand_Turk", "-5:00/-4:00 - America/Grand_Turk"],
 ["America/Guayaquil", "-5:00/-4:00 - America/Guayaquil"],
 ["America/Havana", "-5:00/-4:00 - America/Havana"],
 ["America/Indiana/Indianapolis", "-5:00/-4:00 - America/Indiana/Indianapolis"],
 ["America/Indiana/Marengo", "-5:00/-4:00 - America/Indiana/Marengo"],
 ["America/Indiana/Petersburg", "-5:00/-4:00 - America/Indiana/Petersburg"],
 ["America/Indiana/Vevay", "-5:00/-4:00 - America/Indiana/Vevay"],
 ["America/Indiana/Vincennes", "-5:00/-4:00 - America/Indiana/Vincennes"],
 ["America/Indiana/Winamac", "-5:00/-4:00 - America/Indiana/Winamac"],
 ["America/Iqaluit", "-5:00/-4:00 - America/Iqaluit"],
 ["America/Jamaica", "-5:00/-4:00 - America/Jamaica"],
 ["America/Kentucky/Louisville", "-5:00/-4:00 - America/Kentucky/Louisville"],
 ["America/Kentucky/Monticello", "-5:00/-4:00 - America/Kentucky/Monticello"],
 ["America/Lima", "-5:00/-4:00 - America/Lima"],
 ["America/Nassau", "-5:00/-4:00 - America/Nassau"],
 ["America/New_York", "-5:00/-4:00 - America/New_York"],
 ["America/Nipigon", "-5:00/-4:00 - America/Nipigon"],
 ["America/Pangnirtung", "-5:00/-4:00 - America/Pangnirtung"],
 ["America/Port-au-Prince", "-5:00/-4:00 - America/Port-au-Prince"],
 ["America/Rio_Branco", "-5:00/-4:00 - America/Rio_Branco"],
 ["America/Thunder_Bay", "-5:00/-4:00 - America/Thunder_Bay"],
 ["America/Toronto", "-5:00/-4:00 - America/Toronto"],
 ["America/Anguilla", "-4:00 - America/Anguilla"],
 ["America/Antigua", "-4:00 - America/Antigua"],
 ["America/Aruba", "-4:00 - America/Aruba"],
 ["America/Caracas", "-4:00 - America/Caracas"],
 ["America/Curacao", "-4:00 - America/Curacao"],
 ["America/Dominica", "-4:00 - America/Dominica"],
 ["America/Grenada", "-4:00 - America/Grenada"],
 ["America/Guadeloupe", "-4:00 - America/Guadeloupe"],
 ["America/Guyana", "-4:00 - America/Guyana"],
 ["America/Kralendijk", "-4:00 - America/Kralendijk"],
 ["America/Lower_Princes", "-4:00 - America/Lower_Princes"],
 ["America/Marigot", "-4:00 - America/Marigot"],
 ["America/Montserrat", "-4:00 - America/Montserrat"],
 ["America/Port_of_Spain", "-4:00 - America/Port_of_Spain"],
 ["America/St_Barthelemy", "-4:00 - America/St_Barthelemy"],
 ["America/St_Kitts", "-4:00 - America/St_Kitts"],
 ["America/St_Lucia", "-4:00 - America/St_Lucia"],
 ["America/St_Thomas", "-4:00 - America/St_Thomas"],
 ["America/St_Vincent", "-4:00 - America/St_Vincent"],
 ["America/Tortola", "-4:00 - America/Tortola"],
 ["America/Asuncion", "-4:00/-3:00 - America/Asuncion"],
 ["America/Barbados", "-4:00/-3:00 - America/Barbados"],
 ["America/Blanc-Sablon", "-4:00/-3:00 - America/Blanc-Sablon"],
 ["America/Boa_Vista", "-4:00/-3:00 - America/Boa_Vista"],
 ["America/Campo_Grande", "-4:00/-3:00 - America/Campo_Grande"],
 ["America/Cuiaba", "-4:00/-3:00 - America/Cuiaba"],
 ["America/Glace_Bay", "-4:00/-3:00 - America/Glace_Bay"],
 ["America/Goose_Bay", "-4:00/-3:00 - America/Goose_Bay"],
 ["America/Halifax", "-4:00/-3:00 - America/Halifax"],
 ["America/Manaus", "-4:00/-3:00 - America/Manaus"],
 ["America/Martinique", "-4:00/-3:00 - America/Martinique"],
 ["America/Moncton", "-4:00/-3:00 - America/Moncton"],
 ["America/Porto_Velho", "-4:00/-3:00 - America/Porto_Velho"],
 ["America/Puerto_Rico", "-4:00/-3:00 - America/Puerto_Rico"],
 ["America/Santiago", "-4:00/-3:00 - America/Santiago"],
 ["America/Thule", "-4:00/-3:00 - America/Thule"],
 ["Atlantic/Bermuda", "-4:00/-3:00 - Atlantic/Bermuda"],
 ["America/La_Paz", "-4:00/-3:32 - America/La_Paz"],
 ["America/Santo_Domingo", "-4:00/-4:30 - America/Santo_Domingo"],
 ["America/St_Johns", "-3:30/-2:30 - America/St_Johns"],
 ["America/Araguaina", "-3:00/-2:00 - America/Araguaina"],
 ["America/Argentina/Buenos_Aires", "-3:00/-2:00 - America/Argentina/Buenos_Aires"],
 ["America/Argentina/Catamarca", "-3:00/-2:00 - America/Argentina/Catamarca"],
 ["America/Argentina/Cordoba", "-3:00/-2:00 - America/Argentina/Cordoba"],
 ["America/Argentina/Jujuy", "-3:00/-2:00 - America/Argentina/Jujuy"],
 ["America/Argentina/La_Rioja", "-3:00/-2:00 - America/Argentina/La_Rioja"],
 ["America/Argentina/Mendoza", "-3:00/-2:00 - America/Argentina/Mendoza"],
 ["America/Argentina/Rio_Gallegos", "-3:00/-2:00 - America/Argentina/Rio_Gallegos"],
 ["America/Argentina/Salta", "-3:00/-2:00 - America/Argentina/Salta"],
 ["America/Argentina/San_Juan", "-3:00/-2:00 - America/Argentina/San_Juan"],
 ["America/Argentina/Tucuman", "-3:00/-2:00 - America/Argentina/Tucuman"],
 ["America/Argentina/Ushuaia", "-3:00/-2:00 - America/Argentina/Ushuaia"],
 ["America/Bahia", "-3:00/-2:00 - America/Bahia"],
 ["America/Belem", "-3:00/-2:00 - America/Belem"],
 ["America/Fortaleza", "-3:00/-2:00 - America/Fortaleza"],
 ["America/Maceio", "-3:00/-2:00 - America/Maceio"],
 ["America/Miquelon", "-3:00/-2:00 - America/Miquelon"],
 ["America/Montevideo", "-3:00/-2:00 - America/Montevideo"],
 ["America/Nuuk", "-3:00/-2:00 - America/Nuuk"],
 ["America/Recife", "-3:00/-2:00 - America/Recife"],
 ["America/Sao_Paulo", "-3:00/-2:00 - America/Sao_Paulo"],
 ["America/Argentina/San_Luis", "-3:00 - America/Argentina/San_Luis"],
 ["America/Cayenne", "-3:00 - America/Cayenne"],
 ["America/Paramaribo", "-3:00 - America/Paramaribo"],
 ["America/Punta_Arenas", "-3:00 - America/Punta_Arenas"],
 ["America/Santarem", "-3:00 - America/Santarem"],
 ["Antarctica/Palmer", "-3:00 - Antarctica/Palmer"],
 ["Antarctica/Rothera", "-3:00 - Antarctica/Rothera"],
 ["Atlantic/Stanley", "-3:00 - Atlantic/Stanley"],
 ["America/Noronha", "-2:00/-1:00 - America/Noronha"],
 ["Atlantic/South_Georgia", "-2:00 - Atlantic/South_Georgia"],
 ["America/Scoresbysund", "-1:00/+0:00 - America/Scoresbysund"],
 ["Atlantic/Azores", "-1:00/+0:00 - Atlantic/Azores"],
 ["Atlantic/Cape_Verde", "-1:00 - Atlantic/Cape_Verde"],
 ["Africa/Abidjan", "+0:00 - Africa/Abidjan"],
 ["Africa/Bamako", "+0:00 - Africa/Bamako"],
 ["Africa/Banjul", "+0:00 - Africa/Banjul"],
 ["Africa/Bissau", "+0:00 - Africa/Bissau"],
 ["Africa/Conakry", "+0:00 - Africa/Conakry"],
 ["Africa/Dakar", "+0:00 - Africa/Dakar"],
 ["Africa/Freetown", "+0:00 - Africa/Freetown"],
 ["Africa/Lome", "+0:00 - Africa/Lome"],
 ["Africa/Monrovia", "+0:00 - Africa/Monrovia"],
 ["Africa/Nouakchott", "+0:00 - Africa/Nouakchott"],
 ["Africa/Ouagadougou", "+0:00 - Africa/Ouagadougou"],
 ["Africa/Sao_Tome", "+0:00 - Africa/Sao_Tome"],
 ["Atlantic/Reykjavik", "+0:00 - Atlantic/Reykjavik"],
 ["Atlantic/St_Helena", "+0:00 - Atlantic/St_Helena"],
 ["Africa/Accra", "+0:00/+0:30 - Africa/Accra"],
 ["America/Danmarkshavn", "+0:00/-2:00 - America/Danmarkshavn"],
 ["Antarctica/Troll", "+0:00/+2:00 - Antarctica/Troll"],
 ["Atlantic/Canary", "+0:00/+1:00 - Atlantic/Canary"],
 ["Atlantic/Faroe", "+0:00/+1:00 - Atlantic/Faroe"],
 ["Atlantic/Madeira", "+0:00/+1:00 - Atlantic/Madeira"],
 ["Europe/Guernsey", "+0:00/+1:00 - Europe/Guernsey"],
 ["Europe/Isle_of_Man", "+0:00/+1:00 - Europe/Isle_of_Man"],
 ["Europe/Jersey", "+0:00/+1:00 - Europe/Jersey"],
 ["Europe/Lisbon", "+0:00/+1:00 - Europe/Lisbon"],
 ["Europe/London", "+0:00/+1:00 - Europe/London"],
 ["Africa/Algiers", "+1:00 - Africa/Algiers"],
 ["Africa/Bangui", "+1:00 - Africa/Bangui"],
 ["Africa/Brazzaville", "+1:00 - Africa/Brazzaville"],
 ["Africa/Douala", "+1:00 - Africa/Douala"],
 ["Africa/Kinshasa", "+1:00 - Africa/Kinshasa"],
 ["Africa/Lagos", "+1:00 - Africa/Lagos"],
 ["Africa/Libreville", "+1:00 - Africa/Libreville"],
 ["Africa/Luanda", "+1:00 - Africa/Luanda"],
 ["Africa/Malabo", "+1:00 - Africa/Malabo"],
 ["Africa/Niamey", "+1:00 - Africa/Niamey"],
 ["Africa/Porto-Novo", "+1:00 - Africa/Porto-Novo"],
 ["Africa/Casablanca", "+1:00/+0:00 - Africa/Casablanca"],
 ["Africa/El_Aaiun", "+1:00/+0:00 - Africa/El_Aaiun"],
 ["Europe/Dublin", "+1:00/+0:00 - Europe/Dublin"],
 ["Africa/Ceuta", "+1:00/+2:00 - Africa/Ceuta"],
 ["Africa/Ndjamena", "+1:00/+2:00 - Africa/Ndjamena"],
 ["Africa/Tunis", "+1:00/+2:00 - Africa/Tunis"],
 ["Arctic/Longyearbyen", "+1:00/+2:00 - Arctic/Longyearbyen"],
 ["Europe/Amsterdam", "+1:00/+2:00 - Europe/Amsterdam"],
 ["Europe/Andorra", "+1:00/+2:00 - Europe/Andorra"],
 ["Europe/Belgrade", "+1:00/+2:00 - Europe/Belgrade"],
 ["Europe/Berlin", "+1:00/+2:00 - Europe/Berlin"],
 ["Europe/Bratislava", "+1:00/+2:00 - Europe/Bratislava"],
 ["Europe/Brussels", "+1:00/+2:00 - Europe/Brussels"],
 ["Europe/Budapest", "+1:00/+2:00 - Europe/Budapest"],
 ["Europe/Busingen", "+1:00/+2:00 - Europe/Busingen"],
 ["Europe/Copenhagen", "+1:00/+2:00 - Europe/Copenhagen"],
 ["Europe/Gibraltar", "+1:00/+2:00 - Europe/Gibraltar"],
 ["Europe/Ljubljana", "+1:00/+2:00 - Europe/Ljubljana"],
 ["Europe/Luxembourg", "+1:00/+2:00 - Europe/Luxembourg"],
 ["Europe/Madrid", "+1:00/+2:00 - Europe/Madrid"],
 ["Europe/Malta", "+1:00/+2:00 - Europe/Malta"],
 ["Europe/Monaco", "+1:00/+2:00 - Europe/Monaco"],
 ["Europe/Oslo", "+1:00/+2:00 - Europe/Oslo"],
 ["Europe/Paris", "+1:00/+2:00 - Europe/Paris"],
 ["Europe/Podgorica", "+1:00/+2:00 - Europe/Podgorica"],
 ["Europe/Prague", "+1:00/+2:00 - Europe/Prague"],
 ["Europe/Rome", "+1:00/+2:00 - Europe/Rome"],
 ["Europe/San_Marino", "+1:00/+2:00 - Europe/San_Marino"],
 ["Europe/Sarajevo", "+1:00/+2:00 - Europe/Sarajevo"],
 ["Europe/Skopje", "+1:00/+2:00 - Europe/Skopje"],
 ["Europe/Stockholm", "+1:00/+2:00 - Europe/Stockholm"],
 ["Europe/Tirane", "+1:00/+2:00 - Europe/Tirane"],
 ["Europe/Vaduz", "+1:00/+2:00 - Europe/Vaduz"],
 ["Europe/Vatican", "+1:00/+2:00 - Europe/Vatican"],
 ["Europe/Vienna", "+1:00/+2:00 - Europe/Vienna"],
 ["Europe/Warsaw", "+1:00/+2:00 - Europe/Warsaw"],
 ["Europe/Zagreb", "+1:00/+2:00 - Europe/Zagreb"],
 ["Europe/Zurich", "+1:00/+2:00 - Europe/Zurich"],
 ["Africa/Blantyre", "+2:00 - Africa/Blantyre"],
 ["Africa/Bujumbura", "+2:00 - Africa/Bujumbura"],
 ["Africa/Gaborone", "+2:00 - Africa/Gaborone"],
 ["Africa/Harare", "+2:00 - Africa/Harare"],
 ["Africa/Kigali", "+2:00 - Africa/Kigali"],
 ["Africa/Lubumbashi", "+2:00 - Africa/Lubumbashi"],
 ["Africa/Lusaka", "+2:00 - Africa/Lusaka"],
 ["Africa/Maputo", "+2:00 - Africa/Maputo"],
 ["Africa/Tripoli", "+2:00 - Africa/Tripoli"],
 ["Africa/Cairo", "+2:00/+3:00 - Africa/Cairo"],
 ["Africa/Johannesburg", "+2:00/+3:00 - Africa/Johannesburg"],
 ["Africa/Juba", "+2:00/+3:00 - Africa/Juba"],
 ["Africa/Khartoum", "+2:00/+3:00 - Africa/Khartoum"],
 ["Africa/Maseru", "+2:00/+3:00 - Africa/Maseru"],
 ["Africa/Mbabane", "+2:00/+3:00 - Africa/Mbabane"],
 ["Asia/Amman", "+2:00/+3:00 - Asia/Amman"],
 ["Asia/Beirut", "+2:00/+3:00 - Asia/Beirut"],
 ["Asia/Damascus", "+2:00/+3:00 - Asia/Damascus"],
 ["Asia/Famagusta", "+2:00/+3:00 - Asia/Famagusta"],
 ["Asia/Gaza", "+2:00/+3:00 - Asia/Gaza"],
 ["Asia/Hebron", "+2:00/+3:00 - Asia/Hebron"],
 ["Asia/Jerusalem", "+2:00/+3:00 - Asia/Jerusalem"],
 ["Asia/Nicosia", "+2:00/+3:00 - Asia/Nicosia"],
 ["Europe/Athens", "+2:00/+3:00 - Europe/Athens"],
 ["Europe/Bucharest", "+2:00/+3:00 - Europe/Bucharest"],
 ["Europe/Chisinau", "+2:00/+3:00 - Europe/Chisinau"],
 ["Europe/Helsinki", "+2:00/+3:00 - Europe/Helsinki"],
 ["Europe/Kaliningrad", "+2:00/+3:00 - Europe/Kaliningrad"],
 ["Europe/Kiev", "+2:00/+3:00 - Europe/Kiev"],
 ["Europe/Mariehamn", "+2:00/+3:00 - Europe/Mariehamn"],
 ["Europe/Riga", "+2:00/+3:00 - Europe/Riga"],
 ["Europe/Sofia", "+2:00/+3:00 - Europe/Sofia"],
 ["Europe/Tallinn", "+2:00/+3:00 - Europe/Tallinn"],
 ["Europe/Uzhgorod", "+2:00/+3:00 - Europe/Uzhgorod"],
 ["Europe/Vilnius", "+2:00/+3:00 - Europe/Vilnius"],
 ["Europe/Zaporozhye", "+2:00/+3:00 - Europe/Zaporozhye"],
 ["Africa/Windhoek", "+2:00/+1:00 - Africa/Windhoek"],
 ["Africa/Addis_Ababa", "+3:00 - Africa/Addis_Ababa"],
 ["Africa/Asmara", "+3:00 - Africa/Asmara"],
 ["Africa/Dar_es_Salaam", "+3:00 - Africa/Dar_es_Salaam"],
 ["Africa/Djibouti", "+3:00 - Africa/Djibouti"],
 ["Africa/Kampala", "+3:00 - Africa/Kampala"],
 ["Africa/Mogadishu", "+3:00 - Africa/Mogadishu"],
 ["Africa/Nairobi", "+3:00 - Africa/Nairobi"],
 ["Antarctica/Syowa", "+3:00 - Antarctica/Syowa"],
 ["Asia/Aden", "+3:00 - Asia/Aden"],
 ["Asia/Bahrain", "+3:00 - Asia/Bahrain"],
 ["Asia/Kuwait", "+3:00 - Asia/Kuwait"],
 ["Asia/Qatar", "+3:00 - Asia/Qatar"],
 ["Asia/Riyadh", "+3:00 - Asia/Riyadh"],
 ["Europe/Istanbul", "+3:00 - Europe/Istanbul"],
 ["Europe/Minsk", "+3:00 - Europe/Minsk"],
 ["Europe/Simferopol", "+3:00 - Europe/Simferopol"],
 ["Indian/Antananarivo", "+3:00 - Indian/Antananarivo"],
 ["Indian/Comoro", "+3:00 - Indian/Comoro"],
 ["Indian/Mayotte", "+3:00 - Indian/Mayotte"],
 ["Asia/Baghdad", "+3:00/+4:00 - Asia/Baghdad"],
 ["Europe/Kirov", "+3:00/+4:00 - Europe/Kirov"],
 ["Europe/Moscow", "+3:00/+4:00 - Europe/Moscow"],
 ["Europe/Volgograd", "+3:00/+4:00 - Europe/Volgograd"],
 ["Asia/Tehran", "+3:30/+4:30 - Asia/Tehran"],
 ["Asia/Baku", "+4:00/+5:00 - Asia/Baku"],
 ["Asia/Yerevan", "+4:00/+5:00 - Asia/Yerevan"],
 ["Indian/Mauritius", "+4:00/+5:00 - Indian/Mauritius"],
 ["Asia/Dubai", "+4:00 - Asia/Dubai"],
 ["Asia/Muscat", "+4:00 - Asia/Muscat"],
 ["Asia/Tbilisi", "+4:00 - Asia/Tbilisi"],
 ["Europe/Astrakhan", "+4:00 - Europe/Astrakhan"],
 ["Europe/Samara", "+4:00 - Europe/Samara"],
 ["Europe/Saratov", "+4:00 - Europe/Saratov"],
 ["Europe/Ulyanovsk", "+4:00 - Europe/Ulyanovsk"],
 ["Indian/Mahe", "+4:00 - Indian/Mahe"],
 ["Indian/Reunion", "+4:00 - Indian/Reunion"],
 ["Asia/Kabul", "+4:30 - Asia/Kabul"],
 ["Antarctica/Mawson", "+5:00 - Antarctica/Mawson"],
 ["Asia/Aqtau", "+5:00 - Asia/Aqtau"],
 ["Asia/Ashgabat", "+5:00 - Asia/Ashgabat"],
 ["Asia/Atyrau", "+5:00 - Asia/Atyrau"],
 ["Asia/Oral", "+5:00 - Asia/Oral"],
 ["Indian/Kerguelen", "+5:00 - Indian/Kerguelen"],
 ["Indian/Maldives", "+5:00 - Indian/Maldives"],
 ["Asia/Aqtobe", "+5:00/+6:00 - Asia/Aqtobe"],
 ["Asia/Dushanbe", "+5:00/+6:00 - Asia/Dushanbe"],
 ["Asia/Karachi", "+5:00/+6:00 - Asia/Karachi"],
 ["Asia/Qyzylorda", "+5:00/+6:00 - Asia/Qyzylorda"],
 ["Asia/Samarkand", "+5:00/+6:00 - Asia/Samarkand"],
 ["Asia/Tashkent", "+5:00/+6:00 - Asia/Tashkent"],
 ["Asia/Yekaterinburg", "+5:00/+6:00 - Asia/Yekaterinburg"],
 ["Asia/Colombo", "+5:30/+6:30 - Asia/Colombo"],
 ["Asia/Kolkata", "+5:30/+6:30 - Asia/Kolkata"],
 ["Asia/Kathmandu", "+5:45 - Asia/Kathmandu"],
 ["Antarctica/Vostok", "+6:00 - Antarctica/Vostok"],
 ["Asia/Bishkek", "+6:00 - Asia/Bishkek"],
 ["Asia/Qostanay", "+6:00 - Asia/Qostanay"],
 ["Asia/Thimphu", "+6:00 - Asia/Thimphu"],
 ["Asia/Urumqi", "+6:00 - Asia/Urumqi"],
 ["Indian/Chagos", "+6:00 - Indian/Chagos"],
 ["Asia/Almaty", "+6:00/+7:00 - Asia/Almaty"],
 ["Asia/Dhaka", "+6:00/+7:00 - Asia/Dhaka"],
 ["Asia/Omsk", "+6:00/+7:00 - Asia/Omsk"],
 ["Asia/Yangon", "+6:30 - Asia/Yangon"],
 ["Indian/Cocos", "+6:30 - Indian/Cocos"],
 ["Antarctica/Davis", "+7:00 - Antarctica/Davis"],
 ["Asia/Bangkok", "+7:00 - Asia/Bangkok"],
 ["Asia/Barnaul", "+7:00 - Asia/Barnaul"],
 ["Asia/Ho_Chi_Minh", "+7:00 - Asia/Ho_Chi_Minh"],
 ["Asia/Jakarta", "+7:00 - Asia/Jakarta"],
 ["Asia/Novokuznetsk", "+7:00 - Asia/Novokuznetsk"],
 ["Asia/Novosibirsk", "+7:00 - Asia/Novosibirsk"],
 ["Asia/Phnom_Penh", "+7:00 - Asia/Phnom_Penh"],
 ["Asia/Pontianak", "+7:00 - Asia/Pontianak"],
 ["Asia/Tomsk", "+7:00 - Asia/Tomsk"],
 ["Asia/Vientiane", "+7:00 - Asia/Vientiane"],
 ["Indian/Christmas", "+7:00 - Indian/Christmas"],
 ["Asia/Hovd", "+7:00/+8:00 - Asia/Hovd"],
 ["Asia/Krasnoyarsk", "+7:00/+8:00 - Asia/Krasnoyarsk"],
 ["Asia/Brunei", "+8:00 - Asia/Brunei"],
 ["Asia/Makassar", "+8:00 - Asia/Makassar"],
 ["Asia/Choibalsan", "+8:00/+9:00 - Asia/Choibalsan"],
 ["Asia/Hong_Kong", "+8:00/+9:00 - Asia/Hong_Kong"],
 ["Asia/Irkutsk", "+8:00/+9:00 - Asia/Irkutsk"],
 ["Asia/Macau", "+8:00/+9:00 - Asia/Macau"],
 ["Asia/Manila", "+8:00/+9:00 - Asia/Manila"],
 ["Asia/Shanghai", "+8:00/+9:00 - Asia/Shanghai"],
 ["Asia/Taipei", "+8:00/+9:00 - Asia/Taipei"],
 ["Asia/Ulaanbaatar", "+8:00/+9:00 - Asia/Ulaanbaatar"],
 ["Australia/Perth", "+8:00/+9:00 - Australia/Perth"],
 ["Asia/Kuala_Lumpur", "+8:00/+7:20 - Asia/Kuala_Lumpur"],
 ["Asia/Singapore", "+8:00/+7:20 - Asia/Singapore"],
 ["Asia/Kuching", "+8:00/+8:20 - Asia/Kuching"],
 ["Australia/Eucla", "+8:45/+9:45 - Australia/Eucla"],
 ["Asia/Chita", "+9:00/+10:00 - Asia/Chita"],
 ["Asia/Seoul", "+9:00/+10:00 - Asia/Seoul"],
 ["Asia/Tokyo", "+9:00/+10:00 - Asia/Tokyo"],
 ["Asia/Yakutsk", "+9:00/+10:00 - Asia/Yakutsk"],
 ["Asia/Dili", "+9:00 - Asia/Dili"],
 ["Asia/Jayapura", "+9:00 - Asia/Jayapura"],
 ["Asia/Pyongyang", "+9:00 - Asia/Pyongyang"],
 ["Pacific/Palau", "+9:00 - Pacific/Palau"],
 ["Asia/Khandyga", "+9:00/+11:00 - Asia/Khandyga"],
 ["Australia/Adelaide", "+9:30/+10:30 - Australia/Adelaide"],
 ["Australia/Broken_Hill", "+9:30/+10:30 - Australia/Broken_Hill"],
 ["Australia/Darwin", "+9:30/+10:30 - Australia/Darwin"],
 ["Antarctica/DumontDUrville", "+10:00 - Antarctica/DumontDUrville"],
 ["Pacific/Chuuk", "+10:00 - Pacific/Chuuk"],
 ["Pacific/Port_Moresby", "+10:00 - Pacific/Port_Moresby"],
 ["Antarctica/Macquarie", "+10:00/+11:00 - Antarctica/Macquarie"],
 ["Asia/Vladivostok", "+10:00/+11:00 - Asia/Vladivostok"],
 ["Australia/Brisbane", "+10:00/+11:00 - Australia/Brisbane"],
 ["Australia/Currie", "+10:00/+11:00 - Australia/Currie"],
 ["Australia/Hobart", "+10:00/+11:00 - Australia/Hobart"],
 ["Australia/Lindeman", "+10:00/+11:00 - Australia/Lindeman"],
 ["Australia/Melbourne", "+10:00/+11:00 - Australia/Melbourne"],
 ["Australia/Sydney", "+10:00/+11:00 - Australia/Sydney"],
 ["Pacific/Guam", "+10:00/+11:00 - Pacific/Guam"],
 ["Pacific/Saipan", "+10:00/+11:00 - Pacific/Saipan"],
 ["Asia/Ust-Nera", "+10:00/+12:00 - Asia/Ust-Nera"],
 ["Australia/Lord_Howe", "+10:30/+11:00 - Australia/Lord_Howe"],
 ["Antarctica/Casey", "+11:00 - Antarctica/Casey"],
 ["Asia/Sakhalin", "+11:00 - Asia/Sakhalin"],
 ["Pacific/Bougainville", "+11:00 - Pacific/Bougainville"],
 ["Pacific/Guadalcanal", "+11:00 - Pacific/Guadalcanal"],
 ["Pacific/Kosrae", "+11:00 - Pacific/Kosrae"],
 ["Pacific/Pohnpei", "+11:00 - Pacific/Pohnpei"],
 ["Asia/Magadan", "+11:00/+12:00 - Asia/Magadan"],
 ["Asia/Srednekolymsk", "+11:00/+12:00 - Asia/Srednekolymsk"],
 ["Pacific/Efate", "+11:00/+12:00 - Pacific/Efate"],
 ["Pacific/Norfolk", "+11:00/+12:00 - Pacific/Norfolk"],
 ["Pacific/Noumea", "+11:00/+12:00 - Pacific/Noumea"],
 ["Antarctica/McMurdo", "+12:00/+13:00 - Antarctica/McMurdo"],
 ["Pacific/Auckland", "+12:00/+13:00 - Pacific/Auckland"],
 ["Pacific/Fiji", "+12:00/+13:00 - Pacific/Fiji"],
 ["Asia/Anadyr", "+12:00 - Asia/Anadyr"],
 ["Asia/Kamchatka", "+12:00 - Asia/Kamchatka"],
 ["Pacific/Funafuti", "+12:00 - Pacific/Funafuti"],
 ["Pacific/Kwajalein", "+12:00 - Pacific/Kwajalein"],
 ["Pacific/Majuro", "+12:00 - Pacific/Majuro"],
 ["Pacific/Nauru", "+12:00 - Pacific/Nauru"],
 ["Pacific/Tarawa", "+12:00 - Pacific/Tarawa"],
 ["Pacific/Wake", "+12:00 - Pacific/Wake"],
 ["Pacific/Wallis", "+12:00 - Pacific/Wallis"],
 ["Pacific/Chatham", "+12:45/+13:45 - Pacific/Chatham"],
 ["Pacific/Apia", "+13:00/+14:00 - Pacific/Apia"],
 ["Pacific/Tongatapu", "+13:00/+14:00 - Pacific/Tongatapu"],
 ["Pacific/Enderbury", "+13:00 - Pacific/Enderbury"],
 ["Pacific/Fakaofo", "+13:00 - Pacific/Fakaofo"],
 ["Pacific/Kiritimati", "+14:00 - Pacific/Kiritimati"]]
```
