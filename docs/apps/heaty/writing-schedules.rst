Writing Schedules
=================

A schedule controls the temperature in a room over time. It consists
of a set of rules.

Each rule must at least define a temperature:

::

    schedule:
    - temp: 16

This schedule would just always set the temperature to ``16``
degrees, nothing else. Of course, schedules wouldn't make a lot
sense if they couldn't do more than that.


Basic Scheduling Based on Time of the Day
-----------------------------------------

Here is another one:

::

    schedule:
    - temp: 21.5
      start: "7:00"
      end: "22:00"
      name: Fancy Rule
    - temp: 16

This schedule contains the same rule as the schedule before, but
additionally, it got a new one. The new rule overwrites the second
and will set a temperature of ``21.5`` degrees, but only from 7.00 am
to 10.00 pm. This is because it's placed before the 16-degrees-rule
and Heaty evaluates rules from top to bottom. That is how schedules
work. The first matching rule wins and specifies the temperature to
set. The ``name`` parameter we specified here is completely optional
and doesn't influence how the rule is interpreted. A rule's name is
shown in the logs and may be useful for troubleshooting.

For more fine-grained control, you may also specify seconds in addition to
hour and minute. ``22:00:30`` means 10.00 pm + 30 seconds, for instance.

If you omit the ``start`` parameter, Heaty assumes that you mean midnight
(``00:00``) and fills that in for you. When ``end`` is not specified,
Heaty sets ``00:00`` for it as well. This alone wouldn't make sense,
because the resulting rule would stop being valid the same moment it
starts at.

To achieve the behaviour we'd expect, Heaty applies another
check. Whenever the end time is less or equal to the start time, it
increases another attribute called ``end_plus_days`` (which defaults
to ``0``) by ``1``. This means that the rule is valid up to the time
specified in the ``end`` field, but one day later. Cool, right?

Having done the same manually would result in the following schedule,
which behaves exactly like the previous one.

::

    schedule:
    - { temp: 21.5, start: "7:00", end: "22:00", name: "Fancy Rule" }
    - { temp: 16,   start: "0:00", end: "0:00", end_plus_days: 1 }

Note how each rule has been rewritten to take just a single line.
This is no special feature of Heaty, it's rather normal YAML. But
writing rules this way is often more readable, especially if you
need to create multiple similar ones which, for instance, only
differ in weekdays, time or temperature.

Having that said, it is always a good idea to add a fallback rule
(one with just a temperature and neither ``start`` nor ``end``) as the
last rule in your schedule, which specifies a temperature to use when
no other rule matched.

Now we have covered the basics, but we can't create schedules based
on, for instance, the days of the week. Let's do that next.


Constraints
-----------

::

    schedule:
    - temp: 22
      weekdays: 1-5
      start: "7:00"
      end: "22:00"

    - temp: 22
      weekdays: 6,7
      start: "7:45"

    - temp: 15

With your knowledge so far, this should be self-explanatory. The only
new parameter is ``weekdays``, which is a so called constraint.

Constraints can be used to limit the days on which the rule is
considered. There are a number of these constraints, namely:

* ``years``: limit the years (e.g. ``years: 2016 - 2018``
* ``months``: limit based on months of the year (e.g.
  ``months: 1-3, 10-12`` for Jan, Feb, Mar, Oct, Nov and Dec)
* ``days``: limit based on days of the month (e.g.
  ``days: 1-15, 22`` for the first half of the month + the 22nd)
* ``weeks``: limit based on the weeks of the year
* ``weekdays``: limit based on the days of the week, from 1 (Monday)
  to 7 (Sunday)
* ``start_date``: A date of the form ``{ year: 2018, month: 2, day: 3 }``
  before which the rule should not be considered. Any of the three fields
  may be omitted, in which case the particular field is populated with
  the current date at validation time.
  If an invalid date such as ``{ year: 2018, month: 2, day: 29 }`` is
  provided, the next valid date (namely 2018-03-01 in this case) is
  assumed.
* ``end_date``: A date of the form ``{ year: 2018, month: 2, day: 3 }``
  after which the rule should not be considered anymore. As with
  ``start_date``, any of the three fields may be omitted.
  If an invalid date such as ``{ year: 2018, month: 2, day: 29 }`` is
  provided, the nearest prior valid date (namely 2018-02-28 in this
  case) is assumed.

The format used to specify values for the first five types of constraints
is as follows. We call it range strings, and only integers are supported,
no decimal values.

* ``x-y`` where ``x < y``: range of numbers from ``x`` to ``y``,
  including ``x`` and ``y``
* ``a,b``: numbers ``a`` and ``b``
* ``a,b,x-y``: the previous two together
* ... and so on
* Any spaces are ignored.

All constraints you define need to be fulfilled for the rule to match.


Rules with Sub-Schedules
------------------------

Imagine you need to turn on heating three times a day for one hour,
but only on working days from January to April. The obvious way of doing
this is to define four rules:

::

    schedule:
    - { temp: 23, start: "06:00", end: "07:00", months: "1-4", weekdays: "1-5" }
    - { temp: 20, start: "11:30", end: "12:30", months: "1-4", weekdays: "1-5" }
    - { temp: 20, start: "18:00", end: "19:00", months: "1-4", weekdays: "1-5" }
    - { temp: "OFF" }

But what if you want to extend the schedule to heat on Saturdays as
well? You'd end up changing this at three different places.

The more elegant way involves so-called sub-schedule rules. Look at this:

::

    schedule:
    - months: 1-4
      weekdays: 1-6
      rules:
      - { temp: 23, start: "06:00", end: "07:00" }
      - { temp: 20, start: "11:30", end: "12:30" }
      - { temp: 20, start: "18:00", end: "19:00" }
    - temp: "OFF"

The first, outer rule containing the ``rules`` parameter isn't considered
for evaluation itself. Instead, it's child rules - those defined under
``rules:`` - are considered, but only when the constraints of the parent
rule (``months`` and ``weekdays`` in this case) are fulfilled.

We can go even further and move the ``temp: 20`` one level up, so that
it counts for all child rules which don't have their own ``temp`` defined.

::

    schedule:
    - temp: 20
      months: 1-4
      weekdays: 1-6
      rules:
      - { start: "06:00", end: "07:00", temp: 23 }
      - { start: "11:30", end: "12:30" }
      - { start: "18:00", end: "19:00" }
    - temp: "OFF"

Note how the ``temp`` value for a rule is chosen. To find the value to
use for a particular rule, the rule is first considered itself. In case
it has no ``temp`` defined, all sub-schedule rules that led to this rule
are then scanned for a temperature value until one is found. When looking
at the indentation of the YAML, this lookup is done from right to left.

I've to admit that this was a small and well arranged example, but the
benefit becomes clearer when you start to write longer schedules, maybe
with separate sections for the different seasons.

With this knowledge, writing quite powerful Heaty schedules should be
easy and quick.

The next chapter deals with temperature expressions, which finally
give you the power to do whatever you can do with Python, right inside
your schedules.
