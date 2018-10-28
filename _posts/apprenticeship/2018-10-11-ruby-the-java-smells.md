---
layout: post
title: "Ruby: The Java smells"
author: Javier Garcia
description: "How I kept showing my java-style whilst learning ruby + lessons learnt."
category: ruby
apprenticeship: true
tags: 8th-light, java, ruby, learning, smells
---

After a couple of weeks programming in Ruby every day, all the time, one begins to think he is starting to get the hang of it. But no.

In order to get a good grasp of the new language, I’ve been working on developing a [console-based TicTacToe](https://github.com/Manzanit0/TicTacToeRB). I started nice and simple with a single file, then created some classes until I eventually achieved a fully working human vs human prototype. Along the way, I took the initiative of reading up on conventions, common practices by seasoned Rubyists, etc. Surprisingly enough, at the end of my weekly iteration, while code-reviewing my solution, I found out that it takes much more than just reading and two weeks of practice to get a proper grasp of a language.

It’s fairly easy to craft a working solution in any language. At the end of the day, it’s just learning semantics. OOP is and will be the same in Java, C# or Ruby. On the other hand, even though paradigms don’t change,  expressiveness does. In Ruby many common tasks are achieved cutting, slicing and transforming lists and hashes with built-in functional methods. On the other hand, in C# you end up iterating and crafting your own solution. Or importing a library. Unluckily, C# doesn’t have a beautifully crafted Enumerable API, like Ruby.

Next, I would like to share the method I wrote to compute if a board had a winning sequence of plays or not:

```ruby
def has_won?(player)
  column_win, row_win, diagonal_win, anti_diagonal_win = true, true, true, true

  (0...@size).each do |row|
    (0...@size).each do |column|
      row_win = false if @board[row][column] != player.symbol
      column_win = false if @board[column][row] != player.symbol
      diagonal_win = false if row == column && @board[row][column] != player.symbol
      anti_diagonal_win = false if @board[row][@size-1-row] != player.symbol
    end

    return true if (
      row_win ||
      column_win ||
      (diagonal_win && (row == @size - 1)) ||
      (anti_diagonal_win && (row == @size - 1))
    )

    column_win, row_win = true, true
  end

  false
end
```

As you can see, the method is complex, but then again, there is some complexity to the problem: given a 3×3 matrix, resolve if there are any 3 characters in a row that are the same. The main issue is… I simply used loops to solve the problem. I didn’t use any `inspect` or `slice` method to solve it. That’s what we do in C# or Java. That’s not what they do in Ruby.

And that’s how I realised that learning something is not just about reading as much as you can or even practising as much as you can. To truly learn something you must let go of all your previous ideas and mental maps to embrace the new ones. Then, and just then, you might have a real shot at learning.