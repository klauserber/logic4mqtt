logic4mqtt
==========

  Written and (C) 2015-16 Oliver Wagner <owagner@tellerulam.com> 
  
  Provided under the terms of the MIT license.


Overview
--------
logic4mqtt is a logic and scripting engine for SmartHome automation, based around MQTT as a central
message bus.

It uses Java's generalized scripting interface (JSR-223) so scripts can be implemented in any
script language supported by this interface. By default, the JVM ships with a Javascript scripting
engine (Rhino with Java 7, Nashorn with Java 8), but a variety of other interfaces is available
for languages like Groovy, Jython and others.


Dependencies
------------
* Java 1.8 (or higher) SE Runtime Environment: https://www.java.com/
* Eclipse Paho: https://www.eclipse.org/paho/clients/java/ (used for MQTT communication)
* Minimal-JSON: https://github.com/ralfstx/minimal-json (used for JSON creation and parsing)
* natty: http://natty.joestelmach.com/ (used for natural language time parsing)
* Quartz Scheduler: http://www.quartz-scheduler.org/ (used for cron-alike timer parsing)
* Java Mail: https://javamail.java.net/ (used for sending E-Mails)

[![Build Status](https://travis-ci.org/owagner/logic4mqtt.svg)](https://travis-ci.org/owagner/logic4mqtt) Automatically built jars can be downloaded from the release page on GitHub at https://github.com/owagner/logic4mqtt/releases


API
---
logic4mqtt provides a scripting host and a support API which provides

* MQTT access
* Event handling, based on incoming MQTT messages
* versatile Timer support with both Cron-alike and natural language syntax
* support for Sunrise/Sunset calculations, also tied into timer support
* utility functions for network access, Wake-On-Lan etc.

The API is organized in classes. Documentation for the classes is available at
http://owagner.github.io/logic4mqtt/apidocs/


Topic notation
--------------
Various logic4mqtt functions deal with MQTT topics. To simplify topic usage in accordance with the MQTT-smarthome specification, 
there are some special constructs which are replaced depending on contexts:

  - When setting values, a "//" gets replaced with "/set/"
  - When getting values, a "//" gets replaced with "/status/"
  - A "$" prefix gets replaced with the configured logic4mqtt own topic prefix, with "/status/" added.
    This is intended to faciliate global state variables. 

Example:

	Events.setValue("knx//Floor/Livingroom/Light One",true)
	
publishes the value "1" to `knx/set/Floor/Livingroom/Light One`
whereas 

	Events.getValue("knx//Floor/Livingroom/Light One")
	
returns the value of the topic `knx/status/Floor/Livingroom/Light One`

	Events.storeValue("$Status Lighting",1)

and

	Events.getValue("$Status Lighting")
	
however both reference `logic/status/Stauts Lighting`


Events
------
Everything in logic4mqtt revolves around the core concept of an _Event_


Timers
------
logic4mqtt timers can be specified in one of two ways:

1. cron-alike syntax
2. natural language

### Cron-alike syntax ###

Cron syntax is parsed using the Quartz Scheduler CronExpression handling, which is described in 
detail at http://quartz-scheduler.org/api/2.2.0/org/quartz/CronExpression.html

Two important differences to UNIX-like cron specifications:

- the first field is a 'seconds' field, so the total number of fields is at least 6. UNIX-like crons
often only have minute resolution, and thus only 5 fields.
- it's not possible to specify both Day-Of-Month and Day-Of-Week as "*". Specify the Day-Of-Week 
or Day-Of-Month field as "?" if you don't care for that specific field.

Examples:

    Timers.addTimer("every_5_minutes","0 */5 * * * ?",callback);

### Natural language syntax ###

The natural language syntax utilizies the Natty library http://natty.joestelmach.com/ for parsing time 
specifications, with additional support for specifying sunset/sundown as a reference point, and
randomization.

Examples:

    Timers.addTimer("test1","in 5 minutes",callback);
    Timers.addTimer("test2","10 minutes before sunset",callback);
    Timers.addTimer("test3","1 hour after civil sunrise",callback);
    
Repeating timers are also specified using this syntax:

    Timers.addTimer("test4","every 10 minutes before sunset",callback);

The natty homepage has a test functionality at http://natty.joestelmach.com/try.jsp where you can
try possible specifications. The sunset/sundown keywords are simply replaced with the currently calculated
times before the string is passed to natty for parsing. For example, 

    1 hour after civil sunrise
    
becomes

    1 hour after 08:40
    
before it's passed to natty. The specifications are reparsed everytime the timer is run, to make sure
relative specifications always refer to the current time. This also applies to the sunset/sundown
replacement keywords. The sunset/sundown keywords may optionally be prefixed with the keywords "civil", "nautical", 
"astronomical" or "official", to refer to variants of the sunset/sundown times. The latitude and longitude
of the currently location need to be set with a call to

    Time.setLocation(latitude,longitude)
    
prior to use.

Additionally, a randomization amount can be specified:

    every 11:00 randomize by 30 minutes

The unit specification is optional and can be any of "second(s)", "minute(s)" or "hour(s)". If it is missing,
minutes are assumed. The "by" keyword is entirely optional.


Changelog
---------
* 0.30 - 2016/02/29 - owagner
  - API: added new option "ignoreMissingFiles" to mail properties

* 0.29 - 2016/02/27 - owagner
  - API: added Events.getLastChangeTimestamp()
  - updated quartz-scheduler to 2.2.2 and javax.mail to 1.5.5

* 0.28 - 2015/10/21 - owagner
  - mail sending was broken when logic4mqtt was deployed as a fat jar due to missing handler map
    from mail.jar

* 0.27 - 2015/09/27 - owagner
  - API: add Events.linkValue()
  - updated minimal-json to 0.9.4, natty to 0.12 and javax.mail to 1.5.4
  
* 0.26 - 2015/09/15 - owagner
  - API: Utilities.sendNetMessage() now accepts an optional encoding parameter
  - exit on any exception when executing a script during startup, not only on ScriptException.
    Previously, normal API-level exceptions (like IllegalArgumentException) could be thrown
    and would be logged, but ignored, thus hiding script bugs

* 0.25 - 2015/07/19 - owagner
  - updated to eclipse-paho 1.0.2
  - ignore intermediate syslog errors

* 0.24 - 2015/06/22 - owagner
  - API: extended callback signatures to receive the full JSON encoded object from MQTT as a fifth parameter,
    including all meta-fields
  - API: make event handling deal with messages which are a JSON array or an JSON object without "val" field.
    In this case, the simple value will be the complete object as well, similar to the "full value".
  - API: added "priority" to mail properties
  - API: added Timers.rateLimit()

* 0.23 - 2015/06/14 - owagner
  - no longer support Java 7 (EOLed anyway, and maintaining Rhino compatibility is tricky)
  - API: added support for sending E-Mails via the "Mail" utility object. Uses Java Mail.  

* 0.22 - 2015/06/14 - owagner
  - API: more shuffling with method signatures for setValue() and JSON encoding code to work properly with 
    both Rhino and Nashorn in case of nested arrays

* 0.21 - 2015/06/07 - owagner
  - API: actually changed Events.setValue() and storeValue() array signatures to accept Object[] instead
    of Collection<?>, as the latter caused ambiguity with passing plain objects

* 0.20 - 2015/06/06 - owagner
  - API: added Events.getValues()
  - API: support Events.setValue() and Events.storeValue() with arrays as values
  - API: added "initial" parameter to Events.add() to force checking a callback once
    with the initial value

* 0.19 - 2015/05/13 - owagner
  - API: fix isDaylight(...) checks for the case that the end of a twilight is past midnight

* 0.18 - 2015/05/03 - owagner
  - support randomization in  Natty timers with a "randomize by <amount> seconds/minutes/hours"
    construct. The "by" keyword and the unit specification are optional; if no unit is
    specified, "minutes" are assumed.
  - updated Natty dependency to 0.10.1
  - properly set script filename in ScriptContext when initially executing scripts, so error
    messages generated by Rhino/Nashorn have the proper filename

* 0.17 - 2015/04/20 - owagner
  - replaced libnova with custom routines; fixed sun azimuth calculations, and improved caching
  - API: fixed initial scheduling of solar event related timers; since 0.16, the first scheduling
    could unintentionally be moved to the next day.

* 0.16 - 2015/04/20 - owagner
  - API: Fixed a bug in natty timers with sunset-relative specifications -- since sunset/sunrise
    times change upon rescheduling, it was possible that the next scheduled run was just a few minutes
    after the last one. This made the check to make sure that timers are rerun on the next day
    ineffective, and could cause such timers to trigger twice a day.
  
* 0.15 - 2015/04/13 - owagner
  - API: added Time.getSunAltitude() and Time.getSunAzimuth()

* 0.14 - 2015/04/13 - owagner
  - API: added cache for sunrise/sunset calculations

* 0.13 - 2015/04/12 - owagner
  - API: onChange logic was completely broken and would retrigger
    on every update, or not at all
  - API: fixed Time.is(xx)Daylight(), which got broken in libnova migration
  
* 0.12 - 2015/03/30 - owagner
  - replaced sunrisesunsetlib-java with libnova/novaforjava
  - API: corrected Time.isOfficial() to Time.isOfficialDaylight()

* 0.11 - 2015/03/09 - owagner
  - API: it's now possible to pass Javascript objects to setValue() et.al., which will then be converted
    to JSON before publishing

* 0.10 - 2015/03/07 - owagner
  - API: added Events.requestValue() for MQTT /get/ requests
  - API: topic patterns are now actually RegExes
  - API: topics passed into callbacks now have a "/status/" function prefix replaced by "//",
    so the topics can immediately be used for setValue() calls
  - CmdLine: added "EVENTS" command to diagnose active event handlers
  - CmdLine: added "PARSETIME" command to test natural language time specifications
  - CmdLine: "TIMERS" which are queuedValue()s now have a more readable callback output 

* 0.9 - 2015/03/04 - owagner
  - added a Event.add() call which adds a event handler callback with extended parameters
  - added support for one-shot event handlers. Only available via Event.add()
  - added support for event handlers with expiration. Only available via Event.add(). Expiration
    is specified in standard timer notation.
  - added some Natty test cases
   
* 0.8 - 2015/02/26 - owagner
  - added a TCP command line interface, mainly intended for diagnostic purposes.
  Enable by specifying logic4mqtt.cmdline.port=<TCP port number to listen on>
  
* 0.7 - 2015/02/23 - owagner
  - API: renamed the SunriseSunset object to "Time"
  - API: added Time.isBefore(), Time.isAfter() and Time.isBetween()
  - API: added Utilities.executeCommand(cmd)
  - API: added Events.storeValue() and Events.queueStore(), which complement .setValue() and .queueValue()
    respectivly, but cause the "retain" flag of the published message to be set. This is mainly intended to
    be used with global variables.
  - API: A "$" topic prefix gets replaced with the configured logic4mqtt own topic prefix, with "/status/" added.
    This is intended to faciliate global state variables. 
  
* 0.6 - 2015/01/25 - owagner
  - adapted to new mqtt-smarthome topic hierarchy scheme: /status/ for reports, /set/ for setting values.
    A special notation is supported -- if the first topic part is suffixed with "//", it gets
    replaced with /set/ and /status/ depending on context. For example, an event handler set
    to "knx//kitchen/light" gets to be actually set to "knx/status/kitchen/light", whereas
    a Event.setValue("knx//kitchen/light"...) will set "knx/set/kitchen/light"
  - now skips common backup suffixes when looking for scripts to execute (right now, "~" and ".bak")
  - when publishing to MQTT, round numbers to 8 decimal digits, and try to avoid a decimal point
    alltogether for integers. Also, convert Booleans to 0/1.
  - API: SunriseSunset now has getSunrise() and getSunset() methods which accept a string denoting
    the required Zenith (OFFICIAL, ASTRONOMICAL, NAUTICAL and CIVIL)
  - API: added SunriseSunset.isDaylight() and variants to quickly check whether it's currently
    between sunrise and sunset for the given Zenith

* 0.5 - 2015/01/18 - owagner
  - API: added Utilites.sendNetMessage() function for doing very simple network communication
  - when running Rhino, explicitely set setJavaPrimitiveWrap(false). Otherwise, primitive types returned
    from Java methods like Events.getValue() end up being Javascript Object instead of primitive
    instances
  - reset log prefix to [callback] when initial script runs have finished

* 0.4 - 2015/01/08 - owagner
  - use one ScriptEngine instance per file suffix, so the complete context is shared among scripts
  - read version number from jar manifest or build.gradle
  - redirect script output to java logging, with the log messages being [prefixed] with the script
    name
  

