---
layout: post
title: Shadowing variables in Go
author: Javier Garcia
category: Go
tags: Go, shadowing
---

Today I learn you can shadow shit in Go. The only thing I can think of is...
why would you let me do that, Go?

```go
package main

import (
	"fmt"
)

func main() {
	var myVar int
	myVar = 1

	if true {
		myVar := 5
		myVar++ // Compiler pleasing
	}

	fmt.Println(myVar) // prints 1
}
```

Guess why? It's that damn `:=` operator in the if. Different scope, so it's
allowed. But it shadows.
