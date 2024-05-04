---
layout: single
tag: tech
title: "Adapting Data Models in Python"
---

One interesting problem that we face often when integrating systems, be it from different companies or within the same organisation, is transforming data from the model used in one system to the model used in the other system.
{: .text-justify}

Faced with the challenge of having to do this quite often, I designed `py-transmuter`, a [Python library](https://pypi.org/project/py-transmuter/) that simplifies the process of transforming data objects by allowing the user to simply create a static definition of how the fields of one model map to the other and automating the transformation itself.
{: .text-justify}

# The Problem

It is not unlikely that at some point in our development career we'll come across the task of fetching data from some external party and store it in our systems. It is
also not unlikely that, when doing this, we'll need to adapt the information somehow because _**their**_ data model is different to _**our**_ data model.
{: .text-justify}

## A concrete example

This sounds all very nice, but without a concrete example it might be difficult to fully grasp what kind of problem exactly we want to solve exactly.
{: .text-justify}

Imagine your organisation uses weather data as one of the main inputs for calculating the output of ✨ an amazing product ✨. Imagine also that this data
is stored in a very specific way: a time series where each entry represents 15 minute intervals with a temperature value in Celsius.
{: .text-justify}

Now the trick is that this data is ingested from different providers, and each provider has their own way of storing their data; let's look at some of them:
{: .text-justify}
1. **MericaWeather**: They have 15 minute intervals (in `EST` timezone) and their temperature measures are in Fahrenheit.
2. **SunWatch**: They have 1 minute intervals (in `UTC` timezone) that have measures in Celsius.

More specifically, each reading in their responses looks like:
```python
@dataclass
class MericaWeatherReading:
	quarter: datetime  # 15 minute intervals in EST
	temperature: Decimal  # Fahrenheit values

@dataclass
class SunWatchReading:
	timestamp: datetime  # 1 minute intervals in UTC
	measurement: Decimal  # Celsius values
```

Of course, normally you’d get a bunch of readings at a time, so we can assume that their actual API responses are something like:
{: .text-justify}
```python
class MericaWeatherResponse:
	readings: list[MericaWeatherReading]

class SunWatchResponse:
	readings: list[SunWatchReading]
```

To complete the example we need to know how our model looks, which could be something like:
{: .text-justify}
```python
@dataclass
class IntervalTemperature:
	interval_start: datetime  # 15 minute intervals in UTC
	value: Decimal  # Celsius values
```

Now that we have our problem definition clear, we can proceed to what `py-transmuter` can do for you.
{: .text-justify}
# The Solution
What we need to do should be very clear by now:
* For **MericaWeather** we need to transform each entry’s `quarter` to `UTC` and assign it to `interval_start` and `temperature` to Celsius and assign it to `value`.
* For **SunWatch** we need to aggregate every 15 entries and average the values (this is an  arbitrary decision); so we’d grab the first `timestamp` of each 15 entry group and set it as our `interval_start` and we’d set our `value` as the average of each of the group’s entries `measurement`. 

Each of these problems can be represented as two different things: the first one is simply mapping one entity from one model to another and the second one requires certain aggregations, aside from the mapping. Let’s look at each one individually.
{: .text-justify}

## Mapping
The great thing about `py-transmuter` is that all you need to do is define how models map to each other and what functions we need to transform specific fields. A `Mapper` for the transformation `MericaWeatherReading -> IntervalTemperature` would look like:
{: .text-justify}
```python
import pytz
from py_transmuter.models.mapper import ModelMapper

def to_utc(est_timestamp: datetime) -> datetime:
	return est_timestamp.astimezone(pytz.UTC)

def fahrenheit_to_celsius(fahrenheit: Decimal) -> Decimal:
	return (farenheit - 32) × 5/9

class MericaWeatherMapper(ModelMapper[IntervalTemperature, MericaWeatherReading]):
	mappings = {
		"interval_start": ("quarter", to_utc),
		"value": ("temperature", fahrenheit_to_celsius),
	}
```

Now that we have our mapper defined, getting our `IntervalTemperature` objects is as simple as:
{: .text-justify}
```python
response = MericaWeatherResponse(readings=...)
mapped_objects = MericaWeatherMapper().map_list(response.readings)
```

And that’s it!
## Aggregating
For aggregating multiple instances of the “source model” into our “target model”, we need to do something similar; let’s get right into it:
{: .text-justify}
```python
from statistics import mean
from py_transmuter.models.aggregator import ModelAggregator

first = lambda iterable: iterable[0]

def closest_fifteenth_minute(reading: SunWatchReading) -> datetime:
	"""This method always returns the start of a 15 minute interval. 
    If the reading has timestamp 14:13:00, this will return 14:00:00."""
	timestamp = reading.timestamp
    closest_15 = (timestamp.minute // 15) * 15
    minute_difference = closest_15 - timestamp.minute
    new_dt = timestamp + timedelta(minutes=minute_difference)
    return new_dt.replace(second=0, microsecond=0)

class SunWatchAggregator(ModelAggregator[IntervalTemperature, SunWatchReading]):
	group_by = (closest_fifteenth_minute,)	
	sort_by = ("timestamp",)

	aggregations = {
		"interval_start": ("timestamp", first),
		"value": ("measurement", mean),
	}
```

As with the mapper, getting our `IntervalTemperature` objects now is as simple as doing:
{: .text-justify}
```python
response = SunWatchResponse(readings=...)
aggregated_objects = SunWatchAggregator().aggregate(response.readings)
```

Now this might look a little bit more complex, but what’s important is that we’re only focusing on the aggregation logic, without caring about object instantiation. Let’s look a little bit more in detail what’s happening here.
{: .text-justify}

When we tell our `Aggregator` to `sort_by` the `timestamp` field, we ensure that all the provided objects (in our case the `response.readings`) will be processed ascending by the indicated field. This allows us to define all our aggregation functions knowing that the received records will be in whichever order we indicated; in this case, ascending by `timestamp`.
{: .text-justify}

By telling the `Aggregator` that we want to `group_by` using the `closest_fifteenth_minute` function, we ensure that all the entries that are within the same 15 minute interval will be aggregated together; this is, any reading between `XX:00` and `XX:14` will be together, as well as any reading between `XX:15` and `XX:29` and so on.
{: .text-justify}

Finally, knowing that we’re aggregating readings within the same fifteen minute interval that are also sorted by their `timestamp`, we can simply create our reading objects by setting the `interval_start` as the first `timestamp` of the group (which will always be the beginning of the interval) and the `value` as the mean of all the `measurement`s in the group.
{: .text-justify}