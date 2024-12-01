---
layout: post
title: Structs in Go for Mortals
date: 2024-01-01
tags: [go]
---

Structs in Go are used to group related data together under one type. They are similar to classes (sort of) in other languages. In this post, we will explore how to define and use structs in Go.

### Defining a Struct

To define a struct, you use the `type` keyword followed by the name of the struct you'd like to create. You then list the fields of the struct. Here's an example of a struct that represents a person:

```go

type Car struct {
    Model string
    Year  int
}
```


### Creating an Instance of a Struct

Now that we have a defined it, we can create an instance of it and then we can access its fields using `.` operator :

```go
myCar := Car{}
myCar.Model = "Tesla"
myCar.Year = 2022
```

We can also create an instance directly and assign values to its fields:

```go

mycar := Car{
    Model: "Tesla",
    Year:  2022,
}
```

Now, `mycar.Model` will be "Tesla" and `mycar.Year` will be "2022".


### Nested Structs

You can create nested structs by defining a struct within another struct.

```go
type Person struct {
    Name string
    Age  int
    Car  Car
}

type Car struct {
    Model string
    Year  int
}
```

Here, the `Person` struct has a field `Car` that is of type `Car`. You can then create an instance of `Person` and access its nested `Car` field:

```go

newPerson := Person{
    Name: "Alice",
    Age:  30,
    Car: Car{
        Model: "Tesla",
        Year:  2022,
    },
}
fmt.Println(newPerson.Car.Model) // Output: Tesla

```

You can literally define nested structs as well, for example : 

```go
type Person struct {
    Name string
    Age  int
    Car struct {
        Model string
        Year  int
    }
}
```

### Anonymous Structs

You can also define anonymous structs, which are structs without a name. These are useful when you need a struct for a short period of time and don't want to define a named type. Hence, they cannot be used outside the function where they are defined.

```go

person := struct {
    Name string
    Age  int
}{
    Name: "Alice",
    Age:  30,
}
```


#### Embedded Structs

You can embed a struct within another struct to create a new type that inherits the fields and methods of the embedded struct.

```go
type Car struct {
    Model string
    Year  int
}

type Person struct {
    Name string
    Age  int
    Car
}
```

Now, the `Person` struct has a `Car` field, which is of type `Car`. You can access the fields of the embedded `Car` struct directly from an instance of `Person`:

```go
p := Person{
    Name: "Alice",
    Age:  30,
    Car: Car{
        Model: "Tesla",
        Year:  2022,
    },
}
```

We can access the fields of the embedded `Car` struct directly, for example:

```go
// instead of p.Car.Model
fmt.Println(p.Model) // Output: Tesla
```


### Empty structs

You can create empty structs, which are structs with no fields. These are mostly useful (so far, for me) to create a map with no values. If you only want to check a key exists, where keys are not meaningful, use empty structs. For example:

```go

// named empty struct
empty := struct {}{}

myMap := make(map[string]struct{})

myMap["key"] = struct{}{}

//you can check if a key exists in the map
if _, ok := myMap["key"]; ok {
    fmt.Println("Key exists")
}
```

### Parsing JSON data with Structs

In Go, if you have to work with JSON data, say you make a request to an API and get a JSON response, you can parse that JSON data into a struct. For example, if you `https://pokeapi.co/api/v2/location-area` returns the following JSON data:

```json
{
    "count": 1054,
    "results": [
        {
            "name": "canalave-city-area",
            "url": "https://pokeapi.co/api/v2/location-area/1/"
        },
        {
            "name": "eterna-city-area",
            "url": "https://pokeapi.co/api/v2/location-area/2/"
        }
        ...
    ]
    ...
}
```

We can create a struct to represent this data:

```go
type LocationResp struct {
    Count int `json:"count"`
    Results []struct {
        Name string `json:"name"`
        URL  string `json:"url"`
    } `json:"results"`
}
```

Now, in order for us to parse the JSON data into the struct, we will use `json.Unmarshal()` function:

```go

resp, err := http.Get("https://pokeapi.co/api/v2/location-area")

body, err := io.ReadAll(resp.Body)

location := LocationResp{}

err  = json.Unmarshal(body, &location)
if err != nil {
    log.Fatal(err)
}

//now we can access the fields of the location struct

fmt.Println(location.Count) // Output: 1054

for _, result := range location.Results {
    fmt.Println(result.Name, result.URL)
}
```
One thing to be aware of is `json.Unmarshal()` takes a byte slice and a pointer to the struct where the data will be unmarshalled.

