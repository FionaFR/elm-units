# elm-units

> Simple, safe and convenient unit types and conversions for Elm

*Note*: This package has not yet been published!

`elm-units` is useful if you want to store, pass around, convert between,
compare, or do arithmetic on:

  - Durations (seconds, milliseconds, hours...)
  - Angles (degrees, radians, turns...)
  - Lengths (meters, feet, inches, miles, light years...)
  - Temperatures (Celsius, Fahrenheit, kelvins)
  - Pixels (whole or partial)
  - Speeds (pixels per second, miles per hour...) or any other rate of change
  - Any of the other built-in quantity types: areas, accelerations, masses,
    forces, pressures, currents, voltages...
  - Or even values in your own custom units, such as 'number of tiles' in a
    tile-based game

It is aimed especially at engineering/scientific/technical appliations but is
designed to be generic enough to work well for other fields such as games and
finance. The core of the package consists of functions like

```elm
Length.meters : Float -> Length
Length.feet : Float -> Length
Duration.seconds : Float -> Duration
Duration.milliseconds : Float -> Duration

Length.inMeters : Length -> Float
Length.inFeet : Length -> Float
Duration.inSeconds : Duration -> Float
Duration.inMilliseconds : Duration -> Float
```

You can use these functions to do simple unit conversions:

```elm
Duration.hours 3 |> Duration.inSeconds
--> 10800

Length.feet 10 |> Length.inMeters
--> 3.048

Speed.milesPerHour 60 |> Speed.inMetersPerSecond
--> 26.8224

Temperature.degreesCelsius 30 |> Temperature.inDegreesFahrenheit
--> 86
```

Type-safe values like `Length`s and `Duration`s also work very well as as record
fields or function arguments:

```elm
import Angle exposing (Angle)
import Duration exposing (Duration)
import Temperature exposing (Temperature)

type alias Camera =
    { manufacturer : String
    , fieldOfView : Angle
    , shutterSpeed : Duration
    , minimumOperatingTemperature : Temperature
    }

camera : Camera
camera =
    { manufacturer = "Kodak"
    , fieldOfView = Angle.degrees 60
    , shutterSpeed = Duration.milliseconds 2.5
    , minimumOperatingTemperature = Temperature.celsius -35
    }

canOperateAt : Temperature -> Camera -> Bool
canOperateAt temperature camera =
    temperature |> Temperature.greaterThan camera.minimumOperatingTemperature

camera |> canOperateAt (Temperature.fahrenheit -10)
--> True

camera.fieldOfView |> Angle.inRadians
--> pi / 3
```

Finally, quantity types like `Length` are actually of type `Quantity number
units` (`Length` is `Quantity Float Meters`, for example), and there are several
generic functions which let you work directly with any kind of `Quantity`
values:

```elm
Duration.hours 2 |> Quantity.plus (Duration.minutes 30)
--> Duration.seconds 9000

Quantity.sort [ Length.feet 1, Length.inches 1, Length.meters 1 ]
--> [ Length.inches 1, Length.feet 1, Length.meters 1  ]

Duration.minutes 2
  |> Quantity.at (Speed.metersPerSecond 15)
  |> Length.inKilometers
--> 1.8
```

