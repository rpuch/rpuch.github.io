---
layout: post
title: What's the problem with yyyy-MM-dd'T'HH:mm:ss.SSSZ?
date: '2020-08-25T22:00:00.000+04:00'
author: rpuchkovskiy
tags:
- java
- iso-8601
- format
- timezone designator
- year of era
- proleptic year
excerpt_separator: <!--more-->
---

There are many pages in Internet that recommend or mention `yyyy-MM-dd'T'HH:mm:ss.SSSZ` pattern when dealing with
ISO-8601 date-times, for instance Spring documentation
https://docs.spring.io/autorepo/docs/spring/4.0.2.RELEASE/javadoc-api/org/springframework/format/annotation/DateTimeFormat.ISO.html
and Elasticsearch documentation https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-date-format.html,
to name a couple. But this format is actually troublesome when using ISO-8601 format.

<!--more-->

# Problem 1: timezone

Let's format a date-time value using this format.

```
    DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss.SSSZ");

    System.out.println(formatter.format(ZonedDateTime.now()));
```

This produces the following:

```
2020-08-24T21:49:31.702+0400
```

Looks good, but it isn't. The problem is that ISO-8601 dictates that if the date-time part (everything before
the timezone designation) uses extended format (that is, contains hyphen and colon as delimiters), then the timezone
part must also be in extended format. In our case, the date-time part is in extended format, but the timezone part is
in the basic format (as the colon between `04` and `00` is omitted).

If we parse the produced string using `ZonedDateTime.parse()`, we'll get an exception:

```
Exception in thread "main" java.time.format.DateTimeParseException: Text '2020-08-24T21:49:31.702+0400' could not be parsed, unparsed text found at index 26
```

Another problem is that `Z` would produce `+0000` for UTC, whereas ISO-8601 requires one of the following:

* either `±HH:mm` if any of HH or mm is non-zero
* or `Z` to designate UTC

This means that `z` symbol recommended as an alternative for `Z` will not work either.

The correct answer is `XXX`, so the intermediate result will be `yyyy-MM-dd'T'HH:mm:ss.SSSXXX`.

Why *intermediate*? Because there is a second problem.

# Problem 2: year

Let's run the following code:

```
    DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss.SSSXXX");
    TemporalAccessor parsed = formatter.parse("2020-08-24T21:49:31.702+04:00");
    System.out.println(ZonedDateTime.from(parsed));
```

It works fine. But let's modify it a bit:

```
    DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss.SSSXXX")
            .withResolverStyle(ResolverStyle.STRICT);
    TemporalAccessor parsed = formatter.parse("2020-08-24T21:49:31.702+04:00");
    System.out.println(ZonedDateTime.from(parsed));
```

We just asked the formatter to be strict when resolving date-time fields. The result follows:

```
Exception in thread "main" java.time.DateTimeException: Unable to obtain ZonedDateTime from TemporalAccessor: {DayOfMonth=24, OffsetSeconds=14400, YearOfEra=2020, MonthOfYear=8},ISO resolved to 21:49:31.702 of type java.time.format.Parsed
```

So the formatter has parsed the date-time value, but `ZonedDateTime` cannot be extracted from this parsed
representation!

## How year is resolved

To construct a `ZonedDateTime`, java.time needs to know the exact year. There are two flavors of 'year' field:

* 'year-of-era' (this is the familiar `yyyy`); can only be positive
* proleptic year (the corresponding pattern is `uuuu`); can be positive, zero, or negative (so any integer)

Strictly speaking, to resolve a year from year-of-era, java.time needs to know era, naturally. Who knows what was meant
by year '2000': is it '2000 BC' or '2000 AD'? (By the way, era is denoted by `G` pattern). But everybody cares about
AD years most of the time, so, if we are not so strict, we could still resolve a year from just 'year-of-era'.

On the other hand, *proleptic year* is a year how a mathematician would defined it: it may be positive (for years AD)
or zero/negative (or years BC). Proleptic year is self-sufficient, it does not need era to be resolved, in any context.

So now it can easily be seen what happened in the example: as soon as we entered strict context, the sole 'yyyy'
(year-of-era) is not enough anymore, it requires an era which is not supplied (because ISO-8601 format does not use
era designation).

Also, it is easy to see how this can be fixed: **always use `uuuu` instead of `yyyy`, unless you specifically need
year-of-era semantics**.

# Final pattern (so far)

It's `uuuu-MM-dd'T'HH:mm:ss.SSSXXX`. This allows to format/parse ISO-8601-compliant date-time values even in
strict contexts.