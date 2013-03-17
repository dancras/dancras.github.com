---
layout: post
title: Over-engineering tic-tac-toe
date: 2013-03-17 02:30:00
---

The current problem for our weekly code kata at [work](http://inviqa.com) is tic-tac-toe (noughts and crosses). The focus of a code kata is not the problem but the process, however once we were finished myself and a colleague got into a surprisingly heated discussion over the simple little game. From the momentum of this conversation I've written a rather excessive implementation in PHP.


Value objects
=============

I started by creating classes to represent the primitives of the game: Symbol and Coordinate. SOLID design will typically distribute functionality across a number of classes, and it is important that those classes are explicit about what values they are expecting in their primitives. Passing around low level primitives eg. String, Integer works, but creating domain specific primitives means that developers can be confident about the possible values contained.

For example a Coordinate is not just a number; it is specifically an non-nullable integer greater than or equal to zero. By enforcing this in the constructor and type hinting my classes with Coordinate I am removing any chance of repeating checks against null, casting operations, failed strict comparisons etc. Low level primitives should be considered unsafe boundary data and should not have a place within a well defined model.


Line mappers
============

The first thing people do when they implement tic-tac-toe is create a grid to store the moves in. This makes sense since the visual representation of the game is a grid, and it is also the most normalised representation too. However in the model layer our main concern should be the expressiveness and testability of the rules, and normalised data is not typically associated with simple or expressive queries.

The result of using a grid is high complexity for getting new moves into the game, and high complexity when checking for a winner; conditionals, nested in loops, nested in further conditionals. The code can be split among a number of private methods for better legibility, but realistically all the complexity ends up under the same public method. Consequently, testing the "playMove" or "getWinner" methods thoroughly will basically involve playing every possible outcome of the game - an option that doesn't scale to larger problems.

In order to split this complexity, I am storing the game as a set of lines, one for each possible win. The responsibility of the line mappers is to map played moves to the correct position on the list of lines. For the standard game of tic-tac-toe there are 4 mappers:

 * ForwardDiagonalMapper
 * BackwardDiagonalMapper
 * HorizontalMapper
 * VerticalMapper

Playing a move in the top left corner (0, 0) would result in the BackwardDiagonalMapper setting symbol on its line at position 0, the HorizontalMapper adding a symbol at its first row line at position 0, and the VerticalMapper adding a symbol at its first column line at position 0.

![Illustration of line mappers](/media/2013-03-17/line-mappers.jpg)

The above image shows the three lines that would be affected by playing a move in the top left corner (0, 0).


Lines
=====

Thanks to the line mappers, any constraints (eg. highest coordinate), or logic (eg. game is won) will now affect all the rows, columns and diagonals with just a single implementation. Initially I chose to store symbols in an array, but the logic to check for a winner was a little too hairy for an over-engineered solution. So, inspired by what I learned from the [functional programming in scala course](https://www.coursera.org/course/progfun) (highly recommended), I have the following.

Lines are immutable. Setting a symbol on a line will return a new line with the additional symbol set. Each instance of a line stores a coordinate, the symbol at that coordinate, and the line instance which created it (the rest of the line). Consequently a line is a chain of objects.

![Chain of line objects](/media/2013-03-17/line-chain.jpg)

There are three classes of line:

 * EmptyLine
 * WinningLine
 * DeadLine

An EmptyLine always returns a WinningLine.

A DeadLine always returns a DeadLine.

A WinningLine returns a WinningLine if the symbol matches its own:

![Winning line chain](/media/2013-03-17/winning-line.jpg)

Or a DeadLine if the symbol differs from its own:

![Dead line chain](/media/2013-03-17/dead-line.jpg)

The chain of line objects is used to validate moves, and to calculate the number of coordinates in a line. Creating a WinningLine with 3 coordinates will notify the WinObserver that the game is won with the winning symbol. No loops, no nested conditionals, just a simple length check.


The Game
========

Bringing it all together, the Game is responsible for proxying moves to the four mappers, and notifying the MoveObserver when a move is played without exception. It also keeps track of whose turn it is.

The API is very simple; a single playMove function, and two observer interfaces: IWinObserver and IMoveObserver.

Through the use of type hinting throughout the constructors, incorrectly assembling an instance of Game is actually quite difficult without an immediate error. The weakest classes are the horizontal and vertical mappers which just require an array; if PHP type hinting supported [generics](http://msdn.microsoft.com/en-us/library/512aeb7t%28v=vs.80%29.aspx) this could be solved in a second. I created a very rough application using this code and it worked first time; can't ask much more than that.

* * *

The code is [available on github](https://github.com/dancras/tic-tac-toe) for anyone who is interested.