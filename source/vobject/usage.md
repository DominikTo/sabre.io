---
title: Usage instructions
product: vobject
layout: default
---

Getting started
---------------

Make sure you followed all the steps in the [installation instructions][1],
and you've included the autoloader in your code.

A basic parsing example
-----------------------

Assuming there is a vcard in your local directory, named `cowboyhenk.vcf`,
the following complete example will parse it and display the `FN` property:

    use Sabre\VObject;

    include 'vendor/autoload.php';

    $vcard = VObject\Reader::read('cowboyhenk.vcf');
    echo $vcard->FN;

That is all.

For all the following examples, we are assuming that you have already included
the autoloader, and also that you've called

    use Sabre\VObject;

At the top of your script. This is just to avoid repetition.


Also, keep in mind that while sabre/vobject does support PHP 5.3, most of the
examples in this documentation use syntax that was introduced in PHP 5.4.

Specifically, we specify arrays as such:

    [
        "ORG" => "Henk Inc.",
        "FN" => "Cowboy Henk",
    ]

In PHP 5.3, this is specified as:

    array(
        "ORG" => "Henk Inc.",
        "FN" => "Cowboy Henk",
    )

The actual manual
-----------------

### Creating vCards.

To create a vCard, you can simply instantiate the vCard component, and pass
the properties you need:

    $vcard = new VObject\Component\VCard([
        'FN'  => 'Cowboy Henk',
        'TEL' => '+1 555 34567 455',
        'N'   => ['Henk', 'Cowboy', '', 'Dr.', 'MD'],
    ]);

    echo $vcard->serialize();

This will output:

    BEGIN:VCARD
    VERSION:3.0
    PRODID:-//Sabre//Sabre VObject 3.0.0-alpha5//EN
    FN:Cowboy Henk
    TEL:+1 555 34567 455
    N:Henk;Cowboy;;Dr.;MD
    END:VCARD

### Adding properties

Certain properties, such as `TEL`, `ADR` or `EMAIL` may appear more than once.
To add any additional properties, use the `add()` method on the vCard.

    $vcard->add('TEL', '+1 555 34567 456', ['type' => 'fax']);

The third argument of the add() method allows you to specify a list of
parameters.

### Manipulating properties

The vCard also allows object-access to manipulate properties:

    // Overwrites or sets a property:
    $vcard->FN = 'Doctor McNinja';

    // Removes a property
    unset($vcard->FN);

    // Checks for existence of a property:

    isset($vcard->FN);

### Working with parameters

To get access to a parameter, you can simply use array-access:

    $type = $vcard->TEL['TYPE']:
    echo (string)$type;

Parameters can also appear multiple times. To get to their values, just loop
through them:

    if ($param = $vcard->TEL['TYPE']) {
        foreach($param as $value) {
          echo $value, "\n";
        }
    }

To change parameters for properties, you can use array-access syntax:

    $vcard->TEL['TYPE'] = ['WORK','FAX']:

Or when you're working with singular parameters:

    $vcard->TEL['PREF'] = 1;

It is also possible add a list of parameters while creating the property.

    $vcard->add(
        'EMAIL',
        'foo@example.org',
        [
            'type' => ['home', 'work'],
            'pref' => 1,
        ]
    );

### Parsing vCard or iCalendar

To parse a vCard or iCalendar object, simply call:

    // $data must be either a string, or a stream.
    $vcard = VObject\Reader::read($data);

When you're working with vCards generated by broken software (such as the
latest Microsoft Outlook), you can pass a 'forgiving' option that will do an
attempt to mend the broken data.

    $vcard = VObject\Reader::read($data, VObject\Reader::OPTION_FORGIVING);

### Reading property values

For properties that are stored as a string, you can simply call:

    echo (string)$vcard->FN;

For properties that contain more than 1 part, such as `ADR`, `N` or `ORG` you
can call `getParts()`.

    print_r(
        $vcard->ORG->getParts();
    );

### Looping through properties.

Properties such as `ADR`, `EMAIL` and `TEL` may appear more than once in a
vCard. To loop through them, you can simply throw them in a `foreach()`
statement:

    foreach($vcard->TEL as $tel) {
        echo "Phone number: ", (string)$tel, "\n";
    }

    foreach($vcard->ADR as $adr) {
        print_r($adr->getParts());
    }

### vCard property grouping

