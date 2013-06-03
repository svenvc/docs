# ZTimestamp

I am ZTimestamp.
I am a Magnitude.
I represent a point in time, a combination of a date and a time.

I am an alternative for DateAndTime and TimeStamp.
I have second precision and live in the UTC/GMT/Zulu timezone.
I use ISO/International conventions and protocols only. 
I support some essential arithmetic.

I have an efficient internal representation:

  jnd - the julian day number <SmallInteger>
	secs - the number of seconds since midnight, the beginning of the day <SmallInteger>

Examples:

	ZTimestamp now.
	ZTimestamp fromString: '1969-07-20T20:17:40Z'.

I am somewhat compatible with existing, standard Chronology objects.
I correctly parse representations with a timezone designator
and can print a representation in arbitrary timezones. 

	(ZTimestamp fromString: DateAndTime now truncated printString) localPrintString.
  
