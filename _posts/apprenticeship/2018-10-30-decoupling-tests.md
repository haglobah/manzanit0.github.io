---
layout: post
title: "TDD: keeping tests decoupled"
author: Javier Garcia
description: "TIL how important is to keep tests decoupled"
category: blog
apprenticeship: true
tags: tdd, testing, design
---

Most of the times, the design of choice for test suits is to create a test class for each other class in the production code base. If you have a `Person` class, then also have a `PersonTest` class which tests each of the methods. This makes our code base _covariant_. Today I learnt that covariant code ends up leading to coupled code, and coupled code to code that is difficult to extend.

One of the biggest symptoms of a tightly coupled tests suite is that everytime you change the structure of your production code, a lot of test classes break. This is what some people call the *Fragile Test Problem*.

Today, as I was refactoring the toy TicTacToe application which I'm building during the apprenticeship at 8th Light, this happened to me. I was trying to change the underlying data structure of the application from a matrix to a list. Initially, it went like a breeze – the design of the code was quite good, so I only had to touch a couple of classes; I stayed green throughout the whole process. The problem came when I had to finally touch the actual API of the game's board... suddenly, the tests from the board, the game, the IO, all of them broke. Why? Simple as they come, they were all dependant on the board class' methods.

A good way to resolve this problem is to keep the public API as generic, simple and effective as possible, and have the underlying classes do the magic. This way we can test all the underlying classes via the public API and if we change the implementation, tests will never break, but remain green (given that they should).

To illustrate this, take for instance my Board class with a `make_move` method which takes the coordinates of the move:

```ruby
class Board
  def make_move(row, column)
    # ...
  end
end
```

If instead we have the `make_move` method take a `move` instance, we're hiding the representation of the move. It could be coordinates, or simply an integer which represents a position. In any case, even if we change the implementation of the move, the API of the board class remains the same:

```ruby
class Board
  def make_move(move)
    # ...
  end
end
```

As you can see, when I changed my matrix-like implementation to a position-based one, I had to change the public API of the board, and therefor a lot of tests. On the other hand, if I had encapsulated that, it wouldn't have happened.