It's allowed in vCards to group multiple properties together with an arbitrary
string.

Apple clients use this feature to assign custom labels to things like phone
numbers and email addresses. Below is an example:

    BEGIN:VCARD
    VERSION:3.0
    groupname.TEL:+1 555 12342567
    groupname.X-ABLABEL:UK number
    END:VCARD

In our example, you can see that the TEL properties are prefixed. These are 'groups' and
allow you to group multiple related properties together.

In most situations these group names are ignored, so when you execute the following
example, the `TEL` properties are still traversed.

    foreach($vcard->TEL as $tel) {
        echo (string)$tel, "\n";
    }

But if you would like to target a specific group + property, this is possible too:

    echo (string)$vcard->{'groupname.TEL'};

To expand that example a little bit; if you'd like to traverse through all phone
numbers and display their custom labels, you'd do something like this:


    foreach($vcard->TEL as $tel) {

        echo (string)$vcard->{$tel->group . '.X-ABLABEL'}, ": ";
        echo (string)$tel, "\n";

    }

### iCalendar

iCalendar works much the same way as vCards, but has a couple of features
that vCard does not.

First, in vCard there's only 1 component (everything between `BEGIN:VCARD`
and `END:VCARD`), but in iCalendar, there are nested components.

A simple illustration, lets create an iCalendar that contains an event.

    $vcalendar = new VObject\Component\VCalendar();

    $vcalendar->add('VEVENT', [
        'SUMMARY' => 'Birthday party',
        'DTSTART' => new \DateTime('2013-04-07'),
        'RRULE' => 'FREQ=YEARLY',
    ]);

    echo $vcalendar->serialize();

This will output the following:

    BEGIN:VCALENDAR
    VERSION:2.0
    PRODID:-//Sabre//Sabre VObject 3.0.0-alpha5//EN
    CALSCALE:GREGORIAN
    BEGIN:VEVENT
    SUMMARY:Birthday party
    DTSTART;TZID=Europe/London;VALUE=DATE-TIME:20130407T000000
    RRULE:FREQ=YEARLY
    END:VEVENT
    END:VCALENDAR

The add() method will always return the instance of the property or
sub-component it's creating. This makes for easy further manipulation.

Here's another example that adds an attendee and an organizer:

    $vcalendar = new VObject\Component\VCalendar();

    $vevent = $vcalendar->add('VEVENT', [
        'SUMMARY' => 'Meeting',
        'DTSTART' => new \DateTime('2013-04-07'),
    ]);

    $vevent->add('ORGANIZER','mailto:organizer@example.org');
    $vevent->add('ATTENDEE','mailto:attendee1@example.org');
    $vevent->add('ATTENDEE','mailto:attendee2@example.org');

### Date and time handling

Parsing Dates and Times from iCalendar and vCard can be difficult.
Most of this is abstracted by the VObject library.

Given an event, in a calendar, you can get a real PHP `DateTime` object using
the following syntax:

    $start = $vcalendar->VEVENT->DTSTART->getDateTime();
    echo $start->format(\DateTime::W3C);

To update the property with a new `DateTime` object, just use the following syntax:

    $dateTime = new \DateTime('2012-08-07 23:53:00', new \DateTimeZone('Europe/Amsterdam'));
    $event->DTSTART = $dateTime;


### Free-busy report generation

Some calendaring software can make use of FREEBUSY reports to show when people are
available.

You can automatically generate these reports from calendars using the `FreeBusyGenerator`.

Example based on our last event:

    // We're giving it the calendar object. It's also possible to specify multiple objects,
    // by setting them as an array.
    //
    // We must also specify a start and end date, because recurring events are expanded.
    $fbGenerator = new VObject\FreeBusyGenerator(
        new DateTime('2012-01-01'),
        new DateTime('2012-12-31'),
        $vcalendar
    );

    // Grabbing the report
    $freebusy = $fbGenerator->getResult();

    // The freebusy report is another VCALENDAR object, so we can serialize it as usual:
    echo $freebusy->serialize();

