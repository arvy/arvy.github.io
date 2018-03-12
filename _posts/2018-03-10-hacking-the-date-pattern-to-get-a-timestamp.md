---
layout: post
title: Hacking The Date Pattern To Get A Timestamp
---

It's quite common to be running multiple instances of an app in production,
in most cases - in multiple datacenters. It is also a standard practice to aggregate
their logs in a single location/system and analyze them together as a whole, whether
you're hosting such a system in-house (such as Logstash, Graylog) or using one
in the cloud (such as Splunk, Sumologic, etc.). When you do this, it's
important to make sure you have proper timestamps captured in the logs, so that
the log aggregation system can properly localize them. For
instance, you may have instances running across timezones A, B and C, yet when
you view their logs, based in timezone D, you would prefer to see them normalized to
your local time. It also makes the log files portable - if you send them to your
colleague or to an external consultant to analyze, you do not need to specify which
timezone it was captured in.

It's quite perplexing then that despite all the date pattern macros that popular
open source frameworks, [Logback](https://logback.qos.ch/) and [Log4j](https://logging.apache.org/log4j/2.x/), implement, both make materializing
a [standard](https://en.wikipedia.org/wiki/ISO_8601) "timestamp" in the log quite tricky.
As a reminder, a "timestamp", in whatever form it may be presented, indicates an absolute,
specific instant in time. This means that in addition to the date and time it must
convey the timezone. Without this 3rd, critical piece of information, it is not
a timestamp - it is simply a relative time.

Here are some examples of timestamps that indicate the same instant in time:

    1520708400 //linux timestamp - the timezone here is implied (UTC)
    03/10/2018 12:00pm PST
    noon on March 10th, 2010 in Los Angeles
    2018-03-10T12:00:00-09:00 //ISO8061-compliant timestamp
    2018-03-10T21:00:00Z //ISO8061-compliant timestamp, in "Zulu" time (GMT)

The following are not timestamps:

    2018-03-10 12:00:00.000
    03/10/2018 12:00 (system tz)
    midday on March 10th, 2018

### Is a standard (ISO-8601 compliant) ts too much to ask for?
I came across this issue a while back and have since been hacking my timestamp
format to obtain a a proper, ISO-8601 compliant, timestamp (`2018-03-10T21:00:00,000Z`) as
follows:

[Logback](https://logback.qos.ch/manual/layouts.html):
 `%date{yyyy-MM-dd'T'HH:mm:ss.SSS,UTC}Z`
 Logback's `ISO8601` macro produces a datetime which is not ISO-8601 compliant,
 because it does not use a `T` to separate the date the time fields. [See this JIRA](https://jira.qos.ch/browse/LOGBACK-262?page=com.atlassian.jira.plugin.system.issuetabpanels%3Acomment-tabpanel).

[Log4J2](https://logging.apache.org/log4j/2.x/manual/layouts.html#PatternLayout):
`%d{ISO8601}{UTC+0}Z`

Not pretty, for sure.

In both cases, I force the timestamp presentation to UTC timezone and append
a static `Z` character to indicate it. Per ISO-8601 the last component of the
timestamp indicates the timezone or timezone offset from UTC. `Z` in this case
indicates [Zulu](https://en.wikipedia.org/wiki/Zulutime) time (or UTC), which is
equivalent to `+00:00`.


### Unintutive semantics
Both frameworks allow you to specify a timezone, which will then force the
timestamp representation to that timezone, but they neglect to include what
that timezone is! The ability to force the display of the ts to a particular
timezone is of little value if you're aggregating the logs. But I would expect
there to be an optional macro for displaying the timezone. For example:
`%d{ISO8601}{tz}`.

### Serendipidous unicorn encounter
A few days ago, I just stumbled across [this Stackoverflow post](https://stackoverflow.com/questions/23096129/make-logback-include-the-t-between-date-and-time-in-its-date-format-for-str),
which tipped me off to a hidden Logback date macro, which does just that: `XXX`!
I haven't been able to find it in the docs, but it does work (tested with `logback:1.1.3`).

Ideally, I would (and initially did, naively) expect `%d{ISO8601}` to yield the
proper timestamp with timezone and all (`2018-03-10T21:00:00,000Z`) in both frameworks.
It is after all what the standard calls for.
Let's hope this gets fixed in future versions of these logging frameworks.

Until then..

  | **Logback** | `%date{yyyy-MM-dd'T'HH:mm:ss.SSSXXX,UTC}` |
  | **Log4J2**  | `%d{ISO8601}{UTC+0}Z`                     |