### Structs as Function Arguments
You can pass structs to functions as arguments. For example:

```go
func printCar(c Car) {
    fmt.Printf("%s - %d",c.Model, c.Year) // Output: Tesla - 2022
}

func main() {
    myCar := Car{
        Model: "Tesla",
        Year:  2022,
    }

    printCar(myCar)
}
```

### Structs as method receivers : Value

You can define methods on structs by using a receiver. A receiver is a parameter that is passed to a method. Here's an example of a method that prints the details of a car:

```go
func (c Car) printCar() {
    fmt.Printf("%s - %d",c.Model, c.Year)
}
```

You can then call this method on an instance of the `Car` struct:

```go
myCar := Car{
    Model: "Tesla",
    Year:  2022,
}

myCar.printCar()  // Output: Tesla - 2022
```

### Structs as method receivers : Pointer

The previous example is a value receiver. That means, whenever `printCar()` is called, it gets a copy of the `Car` struct. If you want to modify original the struct, you can use a pointer receiver:

```go

func (c *Car) setModel(model string) {
    c.Model = model
}

func (c *Car) printCar() {
    fmt.Printf("%s - %d",c.Model, c.Year)
}

func main() {
    myCar := Car{
        Model: "Tesla",
        Year:  2022,
    }

    myCar.setModel("BMW")

    myCar.printCar() // Output: BMW - 2022
}

```

In pointer receiver, we use `*` before the type of the receiver. This means that the method will receive a reference to the struct, and any changes made to the struct will be reflected in the original struct.

Notice how we created an instance of the `Car` struct in the beginning and set the model to "Tesla". Then we called the `setModel()` method on the instance and passed "BMW" as an argument. The method then changed the model of the car to "BMW". Without the pointer receiver, the model would have remained "Tesla".


### Difference between Value and Pointer Receivers

When you define a method on a struct, you can use either a value receiver or a pointer receiver. A value receiver receives a copy of the struct, while a pointer receiver receives a reference to the struct.

Pointer receivers are used when you want to modify the original struct.

Here's a [link](https://go.dev/play/p/FxM6dh4Zc6-) to go playground with code which demonstrates the difference between value and pointer receivers.


### Struct as function return type

You can also return a struct from a function : 

```go

func newCar(model string, year int) Car {
    return Car{
        Model: model,
        Year:  year,
    }
}

func (c Car) printCar() {
    fmt.Printf("%s - %d",c.Model, c.Year)
}

func main() {
    myCar := newCar("Tesla", 2022)

    myCar.printCar() // Output: Tesla - 2022
}
```


## Exported and Unexported Fields in Structs

In Go, f the first letter of a filed, method or struct name is uppercase, it is exported; if lowercase, it is unexported.

```go
// internals/foobar/foobar.go
package foobar

import "fmt"

type car struct { // unexported struct
	model string // exported
	year  int    // unexported
}

func (c car) printCar() { // unexported method
	fmt.Printf("%s - %d", c.model, c.year)
}

```

```go
//main.go
package main

import (
	"mypackage/internals/foobar"
)

func main() {
	myCar := foobar.car{
		model: "Tesla",
		year:  2022,
	}

	myCar.printCar()
}

```

As we are trying to access the unexported fields and methods of the struct, we will get an error : 
```bash
./main.go:8:18: undefined: foobar.car
```

This can be fixed by exporting the struct and its fields and methods by making the first letter of the struct, fields and methods uppercase.


### Struct Comparison

You can compare two structs using the `==` operator. Two structs are considered equal if all their fields are equal. Here's an example:

```go
func main() {
	myCar := Car{
		model: "Tesla",
		year:  2022,
	}

	myCar2 := Car{
		model: "BMW",
		year:  2023,
	}
	fmt.Println(myCar == myCar2) // Output: false
}
```

I hope this post helps to explain the basics of structs in Go based on what I have learned so far. I'll try to keep updating it as I learn more.

---

### References
- [Go by Example](https://gobyexample.com/structs)
- [Go Documentation](https://go.dev/doc/)
- [Effective Go](https://go.dev/doc/effective_go)
- [Learning Go by Instrumenting a Go Application for Prometheus Metrics](https://tanmay-bhat.github.io/posts/learning-go-by-instrumenting-a-go-application-for-prometheus-metrics/)

