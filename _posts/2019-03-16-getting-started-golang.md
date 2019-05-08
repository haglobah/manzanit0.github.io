---
layout: post
title: "Getting started with Golang"
author: Javier García
description: "My personal journey with Go"
category: golang
apprenticeship: false
tags: golang, vim
---

# Getting started with Golang

For a long time, one of the things I've enjoyed the most in my free time is
reading and learning new languages. Last week I started to pick up Go. It's one
of those languages which you've usually heard both a lot of good things and a
lot of bad things.

## Setting up the environment

Usually, before I start playing around with any language, I try to have my editor
running with a couple of plugins so it can help me around. For Go, I went
with `vim` and [`vim-go`](https://github.com/fatih/vim-go). Since I want to
focus more on actual Go in this blog post, I'll just forward you to my
[dotfiles](https://github.com/Manzanit0/dotfiles/blob/master/src/.nvimrc#L106)
repository in case you're interested in my setup. Nonetheless, feel free to use any editor
of your choice. Once you have your editor up and running, we can install the Go
compiler and create a project.

To install the Go compiler, I think the most accurate information is in the 
[Getting Started](https://golang.org/doc/install) page, under the golang website.
In case you're running OSX like me, you can simply run `brew install go` and it will
install the latest version, as of this writing, v1.12.

## To GOPATH or not to GOPATH

Once we have an editor installed and the compiler in place, the first question that arises
is *how do I set up an initial project?*

Prior to Go 1.11, the recommended way to develop in Go was through a unique
workspace located at what they call the `GOPATH`, which usually was `$HOME/go`. The problem
with this approach is that if you want to create a project on a different path, it's cumbersome.
Also, the imports all go through that common workspace.

The good thing is that as of Go 1.11, [Go modules](https://github.com/golang/go/wiki/Modules)
were introduced. This basically allows us to create a Go repository wherever we want,
set it up as a module and start coding, right there. No common workspace worries. No awkward
import errors.

### Setting up a Go module

Setting up a Go module is actually very easy. All we have to do is create our directory,
throw in a Go file and use the `go mod` tool. Let's check it out.

First, we create a directory outside our `GOPATH`:

```bash
mkdir gomoody
cd gomoody/
```

We add some go magic to it:

```bash
cat <<EOF > main.go
package main

import (
    "fmt"
)

func main() {
    fmt.Println("Hello world!")
}
EOF
```

And now make it a module:

```bash
go mod init gomoody
```

Up to this point you will probably be thinking what's the big deal so far. The good thing
is that right now, if we are to create a couple of packages inside our `gomoody` project,
let's say like the following structure:

```
gomoody/
├── main.go
├── go.mod
├── bar
│   └── bar.go
└── foo
    └── foo.go
```

We can import any of the packages through an absolute path along the lines of `gomoody/bar`,
making it much easier to have relative packages.

## Building a contact manager

Currently we have a project created, set up as a module, but with no code in it. To touch base with
basic Go syntax, we're going to build a very simple contact manager. A small application which
will allow us to create contacts, add them to an addressbook and list them. Nothing fancy, but
enough to get a basic understanding of Go syntax.

### Starting with a failing test

Since I started to work at 8th Light, I've discovered that the best way to start learning a language
is to start with tests. They help you get acquainted with the basic operators and very quickly introduce
you into a rapid loop of feedback regarding failing code – This allows you to quickly see if the
written code makes sense.

Let's start by creating a test which asserts the creation of a contact. For that we will create a new
file named `addressbook_test.go` in our module directory, with the following content:

```go
package main

import "testing"

func TestCreateContact(t *testing.T) {
	c := NewContact("Javier", "jgarcia@8thlight.com")

	if c.Name != "Javier" {
		t.Errorf("The contact's first name doesn't match")
	}

	if c.Email != "jgarcia@8thlight.com" {
		t.Errorf("The contact's email doesn't match")
	}
}
```

Before executing our test, let's highlight a couple of things:

1. In order for Go to understand a test it must be a function named as `func TestCreateContact(t *testing.T)`.
As long as it's camel-cased and accepts `t *testing.T` as a parameter, we're good to go.

2. By placing our tests in a separate file ending in `_test.go` we're telling the Golang compiler to
build those files into a separate package, excluded from the production build. We don't have to actually worry
about our tests being in the same directory as the actual code.

3. As you might have noticed, we haven't used any assertion-type checks in the test. This is because the Go designers
thought that it would be best to leave the checking of errors and values up to the developers, *less is more*, they say.
In order to achieve this, `t` gives us a couple of methods to throw errors upon an unexpected result, like `Errorf`.

Now, if we execute the above code it will give is an error along the lines of  `undefined: New`. Let's try to fix it.
For it, let's create another file called `addressbook.go` with the following content:

```go
package main

type Contact struct {
	Name  string
	Email string
}

func NewContact(name, email string) Contact {
	return Contact{name, email}
}
```

If we now go to the console, place ourselves in our project's root directory and execute `go test`, we'll get:

```
PASS
ok      gomoody  0.004s
```

Brilliant! But what does the code do? First, we have created a structure with two properties, `Name` and `Email`.
In Go, types are usually represented by structures – There isn't a concept of classes, like in Java or C#. Secondly,
we have created a function which creates a structure of type `Contact` and returns it with the values sent via the
parameters. As you've seen in the test, to access the properties of a structure, it's as easy as `c.Name` or `c.Email`.

### Naming conventions

As you might have noticed, so far we have named all our functions with camel-cased syntax. In go it's not just a matter
of preference but a matter of the compiler's preference. The Go compiler understands that all functions which start
with an uppercase letter has to be exported (public visibility) and all functions which start with lowercase don't
(private visibility). As everything in Go, it's quite opinionated, but on the other hand it makes it very easy for us
to make certain functions public or private.

### Adding more functionality

Let's say that we want to add a function to, ironically, add contacts to an addressbook structure that we want.
How would we like the test to look? I'd like it like this:

```go
func TestAddContact(t *testing.T) {
	addressbook := new(Addressbook)
	contact := New("Javier", "jgarcia@8thlight.com")

	addressbook.Add(contact)

	if addressbook.Contacts == nil {
		t.Errorf("The addressbook hasn't been initialized")
	}

	if len(addressbook.Contacts) <= 0 {
		t.Errorf("The addressbook doesn't have any contacts")
	}

	if name := addressbook.Contacts[0].Name; name != "Javier" {
		t.Errorf("Expecting name to be Javier, but is %s", name)
	}
	
	if email := addressbook.Contacts[0].Email; email != "jgarcia@8thlight.com" {
		t.Errorf("Expecting email to be jgarcia@8thlight.com, but is %s", email)
	}
}
```

Again, two things to highlight:

1. In the first line the test uses the function `new`. It's basically a shorthand for allocating memory.
Do take into account that it *allocates* but does not *initialize*. This means it fills up the allocated
memory with zeros.

2. Even though we said above that structures are primitive types not at all like classes, we've used some
kind of syntax which makes the addressbook seem like it has some behaviour to it: `addressbook.Add(...)`.
That's called a *method*. As I said, Go structures are primitive types and they only hold data,
Nonetheless Go does have a syntax which allows us to use functions in a way which makes it look like that.

	If we declare a function like `func (a *Addressbook) Add(c *Contact) {...}` it's basically the same as passing
the Addressbook as a parameter, just that we can invoke it like above. Alas, syntactic sugar. In this case,
the addressbook variable is called the *receiver type*.

From our test we now know that we want a new structure, an `Addresbook`, and a method attached to it. The
structure will have a property `.Contacts` which will be an array which will allow us to add contacts to it.
Let's code that!

```go
type Addressbook struct {
	Contacts []Contact
}

func (addressBook *Addressbook) Add(contact Contact) {
	addressBook.Contacts = append(addressBook.Contacts, contact)
}
```

Aaaaaand, if we execute again `go test`, it should pass. As you can see, we've used the *method* syntax
so the function is attached to the Addressbook and the function simply uses the standard `append` function
to add an element to a *slice*.

## Wrapping up

We started the post talking about Go modules but we didn't really explore much. What we're going to do now is
move the two addressbook files into a directory of their own, call the package `addresbook` and enhance our
main function within `main.go` to make our application work! Thanks to Go modules, doing this is a breeze.

To create and move the files, let's run the following commands in the terminal:

```bash
mkdir addresbook
mv addressbook.go addressbook/addressbook.go
mv addressbook_test.go addressbook/addressbook_test.go
```

Now, let's change `package main` to `package addressbook` in both files. Once we've done that, let's fill our 
`main.go` file with the following code:

```go
package main

import (
	"fmt"
	"gomoody/addressbook"
)

func main() {
	a := new(addressbook.Addressbook)
	luke := addressbook.NewContact("Luke S.", "luke@skywalker.com")
	finn := addressbook.NewContact("Finn, The Human", "finn@treehouse.ooo")

	a.Add(luke)
	a.Add(finn)

	fmt.Println(a.Contacts)
}
```

If you're using `vim-go` like me, you can actually write the code without the imports and upon save, Go will fill
those in for you.

If we now run `go run main.go`...

```
[{Luke S. luke@skywalker.com} {Finn, The Human finn@treehouse.ooo}]
```

A list with both our contacts displays on the console, proving how our application can add contacts and 
display the list. At the end of the day, this is but a little taste of Go. There is much more to explore, discover
and learn. If you're interested, I have further enhanced the Contact Manager application by adding a few more functions
as well as a web API set up with the `gin` framework ([Repository](https://github.com/Manzanit0/gontact-manager)).
Hopefully that serves as a further example.

Further more, in the end are some resources I found helpful. Happy coding!

## References

### Miscellaneous

- [The Go Tour](https://tour.golang.org/welcome/1)
- [How to write Go code](https://golang.org/doc/code.html)
- [Writing unit tests in Go](https://blog.alexellis.io/golang-writing-unit-tests/)
- [7 Common mistakes in Go and how to avoid them](https://vimeo.com/115776445)
- [Writing effective and idiomatic Go](https://golang.org/doc/effective_go.html)
- [Who needs generics? Use... instead!](https://appliedgo.net/generics/)

### On slices
- [Go slices, usage and internals](https://blog.golang.org/go-slices-usage-and-internals)
- [Arrays, slices (and strings): The mechanics of 'append'](https://blog.golang.org/slices)
- [Slices - tricks](https://github.com/golang/go/wiki/SliceTricks)

### Vim & vim-go
- [vim-go tutorial](https://github.com/fatih/vim-go-tutorial)
- [A neovim setup for go](https://hackernoon.com/my-neovim-setup-for-go-7f7b6e805876)

### Free Books
- [The Little Go book](https://www.openmymind.net/The-Little-Go-Book/)
- [Go Bootcamp](http://www.golangbootcamp.com)
- [An introduction to programming in Go](http://www.golang-book.com/)
