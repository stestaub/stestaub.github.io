---
layout: post
title:  "GPX modification with Nokogiri"
date:   2020-10-16 12:00:00 +0100
categories: nokogiry gpx android
---

# GPX modification with Nokogiri
GPX is a xml based file, often provided by GPS trackers. It can also be used for the Android Simulator to simulate location updates. That was exactly what we were using it for.

## Real live GPX data vs Simulation
The issue is, that if you want to use the GPX file recorded in reallife to test you app, you sometimes have the issue, that its going very slowly. In our case, the track was > 1 hour. 

Now I want that one hour track happening in 2 minutes. To do that, I simply modified the timestamps of each track point element.

## Decreaseing duration between trkpts in GPX

This is a simple ruby script to change the speed of a gpx track based using Rails. Basically nokogiri and activesupport for datetime would suit but it was easier to just require rails.

```ruby
require 'rails'
doc = Nokogiri::XML(File.read("/path/to/the/file.gpx"))
track_points_count = doc.css('trkpt').length

start_time = DateTime.parse('2020-06-13T16:00:00')

# lets decrease to target duration of 2 minutes
target_duration = 12.0.seconds
increment = target_duration / track_points_count

# Loop through all track points and calculate its timestamp
timestamps = xml.css('trkpt').each_with_index do |trkpt, idx| 
  trkpt.at_css('time').content = (start_time + (idx+1)*increment).iso8601 
end
```

This does simpli start with the `start_time` and increases equally for each track point. If the trackpoints originally have a lot different durations between them (meaning the speed of the track is not constant),a more sophisticated approach would be required, by taking the original times into account.