## Table of Contents

  - [Installation](#installation)
  - [Usage](#usage)
    - [Fundamentals](#fundamentals)
    - [The `Quantity` Type](#the-quantity-type)
    - [Arithmetic and Comparison](#arithmetic-and-comparison)
    - [Custom Functions](#custom-functions)
    - [Custom Units](#custom-units)
    - [Understanding Quantity Types](#understanding-quantity-types)
  - [Getting Help](#getting-help)
  - [API](#api)
  - [Contributing](#contributing)
  - [License](#license)

## Installation

Once this package is published, you will be able to install it with

```
elm install ianmackenzie/elm-units
```

## Usage

### Fundamentals

To take code that currently uses raw `Float` values and convert it to using
`elm-units` types, there are three basic steps:

  - Wherever you store a `Float`, such as in your model or in a message, switch
    to storing a `Duration` or `Angle` or `Temperature` etc. value instead.
  - Whenever you *have* a `Float` (from an external package, JSON decoder etc.),
    use a function such as `Duration.seconds`, `Angle.degrees` or
    `Temperature.fahrenheit` to turn it into a type-safe value.
  - Whenever you *need* a `Float` (to pass to an external package, encode as
    JSON etc.), use a function such as `Duration.inMillliseconds`,
    `Angle.inRadians` or `Temperature.inCelsius` to extract the value in
    whatever units you want.

### The `Quantity` Type

All values produced by this package (with the exception of `Temperature`, which
is a bit of a special case) are actually values of type `Quantity`, defined as

```elm
type Quantity number units
    = Quantity number
```

with some convenient type aliases

```elm
-- A fractional number of units, useful for general quantities like length
type alias Fractional units =
    Quantity Float units

-- A whole number of units, useful for exact values in cents/pixels
type alias Whole units =
    Quantity Int units
```

For example, `Length` is defined as

```elm
type alias Length =
    Fractional Meters
```

This means that a `Length` is internally stored as a `Float` number of `Meters`,
but this can mostly be treated as an implementation detail.

Having a common `Quantity` type means that it is possible to define generic
arithmetic and comparison operations that work on any kind of quantity; read on!

### Arithmetic and Comparison

You can do basic math with `Quantity` values:

```elm
-- 6 feet 3 inches, converted to meters
Length.feet 6 |> Quantity.plus (Length.inches 3) |> Length.inMeters
--> 1.9050000000000002

-- pi radians plus 45 degrees is 5/8 of a full turn
Quantity.sum [ Angle.radians pi, Angle.degrees 45 ] |> Angle.inTurns
--> 0.625

-- Area of a triangle with base of 2 feet and height of 8 inches
Quantity.product (Length.feet 2) (Length.inches 8)
    |> Quantity.scaleBy 0.5
    |> Area.inSquareInches
--> 96
```

Special support is provided for calculations involving rates of change:

```elm
-- How long do we travel in 10 seconds at 100 km/h?
Duration.seconds 10
    |> Quantity.at (Speed.kilometersPerHour 100)
    |> Length.inMeters
--> 277.77777777777777

-- How long will it take to travel 20 km if we're driving at 60 mph?
Length.kilometers 20
    |> Quantity.at_ (Speed.milesPerHour 60)
    |> Duration.inMinutes
--> 12.427423844746679

-- How fast is "a mile a minute", in kilometers per hour?
Length.miles 1 |> Quantity.per (Duration.minutes 1) |> Speed.inKilometersPerHour
--> 96.56064

-- Reverse engineer the speed of light from defined lengths/durations
speedOfLight =
    Length.lightYears 1 |> Quantity.per (Duration.years 1)

speedOfLight |> Speed.inMetersPerSecond
--> 299792458

-- One astronomical unit is the (average) distance from the Sun to the Earth
-- Roughly how long does it take light to reach the Earth from the Sun?
Length.astronomicalUnits 1 |> Quantity.at_ speedOfLight |> Duration.inMinutes
--> 8.316746397269274
```

Note that the various functions above are not restricted to speed (length per
unit time) - any units work:

```elm
pixelsPerInch =
    Pixels.pixels 96 |> Quantity.per (Length.inches 1)

Length.centimeters 3 |> Quantity.at pixelsPerInch |> Pixels.inPixels
--> 113.38582677165354
```

Finally, `Quantity` values can be compared/sorted:

```elm
Length.meters 1 |> Quantity.greaterThan (Length.feet 3)
--> True

Quantity.compare (Length.meters 1) (Length.feet 3)
--> GT

Quantity.max (Length.meters 1) (Length.feet 3)
--> Length.meters 1

Quantity.maximum [ Length.meters 1, Length.feet 3 ]
--> Just (Length.meters 1)

Quantity.sort [ Length.meters 1, Length.feet 3 ]
--> [ Length.feet 3, Length.meters 1 ]
```

### Custom Functions

Some calculations cannot be expressed using the built-in `Quantity` functions.
Take kinetic energy `E_k = 1/2 * m * v^2`, for example - the `elm-units` type
system is not sophisticated enough to work out the units properly. Instead,
you'd need to create a custom function like

```elm
kineticEnergy : Mass -> Speed -> Energy
kineticEnergy (Quantity m) (Quantity v) =
    Quantity (0.5 * m * v^2)
```

In the _implementation_ of `kineticEnergy`, you're working with raw `Float`
values so you need to be careful to make sure the units actually do work out.
(The values will be in [SI](https://en.wikipedia.org/wiki/International_System_of_Units)
units - meters, seconds etc.) Once the function has been implemented, though, it
can be used in a completely type-safe way - callers can supply arguments using
whatever units they like, and extract results in whatever units they want:

```elm
kineticEnergy (Mass.tonnes 1.5) (Speed.milesPerHour 60)
    |> Energy.inKilowattHours
--> 0.14988357119999998
```

### Custom Units

`elm-units` defines many standard unit types, but you can easily define your own! See [CustomUnits](doc/CustomUnits.md) for an example.

### Understanding Quantity Types

The same quantity type can often be expressed in multiple different ways, which is important to understand especially when trying to interpret error messages! Take the `Speed` type alias as an example. It is defined as

```elm
type alias Speed =
    Fractional MetersPerSecond
```

Expanding the `MetersPerSecond` type alias, this is

```elm
Fractional (Rate Meters Seconds)
```

Expanding the `Fractional` type alias then gives

```elm
Quantity Float (Rate Meters Seconds)
```

which is the "true" type with no type aliases left to expand.

## Getting Help

For general questions about using `elm-units`, try asking in the [Elm Slack](http://elmlang.herokuapp.com/)
or posting on the [Elm Discourse forums](https://discourse.elm-lang.org/) or the
[Elm subreddit](https://www.reddit.com/r/elm/). I'm **@ianmackenzie** on all
three platforms =)

## API

Full API documentation will be available on the Elm package web site once this
package is published.

## Contributing

TODO

## License

[BSD-3-Clause © Ian Mackenzie](LICENSE)