The output of this script will look like this:

    BEGIN:VCALENDAR
    VERSION:2.0
    PRODID:-//Sabre//Sabre VObject 2.0//EN
    CALSCALE:GREGORIAN
    BEGIN:VFREEBUSY
    DTSTART;VALUE=DATE-TIME:20111231T230000Z
    DTEND;VALUE=DATE-TIME:20111231T230000Z
    DTSTAMP;VALUE=DATE-TIME:20120808T131628Z
    FREEBUSY;FBTYPE=BUSY:20120109T140000Z/20120109T140000Z
    FREEBUSY;FBTYPE=BUSY:20120213T140000Z/20120213T140000Z
    FREEBUSY;FBTYPE=BUSY:20120312T140000Z/20120312T140000Z
    FREEBUSY;FBTYPE=BUSY:20120409T140000Z/20120409T140000Z
    FREEBUSY;FBTYPE=BUSY:20120514T140000Z/20120514T140000Z
    FREEBUSY;FBTYPE=BUSY:20120611T140000Z/20120611T140000Z
    FREEBUSY;FBTYPE=BUSY:20120709T140000Z/20120709T140000Z
    FREEBUSY;FBTYPE=BUSY:20120813T140000Z/20120813T140000Z
    FREEBUSY;FBTYPE=BUSY:20120910T140000Z/20120910T140000Z
    FREEBUSY;FBTYPE=BUSY:20121008T140000Z/20121008T140000Z
    FREEBUSY;FBTYPE=BUSY:20121112T140000Z/20121112T140000Z
    FREEBUSY;FBTYPE=BUSY:20121210T140000Z/20121210T140000Z
    END:VFREEBUSY
    END:VCALENDAR

### Splitting export files

Generally when software makes backups of calendars or contacts, they will
put all the objects in a single file. In the case of vCards, this is often
a stream of VCARD objects, in the case of iCalendar, this tends to be a
single VCALENDAR objects, with many components.

Protocols such as Card- and CalDAV expect only 1 object per resource. The
vobject library provides 2 classes to split these backup files up into many.

To do this, use the splitter objects:

    // You can either pass a readable stream, or a string.
    $h = fopen('backupfile.vcf', 'r');
    $splitter = new VObject\Splitter\VCard($h);

    while($vcard = $splitter->next()) {

        // $vCard is a single vCard object. You can just call serialize() on it
        // if you were looking for the string version.

    }

Next to the VCard splitter, there's also an ICalendar splitter. The latter
creates a `VCALENDAR` object per `VEVENT`, `VTODO` or `VJOURNAL`, and ensures
that the `VTIMEZONE` information is kept intact, and any `VEVENT` objects that
belong together (because they are expections for an `RRULE` and thus have the
same `UID`) will be kept together, exactly like CalDAV expects.

### Converting between different vCard versions.

Since sabre/vobject 3.1, there's also a feature to convert between various
vCard versions. Currently it's possible to convert from vCard 2.1, 3.0 and
4.0 and to 3.0 and 4.0. It's not yet possible to convert to vCard 2.1.

To do this, simply call the convert() method on the vCard object.

    $input = <<<VCARD
    BEGIN:VCARD
    VERSION:2.1
    FN;CHARSET=UTF-8:Foo
    TEL;PREF;HOME:+1 555 555 555
    END:VCARD
    VCARD;

    $vCard = VObject\Reader::read($input);
    $vCard->convert(VObject\Document::VCARD40);

    echo $vcard->serialize();

    // This will output:
    /*
    BEGIN:VCARD
    VERSION:4.0
    FN:Foo
    TEL;PREF=1;TYPE=HOME:+1 555 555 555
    END:VCARD
    */

Note that not everything can cleanly convert between versions, and it's
probable that there's a few properties that could be converted between
versions, but isn't yet. If you find something, open a feature request ticket
on Github.

CLI tool
--------

Since vObject 3.1, a new cli tool is shipped in the bin/ directory.

This tool has the following features:

* A `validate` command.
* A `repair` command to repair objects that are slightly broken.
* A `color` command, to show an iCalendar object or vCard on the console with
  ansi-colors, which may help debugging.
* A `convert` command, allowing you to convert between iCalendar 2.0, vCard 2.1,
  vCard 3.0, vCard 4.0, jCard and jCal.

Just run it using `bin/vobject`. Composer will automatically also put a
symlink in `vendor/bin` as well, or a directory of your choosing if you set
the `bin-dir` setting in your composer.json.

Support
-------

Head over to the [SabreDAV mailing list](http://groups.google.com/group/sabredav-discuss) for any questions.

[1]: /vobject/install

