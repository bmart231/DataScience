# Time Series

Time series data is an important form of structured data in many different
fields, such as finance, economics, ecology, nueroscience, and physics. Anything
that is recorded repeatedly at many points in time forms a time series. Many
time series are _fixed frequency_, which is to say that data points occur at
regular intervals according to some rule, such as every 15 seconds, every 5
minutes, or once per month. Time series can also be _irregular_ without a fixed
unit of time or offset between units. How you mark and refer to time series data
depends on the application, and you may have one of the following:

**Timestamps**: Specific instants in time.

**Fixed periods**: Such as the whole month of Janurary 2017, or the whole
year 2020.

**Intervals of time**: Indicated by a start and end timestamp. Periods can be
thought of as special cases of intervals.

**Experiment or elapsed time**: Each timestamp is a measure of time relative to
a particular start time (e.g., the diameter of a cookie baking each second since
being placed in the oven), starting from 0.

The simplest kind of time series is indexed by timestamp.

`pandas` provides many built-in time series tools and algorithms. You can
efficiently work with large time series, and slice and dice, aggregate, and
resample irregular and fixed-frequency time series. Some of these tools are
useful for financial and economics applications, but you could certainly use
them to analyze server log data, too.

Contents:

- 11.1: Date and Time Data Types and Tools
  - Converting Between String and Datetime
- 11.2: Time Series Basics
  - Indexing, Selection, Subsetting
  - Time Series with Duplicate Indexes
- 11.3: Date Ranges, Frequencies, and Shifting
  - Generating Date Ranges
  - Frequencies and Date Offsets
  - Shifting (Leading and Lagging) Data
- 11.4: Time Zone Handling
  - Time Zone Localization and Conversion
  - Operations with Time Zone-Aware Timestamp Objects
  - Operations between Different Time Zones
- 11.5: Periods and Period Arithmetic
  - Period Frequency Conversion
  - Quarterly Period Frequency
  - Converting Timestamps to Periods (and Back)
  - Creating a PeriodIndex from Arrays
- 11.6: Resampling and Frequency Conversion
  - Downsampling
  - Upsampling and Interpolation
  - Resampling with Periods
  - Grouped Time Resampling
- 11.7: Moving Window Functions
  - Exponentially Weighted Functions
  - Binary Moving Window Functions
  - User-Defined Moving Window Functions
- 11.8: Conclusion
