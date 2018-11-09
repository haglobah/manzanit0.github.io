---
layout: post
title: "Ruby: Minimax Post-mortem"
author: Javier Garcia
description: "Lessons learnt and thoughts after implementing the minimax algorithim in my Ruby TTT."
category: ruby
apprenticeship: true
tags: ruby, algorithm, minimax, post-mortem
---

Earlier this week I finished the implementation of an unbeatable Tictactoe. In order to achieve this, there are always multiple options: One approach is a rule-based algorithm, probably the easiest to implement. I used it earlier this year for my 8th light application ([repository](https://github.com/Manzanit0/My8thTictactoe)). Another approach is to go to the recursive algorithms, like minimax or negamax. This time I implemented minimax within my ruby Tictactoe.

## What is Minimax?

Minimax is basically a decision rule for minimizing the possible loss for a worst case scenario. At the end of the day, within Tictactoe, it all comes down **the player who uses minimax always makes the best possible move always assuming that the opponent will also make the best possible move for himself.**

## How does it work?

In very few words, the minimax algorithm works by simulating all possible games, assigning a score to them and then choosing the best one based on the score.

In our Tictactoe context, Minimax assigns a positive score to a winning game, a negative score to a losing game and 0 to a draw. Then based on the length of the game (the number of turns until the win/loss/draw), it adds or subtracts the number of turns to the score. This way quicker wins will always have a higher score than slower ones and slow wins will have a higher score than a loss.

In order to simulate all possible games and pick the best ending one, Minimax builds a graph tree recursively. Every move gives place to N other possible endings up until a player wins or there are no free tiles. Once the tree is built, it will pick the best game and traverse all the way back in order to give the player the initial move he must make to get there.

## Test driving the algorithm

To achieve this I tried following a TDD approach, just to eventually realize it is not all that easy.

Initially, I went with the easy scenarios: given the possibility to make a winning move, take it and given the possibility to prevent the opponent from making a winning move, make it.

My board evolved in the following way with those two tests:

After the first test:

```ruby
def get_move(board)
    board.available_tiles.each do |index|
      board.check_tile(index, self)
      if(board.has_won?(self))
        board.uncheck_tile(index)
        return index
      end
      board.uncheck_tile(index)
    end
    nil
  end
```

After the second test:

```ruby
  def get_move(board)
    opponent = Player.new(self.symbol == "X" ? "O" : "X")
    board.available_tiles.each do |index|
       board.check_tile(index, self)
      if(board.has_won?(self))
        board.uncheck_tile(index)
        return index
      end
       board.check_tile(index, opponent)
      if(board.has_won?(opponent))
        board.uncheck_tile(index)
        return index
      end
       board.uncheck_tile(index)
    end
    nil
```
But then, after these, the next step was to be able to predict more complicated moves... so I just ended up implementing [the full algorithm](https://github.com/Manzanit0/TicTacToeRB/blob/master/lib/core/hard_machine.rb). I tried to fight against making such a huge step, but the algorithm just isn't emergent.

After working on it and multiple iterations and refactorings, this is how the pull request with the full feature looked like: [link](https://github.com/Manzanit0/TicTacToeRB/pull/6)

## Lessons learnt

The good thing is I learnt a couple of things throughout the process of implementing minimax within my Ruby Tictactoe, not necessarily technical.

The first one is that I realized about the importance of not staying stuck. It's very easy, specially when implementing complicated things, to get stuck in decision-limbo or even worse in I-dont-know-how-to-proceed-land. At this time, my mentor gave me some very valuable advice: just duplicate the logic, and feel free to go back to it some time afterwards. It might not sound like the perfect advice but sometimes creating duplicated code and overall spaghetti to then just simply refactor it becomes a lot much easier than writing that clean, optimal code from the start. It's good to strive for good results, but that should never put a stop to getting actual results. **Make it work, then refine.**

The second lesson I learnt is that using vim for refactoring Ruby code is not all that comfortable. I'm not saying there are no plugins out there, I just haven't been able to find them easily. The closest was [vim-ruby-refatoring](https://github.com/ecomba/vim-ruby-refactoring), but it still did not cut it. I checked VS Code's plugin repository, in case it had some magic beans in it, but neither. That's when tried RubyMine, and luckily it shed some light. As of today, I can happily say I'm a fan of vim and it's very comfortable to program in Ruby with vim, but when you have to refactor method names throughout multiple files, or similar things, RubyMine ended up being a much better tool.

The third lesson I learnt is that Minimax is a fun algorithm to develop, but it just doesn't cut it for production-ready applications. It's very slow when the board is empty due to the computer having to calculate all the tree and that ends up in a bad UX. It takes around 25 seconds to make the first turn if the computer plays first. I guess if I were to actually develop an application for users to play I would have to look into other options, maybe adding some Alpha-Beta pruning or simply simpler/quicker solutions.
