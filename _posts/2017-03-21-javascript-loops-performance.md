---
layout: post
title:  "Javascript map performance"
date:   2017-03-22 10:00:00 +0100
categories: rails coding
---

Recently I had to track down some performance issues on a Typescript application I'm working on. In that application
I have to loop over many items. As a ruby programmer I'm used to do stuff like:

```ruby
evens = [2, 4, 6, 8, 10]
odds = evens.map { |n| n -1 }
```

Array.map is available in Javascript (typescript) as well:

```typescript
let evens = [2, 4, 6, 8, 10];
odds = evens.map( n => n-1 );
```

But actually this is about [95% slower](https://jsperf.com/native-map-versus-array-looping/36) (in Edge) than a classic for loop. Only the Javascript engine
of Firefox seems to be optimized for `Array.map`. As this is not a problem in most cases, but if you need to iterate
thousands of items, you probably should go for the classic for loop.

## Side note on Ruby

In ruby, map is actually faster than using Array#each and push and even faster than a `for` block.