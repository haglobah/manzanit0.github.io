---
layout: post
title: "Refactoring: Keeping the level of abstraction"
author: Javier Garcia
description: ""
category: refactoring
apprenticeship: true
tags: 8th-light, refactoring, abstraction
---

Today I’ve been working mostly in refactoring my Tictactoe application in order to be able to test the game loop. It’s been an interesting exercises because even though the amount of code was fairly small (~20 lines), I had kept postponing the testing due to the extra mental effort it required.

After finishing the refactor, along with the tests,  I realized what was making me feel uncomfortable with testing that piece of functionality: **the levels of abstraction were completely messed up**. 

Here is the initial code:

```ruby
def play
  clear_console
  print_menu
  option = get_menu_option
  factory = GameFactory.new
  @game = factory.create_game(option)

  clear_console
  print_board(@game.board)

  until @game.has_ended?
    @game.make_move
    clear_console
    print_board(@game.board)
  end

  if @game.winner != nil
    @out.print "The winner is #{@game.winner.symbol}"
  end
end
```

Here is the code after a little refactoring with TDD in practice:

```ruby
def play
  init_game
  until @game.has_ended?
    process_turn
  end
  restart
end

private

def init_game
  print_menu

  options = get_menu_option
  factory = GameFactory.new
  @game = factory.create_game(options)

  print_board(@game.board)
end

def process_turn
  @game.make_move
  print_board(@game.board)
end

def restart
  if @game.winner != nil
    @out.print "The winner is #{@game.winner.symbol}"
  end
  # TODO prompt for restart
end
```

It might seem like an obvious conclusion, for any fairly seasoned developer, but it’s interesting how many times we forget this principle and we end up with harder to read code. I’m finding out how, funnily enough, experience ends up making me write more complex code, rather than better and simpler code.

Regardless, my lesson today is simple: **When you keep the levels of abstraction straight, testing is easy**. If you’re finding it hard to make a test, then review the code, make the test how you would like it to be, and refactor until it passes.