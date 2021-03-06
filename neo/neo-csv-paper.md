# CSV

*Sven Van Caekenberghe*

*June 2012*

*(This is a draft)*


CSV (Comma Separated Values) is a popular data-interchange format.
NeoCSV is an elegant and efficient standalone Smalltalk framework to 
read and write CSV converting to or from Smalltalk objects.   


## An introduction to CSV


CSV is a lightweight text-based de facto standard for human-readable tabular data interchange.
Essentially, the key characteristics are that CSV (or more generally, delimiter separated text data):

- is text based (ASCII, Latin1, Unicode)
- consists of records, 1 per line (any line ending convention)
- where records consist of fields separated by a delimiter (comma, tab, semicolon)
- where every record has the same number of fields
- where fields can be quoted should they contain separators or line endings

Here are some relevant links:

- <http://en.wikipedia.org/wiki/Comma-separated_values>
- <http://tools.ietf.org/html/rfc4180>

Note that there is not one single offical standard specification.


## NeoCSV


The NeoCSV framework contains a reader (NeoCSVReader) and a writer (NeoCSVWriter)
to parse respectively generate delimiter separated text data to or from Smalltalk objects.
The goals of this project are:

- to be standalone (have no dependencies and have little requirements)
- to be small, elegant and understandable
- to be efficient (both in time and space)
- to be flexible and non-intrusive

To use either the reader or the writer, 
you instanciate them on a character stream and use standard stream access messages.
Here are two examples:

    (NeoCSVReader on: '1,2,3\4,5,6\7,8,9' withCRs readStream) upToEnd.

    String streamContents: [ :stream |
      (NeoCSVWriter on: stream) 
        nextPutAll: #( (x y z) (10 20 30) (40 50 60) (70 80 90) ) ].


## Generic Mode


NeoCSV can operate in generic mode without any further customization.
While writing,

- record objects should respond to the #do: protocol
- fields are always send #asString and quoted
- CRLF line ending is used

While reading,

- records become arrays
- fields remain strings
- any line ending is accepted
- both quoted and unquoted fields are allowed

The standard delimiter is a comma character.
Quoting is always done using a double quote character.
A double quote character inside a field will be escaped by repeating it.
Field separators and line endings are allowed inside a quoted field.
All whitespace is significant. 


## Customizing NeoCSVWriter


Any character can be used as field separator, for example:

    neoCSVWriter separator: Character tab

or

    neoCSVWriter separator: $;

Likewise, any of the 3 common line end conventions can be set:

    neoCSVWriter lineEndConvention: #cr

There are 3 ways a field can be written (in increasing order of efficiency):

- quoted - converting it with #asString and quoting it (the default)
- raw - converting it with #asString but not quoting it
- object - not quoting it and using #printOn: directly on the output stream 

Obviously, when disabling qouting, you have to be sure your values do not contain embedded separators or line endings.
If you are writing arrays of numbers for example, this would be the fastest way to do it:

    neoCSVWriter
      fieldWriter: #object;
      nextPutAll: #( (100 200 300) (400 500 600) (700 800 900) )

The fieldWriter option applies to all fields.

If your data is in the form of regular domain level objects it would be wasteful 
to convert them to arrays just for writing them as CSV.
NeoCSV has a non-intrusive option to map your domain object's fields.
You add field specifications based on accessors.
This is how you would write an array of Points.

    neoCSVWriter
      nextPut: #(x y);
      addFields: #(x y);
      nextPutAll: { 1@2. 3@4. 5@6 }

Note how we first write the header (before customizing the writer).
The addField: and addFields: methods arrange for the specified accessors 
to be performed on the incoming objects to produce values that will be written by the fieldWriter.
Additionally, there is protocol to specify different field writing behavior per field,
using addQuotedField:, addRawField: and addObjectField:.
To specify different field writers for an array (actually an SequenceableCollection subclass), 
you can use the methods first, second, third, ... as accessors.


## Customizing NeoCSVReader


The parser is flexible and forgiving.
Any line ending will do, quoted and non-quoted fields are allowed. 

Any character can be used as field separator, for example:

    neoCSVReader separator: Character tab

or

    neoCSVReader separator: $;

NeoCSVReader will produce records that are instances of its recordClass, which defaults to Array.
All fields are always read as Strings.
If you want, you can specify converters for each field, to convert them to integers or floats,
any other object. Here is an example:

    neoCSVReader
      addIntegerField;
      addFloatField;
      addField;
      addFieldConverter: [ :string | Date fromString: string ];
      upToEnd.

Here we specify 4 fields: an integer, a float, a string and a date field.
Field convertions specified this way only work on indexable record classes, like Array.

In many cases you will probably want your data to be returned as one of your domain objects.
It would be wasteful to first create arrays and then convert all those.
NeoCSV has non-intrusive options to create instances of your own object classes and 
to convert and set fields on them directly.
This is done by specifying accessors and converters.
Here is an example for reading Associations of Floats.

    (NeoCSVReader on: '1.5,2.2\4.5,6\7.8,9.1' withCRs readStream)
      recordClass: Association;
      addFloatField: #key: ;
      addFloatField: #value: ; 
      upToEnd.

For each field you give the mutator accessor to use, as well as an implicit or explicit conversion block.

One thing that it will enforce is that all records have an equal number of fields.
When there are no field accessors or converters, the field count will be set automatically after the first record is read.
If you want you could set it upfront.
When there are field accessors or converters, the field count will be set to the number of specified fields.
