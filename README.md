# Nanoflakes Specification

This is the official specification for Nanoflakes, a unique identifier
format and generation algorithm based on Twitter's Snowflake algorithm.

This specification is licensed under the [CCO 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)
license.

## A Nanoflake

A nanoflake is a 64-bit signed integer which is generated locally or
remotely by a nanoflake server.  The nanoflake is composed of the following
information:

* 41-bit timestamp
* 10-bit generator ID
* 12-bit sequence number

Additionally, the epoch for a Nanoflake is defined by the context in which
it is generated and used.  That means that users are free to define their
own epoch for their own applications, and use said epoch across all generators
in order to ensure that nanoflakes are globally unique in their application.

Since nanoflakes contain a timestamp, they can be sorted by time, and
therefore can be used as a unique ID for a time series. A user can also
use the nanoflake's timestamp to determine when the nanoflake was generated.
This is done by shifting the timestamp 22 bits to the right, and adding the
epoch to the result in order to get the nanoflake's timestamp in milliseconds
since the UNIX epoch.

## Ensuring Global Uniqueness

In order to ensure that nanoflakes are globally unique, the following
conditions must be met:

* Each nanoflake generator for a given system or application must have its own,
  unique generator ID (10-bit integer) for the entire application context.
  * The generator ID can be any number between 0 and 1023 (inclusive).
* All nanoflake generators for a given system or application must be configured 
  to use a common known epoch for the entire application context.
  * The epoch is a UNIX timestamp in milliseconds. The only requirement is that
    the epoch must represent a time in the past relative to the generator's current
    time.

The user is responsible for ensuring that these conditions are met, by
specifying a common epoch across all generators, and by ensuring that each
generator has a unique generator ID.

If these conditions are met then a nanoflake is guaranteed to be globally
unique within the context of the application.

* The sequence number is incremented for each nanoflake generated by a given
  generator, and if the sequence number overflows, the generator will wait
  until the next millisecond to generate a new nanoflake.
* The generator ID is unique to the generator, and is used to ensure that
  nanoflakes can be generated by multiple generators within the same system
  without colliding or requiring coordination between the generators.
* The timestamp is the number of milliseconds since the epoch, and allows
  nanoflakes to be sorted by time of generation, as well as to enable generators
  to be restarted without requiring persistence of a sequence number.

## Compatibility with Twitter Snowflakes

Nanoflakes are compatible with Twitter Snowflakes, and can be used as a
drop-in replacement for Snowflakes in most cases.  The only difference
between a nanoflake and a Snowflake is that nanoflakes merges Twitter's
5-bit `datacenter` ID and 5-bit `worker` ID into a single 10-bit `generator`
ID. This allows nanoflakes to support 1024 unique generators instead of
32 generators per datacenter, with 32 datacenters per application context.

## Generating of Nanoflakes

A generator is responsible for generating nanoflakes.  This specification
does not specify how or where a generator should be implemented.  The generator
can be implemented as a standalone application, as a microservice, or as a
simple instance of a class in a larger application. As long as the generator
is able to generate nanoflakes, it is considered a valid generator.

In order to generate a nanoflake, a generator must:

* Get the current timestamp in milliseconds since the epoch.
* Check if the system clock did not go backwards.
  * If the system clock did go backwards, the generator must either wait until the
    system clock catches up to the current time, or throw an exception.
* Reset the sequence number if the timestamp has changed since the last nanoflake
  was generated.
* Increment the sequence number, checking for a 12-bit overflow.
  * If the sequence number overflows, the generator must either wait until the
    next millisecond, or throw an exception.
* Generate the nanoflake by combining the timestamp, generator ID, and sequence.
  * The nanoflake is generated by shifting the timestamp 22 bits to the left,
    shifting the generator ID 12 bits to the left, and ORing the sequence number,
    the timestamp and generator ID together.

Reference implementations of nanoflake generators are available in the
[official organization](https://github.com/nanoflakes).

## Encoding of Nanoflakes

Nanoflakes are encoded as 64-bit signed integers, and can be encoded in
any format that supports 64-bit signed integers.

This specification suggests either saving nanoflakes directly as 64-bit signed
integers if possible, or encoding them as strings, since some programming
languages, such as JavaScript, [do not (fully) support 64-bit signed integers.](https://tqdev.com/2016-javascript-cannot-handle-64-bit-integers)

If nanoflakes are to be encoded as strings, this specification suggests
encoding nanoflakes using base-36, in order to fit the largest possible
nanoflake into a 13-character string. The base-36 encoding is not required,
and users are free to encode, decode, store, and transmit nanoflakes in any
format that supports Nanoflakes' underlying 64-bit signed integers.

## Limitations, Caveats, and Considerations

* The 41-bit timestamp limits the nanoflake's lifespan to 69 years.
* The 10-bit generator ID limits the number of generators to 1024.
* The 12-bit sequence number limits the number of nanoflakes that can be
  generated per millisecond to 4096.
* Since the user is responsible for ensuring that generators have unique
  10-bit IDs, misconfiguration of said generator ID can result in collisions 
  between nanoflakes generated by different generators.
* Since the user is responsible for ensuring that generators use a common
  epoch, misconfiguration of said epoch can result in nanoflakes seemingly
  looking like they were generated in the future.
* As generation relies on the system clock, which is not guaranteed to be
  monotonic depending on the operating system and hardware, the generator
  might be forced to wait until the system clock catches up to the last
  nanoflake's timestamp, which can be a long time if the system clock was
  set backwards by a large amount, and possibly result in a disruption of
  service as one or more generators are forced to wait.

## License

This specification is licensed under the [CCO 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/) license.
