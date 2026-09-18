# Golden Gate Henge calculations

This site estimates when the Sun will appear to set between the two pillars of the Golden Gate Bridge's at a given location. It uses solar position data to calculate sunset, then checks the Sun's direction from the location against the bridge's direction.

# How the files work together

Downloaded DE440 and IERS files (from NASA and the IERS) 
-> generate-solar-ephemeris.py (uses mathematical information in the files to calculate values for solar ephemeris)
-> solar-ephemeris.js (solar data) 
-> solar-calculations.js (sunset calculations)
-> index_fa26.html (GGH dates and map)

# 1. Prepare the data

Download the DE440 file via https://ssd.jpl.nasa.gov/ftp/eph/planets/bsp/de440s.bsp, which mathematically describes the positions of the Sun, Moon, and major planets, and the IERS file via https://datacenter.iers.org/data/9/finals2000A.all, which mathematically describes Earth's rotation and orientation. Then run
`generate-solar-ephemeris.py` with the folder containing those files.

The Python script reads the downloaded files, uses the Skyfield library to calculate the values of right ascension, solar declination, distance (from the Earth's center to the Sun), Greenwich apparent sidereal time, polar motion x (x-component of Earth's rotational-pole shift), and polar motion y (y-component of Earth's rotational-pole shift) at three times in each day, then uses the NumPy library to find coefficients of cubic formulas that passes through those values, which are written into `solar-ephemeris.js`. These values are required for the calculations discussed in section 2. The script does not fetch fresh source files itself; the table is generated ahead of time so visitors need not download the full astronomy files or run Python.

`solar-ephemeris.js` assigns the table to `globalThis.GGHEphemeris`. This creates
a global object called `GGHEphemeris` that `solar-calculations.js` can read.

An ephemeris is a table describing an object's position over time. Each row
here represents one UTC day and contains coefficients used for cubic formulas to estimate the aformentioned values (see paragraph 2) at different times within that day.

# 2. Calculate sunset for a location

`solar-calculations.js` reads `GGHEphemeris` and creates another global object,
`GGHSolar`, with two functions and one number:

`GGHSolar.sunset(year, dayOfYear, latitude, longitude)`, which returns `timeMs`, the sunset time that day in milliseconds and `azimuth`, the Sun's apparent angle at sunset, or null if there is no sunset

`GGHSolar.position(timeMs, latitude, longitude)`, which returns `altitude`, the Sun's angle above the horizontal, and `azimuth`, the Sun's apparent angle at the requested time

`GGHSolar.sunsetAltitude`, the number -0.833, used as the target solar altitude for sunset

***
`timeMs` is the time in milliseconds since Jan 1, 1970 at 00:00 UTC.
`azimuth` is direction clockwise from true north, in degrees, eg. West is 270.
`altitude` is the center of the Sun's angle above horizontal, in degrees; negative means below the horizontal.
`dayOfYear` starts at 1 on Jan 1 of the specified year.

The wrapping function keeps the internal helpers private. It makes `GGHSolar`
global so the HTML can call its functions. Both `sunset` and `position` are
functions; the `azimuth` values they return are numbers.

# 3. Find GGH dates and display them

`index_fa26.html` loads the two previous JavaScript files, then does the following:

1. `updatePinDateRanges()` goes through the locations in `PINS`.
2. `calculatePinWindow()` checks each day in the fall and spring periods.
3. `sunsetForDay()` calls `GGHSolar.sunset()` for that day and location.
4. The page compares the returned `sunset.azimuth` with the azimuths/angles of the accepted bridge bounds, which we define as a deviation of one sixth of the bridge's total length from its center or less (see section: Alignment definition and date coverage).
5. It stores the first and last qualifying dates on the pin as `gghStartFa`,
   `gghEndFa`, `gghStartSp`, and `gghEndSp`.

These dates are calculated when the page loads. The stored dates count continuously from Jan 1, 2026: day 1 is Jan 1, 2026, and day 366 is Jan 1, 2027.

The page uses the date ranges for the pin labels and date filter. It uses
the same solar calculations to draw the gold band and show sunset times for
selected or dropped pins. A pin without qualifying dates shows
“No dates in this period”.

The prepared gold-band shapes load into the map once. Moving the date slider
switches visibility between the old and new day's shapes, so the map does not
reprocess geometry on every movement. Pin data and menu rows are updated only
when their visible list changes.

# File reference

`generate-solar-ephemeris.py`: Reads the downloaded astronomy files and writes the browser's solar data table. Can also regenerate the test reference data.
`solar-ephemeris.js`: Stores the generated table in the global `GGHEphemeris` object.
`solar-calculations.js`: Uses that table to calculate solar positions and sunset through `GGHSolar`.
`index_fa26.html`: Runs the website; contains the locations, bridge bounds, date checks, map, controls, and popups for fall 2026 and spring 2027.
`gold-band.js`: Prepared daily map polygons, avoiding expensive geometry calculations while dragging the date slider.
`generate-gold-band.cjs`: Generates those polygons using the page's exact geometry code and solar data.
`index_2026.html`: Original site from spring 2026 using earlier approximate calculations.
`tests/fixtures/solar-reference.json`: Stores comparison dates and sunset results calculated independently with the full Skyfield model.
`tests/solar.test.cjs`: Checks the website's calculations and date handling against those reference results.
`SOLAR_MODEL.md`: Explains this flow and how to regenerate and check the data.

# Alignment definition and date coverage

A location qualifies for Golden Gate Henge when the Sun's center at sunset falls within the middle third of the bridge's angular width centered on the bearing to its midpoint (from the location):

bridge delta = angular separation of the two towers
allowed delta = bridge delta / 3
upper bound = midpoint direction + allowed delta / 2
lower bound = midpoint direction - allowed delta / 2

Sunset is when the descending Sun's center reaches −0.833° solar altitude. This
includes the usual allowance for atmospheric refraction and the Sun's radius.
Observer elevation is set to zero for all locations. The model estimates alignment at nominal sunset (when the Sun crosses the horizon); it does not calculate when the Sun crosses the bridge roadway.

- Slider: Oct 1, 2026 through March 31, 2027.
- Pin date checks: Sep–Dec 2026 and Jan–March 2027.
- Solar data coverage: Aug 31, 2026 through Apr 2, 2027 UTC. Calculations outside this coverage raise an error.

# Regenerate and check the data

Install `skyfield==1.55` and `numpy` in a separate Python environment. Download
these files into the same data folder:

- JPL de440s.bsp: https://ssd.jpl.nasa.gov/ftp/eph/planets/bsp/de440s.bsp
- IERS finals2000A.all: https://datacenter.iers.org/data/9/finals2000A.all

Note: the DE440 file need not be redownloaded; it contains information covering through 2150.

From the project folder, run:

```sh
python generate-solar-ephemeris.py --data-dir /path/to/data --references
node generate-gold-band.cjs
node tests/solar.test.cjs
```

To verify the saved gold bands later without regenerating them, run
`node generate-gold-band.cjs --check`.

Replace `/path/to/data` with your data folder. The optional `--references` flag
also updates `tests/fixtures/solar-reference.json` using full Skyfield
calculations. The generator records fingerprints of the two source files so
we can identify which data produced the table.

To cover another season, update the date ranges in the generator and HTML,
along with the reference-generation and test periods, then regenerate and test.
Running the script again without changing those ranges keeps the same coverage.

The current tests check 6,996 pin/day combinations, 829 reference sunsets, map
boundaries, filters, popup dates, year rollover, and daylight saving changes.
The checked sunset results agree with full Skyfield to within 0.000001° in
azimuth and 0.001 second in time. Those numbers describe agreement between
calculations. Actual viewing conditions also depend on refraction, terrain,
observer height, and the exact location.

# Publish the page

Keep `index_fa26.html`, `solar-ephemeris.js`, `solar-calculations.js`, and `gold-band.js` together
in the same folder when publishing. Visitors run these HTML and JavaScript
files. The Python generator, source astronomy files, and tests are only needed
for preparing and checking the data.

# Sources

- JPL DE440: https://ssd.jpl.nasa.gov/doc/de440_de441.html
- Skyfield solar positions: https://rhodesmill.org/skyfield/positions.html
- Skyfield Earth orientation: https://rhodesmill.org/skyfield/accuracy-efficiency.html
- USNO sunset definition: https://aa.usno.navy.mil/faq/RST_defs
