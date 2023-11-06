---
layout: post
title: JSON polymorphism in Go
category: go
author: alex
---

How to serialize and deserialize polymorphic JSON into Go structures.
{:.lead}

## Introduction

It is quite common in object-oriented languages to rely on polymorphism to restrict the range of types that can be used in a particular case. This can be achieved either by using an interface or by subtyping.

In Go we can use an interface to achieve polymorphism, but subtyping however is not possible as there is no traditional notion of inheritance (e.g. via subclassing). We can embed a struct within another, but that doesn’t achieve the same effect as subtyping.

## Example

Let’s take an example that is commonly used to describe this problem. A fleet of vehicles which can be either cars, trucks, bikes or any special case of vehicle we can imagine.

```json
{
  "vehicles": [
    {
      "type": "car",
      "make": "BMW",
      "model": "M3",
      "seatingCapacity": 4,
      "topSpeed": 250
    },
    {
      "type": "truck",
      "make": "Volvo",
      "model": "FH",
      "payloadCapacity": 40000
    },
    {
      "type": "bike",
      "make": "Yamaha",
      "model": "YZF-R1",
      "topSpeed": 293
    }
  ]
}
```

## Types

Let’s attempt to define types that can serialize and deserialize this payload.

```go
type Vehicle struct {
    Type  string `json:"type"`
    Make  string `json:"make"`
    Model string `json:"model"`
}

type Car struct {
    SeatingCapacity int     `json:"seatingCapacity"`
    TopSpeed        float64 `json:"topSpeed"`
    Vehicle
}

type Truck struct {
    PayloadCapacity float64 `json:"payloadCapacity"`
    Vehicle
}

type Bike struct {
    TopSpeed float64 `json:"topSpeed"`
    Vehicle
}

type Fleet struct {
    Vehicles []interface{} `json:"vehicles"`
}
```

While Go will happily serialize our example payload using these data structures, there are some serious limitations.

Most importantly, when deserializing Go won’t be able to figure out when to deserialize into a Car, Truck or Bike. It will instead populate a map[string]interface{} which is not really what we want.

Also important is the fact that the vehicles are not limited much by the empty interface. In fact any type implements the empty interface.

To address the first issue, we’ll need to implement our own MarshalJSON and UnmarshalJSON methods for the Fleet type. We’ll also need a json.RawMessage field to represent our vehicles in their intermediate state.

```go
type Fleet struct {
    Vehicles    []interface{}     `json:"-"`
    RawVehicles []json.RawMessage `json:"vehicles"`
}
```

Notice how the JSON tags now instruct Go’s json package to ignore the Vehicles field when serializing/deserializing but to use RawVehicles instead.

## Methods

```go
func (f *Fleet) UnmarshalJSON(b []byte) error {

    type fleet Fleet

    err := json.Unmarshal(b, (*fleet)(f))
    if err != nil {
        return err
    }

    for _, raw := range f.RawVehicles {
        var v Vehicle
        err = json.Unmarshal(raw, &v)
        if err != nil {
            return err
        }

        var i interface{}
        switch v.Type {
        case "car":
            i = &Car{}
        case "truck":
            i = &Truck{}
        case "bike":
            i = &Bike{}
        default:
            return errors.New("unknown vehicle type")
        }

        err = json.Unmarshal(raw, i)
        if err != nil {
            return err
        }

        f.Vehicles = append(f.Vehicles, i)
    }

    return nil
}
```

What we’ve done here, is alias the Fleet type to fleet internally, so that json.Unmarshal doesn’t get trapped in an infinite loop. Once the unmarshaling takes place, f is populated with the contents of the payload. But until this point Vehicles is an empty slice, and RawVehicles is populated with all vehicles in the payload, but as raw JSON.

At this stage we will examine each item in RawVehicles by unmarshaling it into an intermediate representation which can help us understand what type of vehicle we need to unmarshal into. We could use a map[string]interface{} but since we already abstract some common properties into the Vehicle struct, we’ll just go ahead and use that.

Now that the type is known to us, we then use a switch statement to unmarshal the vehicle into its correct struct — one of Car, Truck or Bike. Then it’s only a matter of adding each car, truck or bike to the Vehicles slice.

Since we introduced the RawVehicles field and instructed the json package to ignore Vehicles we’ll need to implement the MarshalJSON method to ensure the correct serialization takes place. To do so, we will serialize each item in Vehicles and copy it to RawVehicles. Then let json.Marshal do what it usually does.

```go
func (f *Fleet) MarshalJSON() ([]byte, error) {

    type fleet Fleet

    if f.Vehicles != nil {
        for _, v := range f.Vehicles {
            b, err := json.Marshal(v)
            if err != nil {
                return nil, err
            }
            f.RawVehicles = append(f.RawVehicles, b)
        }
    }

    return json.Marshal((*fleet)(f))
}
```

With all structs and their methods in place, we can now deserialize into Go structs from this polymorphic JSON payload.

```go
var f *Fleet
err := json.Unmarshal([]byte(body), &f)
if err != nil {
    t.Fatal(err)
}

for _, vehicle := range f.Vehicles {

    switch v := vehicle.(type) {
    case *Car:
        fmt.Printf("%s %s has a seating capacity of %d and a top speed of %.2f km/h\n",
            v.Make,
            v.Model,
            v.SeatingCapacity,
            v.TopSpeed)
    case *Truck:
        fmt.Printf("%s %s has a payload capacity of %.2f kg\n",
            v.Make,
            v.Model,
            v.PayloadCapacity)
    case *Bike:
        fmt.Printf("%s %s has a top speed of %.2f km/h\n",
            v.Make,
            v.Model,
            v.TopSpeed)
    }
}
```

Now that we addressed the correct serialization and deserialization, we can improve upon the example by replacing the empty interface with something that actually limits the types we can use in the Vehicles slice. In principle, we’ll only be replacing it with a more narrow interface, and make sure our vehicles implement that interface.

```go
type FleetVehicle interface {
    MakeAndModel() string
}
```

This interface is now much narrower. Only types that implement MakeAndModel() function are legal implementations of FleetVehicle. Let’s replace all instances of the empty interface and replace them with FleetVehicle.

```go
type Fleet struct {
    Vehicles    []FleetVehicle    `json:"-"`
    RawVehicles []json.RawMessage `json:"vehicles"`
}
```

## Conclusion

Using Go’s interfaces and a little creative use of the MarshalJSON and UnmarshalJSON methods, we were able to serialize and deserialize polymorphic JSON data structures into Go structs.

By replacing the empty interface with a smaller, more narrow interface, we allow ourselves to manipulate polymorphic objects without necessarily knowing their concrete implementation.

See the complete example [here](https://gist.github.com/alexkappa/4b541a712dc06c047a38b005178978b5).
