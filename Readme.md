# Testing a CLI

A lesson for the Turings

This is oriented around their most recent project to implement a command-line mastermind style game.


## Interface vs Library

The "I" in "CLI" stands for "interface", which is a way to expose the thing to a user.
Another interface would be the web.
On the web, you don't have streams, you can't embed turn basedness or state into the thing, because you are responding to events from a user.
You will need to be able to instantiate a game partway through and continue it.
Maybe saving it in a database until the user submits their next move, then pulling it out,
instantiating the game, playing their turn and yours, and saving it back into the databse to wait for their next move.

Thus, we see there is one piece of the code, the interface,
and there is some other facet, the core of the idea, the mastermind game itself.
We usually call this the Library code.

The library code should be utterly ignorant of the interface code.
It is a self-contained idea, it is the interface's responsibility to use the library to do what it needs.

The library should be fast, easily tested, decoupled.
It contains the logic, but makes none of the decisions.

## What feedback to look for in a test

Okay, yeah, there's value in testing:
* You can change shit and know you didn't break it
* You can press a button to see your code works rather than having to go through by hand
* You have to think through the use-cases

But some people say the last "D" in "TDD" stands for "design", rather than "development".
So how do we talk about design? We need some way to measure it.

Here's a few metrics that tests can help you identify:

### Can this code be used in novel ways?

If yes, congrats, you probably have a decent design.
If no, what is preventing this from being the case? Try and change that.

This is where we begin to see the relevance of the design.
Separating the concern of how you present the game from the concern of "what kinds of things does a game class need to provide to be used in any given way?"
The library from the interface.

Your design is implicitly better if you can use it in novel ways.
Whether you actually do this or not.
Making any change will be easier if you've done this.
Your ability to reuse your code will increase dramatically.
Maybe it will require some refactoring, but not rewriting.

### Does this code know about the environment its being used in?

A game class that talks to a stream knows it is being used on the command-line.

A game class that checks markers by their presented value knows that it is being displayed in a textual manner.
This is one of the reasons we try to separate logic from presentation.
What if you wanted to display the markers as coloured squares instead of the corresponding letters?
Does your code need to change?

What if you wanted to change the number of pegs the user was guessing?
Or the number of turns a user can take?
Or to let the user guess twice before showing them feedback.

Only the interface should change, not the underlying library code.

### Does this code talk directly to objects I might want to change?

What if you wanted to make your game available in China?
Is your code dependent on English strings?
Or on a [specific](https://github.com/turingschool-examples/numbermind/blob/731af8c07d9f2f351ce3380ec214ec181dcd21cb/lib/cli.rb#L6)
`MessagePrinter` class?

Does it [print directly to the standard output](https://github.com/turingschool-examples/numbermind/blob/731af8c07d9f2f351ce3380ec214ec181dcd21cb/lib/message_printer.rb#L3)?

How are you going to test that?
We would have to inject the stream its printing to.
Notice that this change, for our tests, corresponds to better design.
Now we could connect it to a socket and play the game over [telnet](https://en.wikipedia.org/wiki/Telnet).

### How hard is it to get at one piece of this code, independent of all the fkn context?

What if I have a guess and a secret, and I just want to get the stats.
Maybe I'm working on a solver, and I need to give it information about how correct it is.
If my logic is embedded within the game class,
I have to construct and fake out a game just to get at the piece of code that analyzes a guess.
And if my class does not present a way to provide it with the secret that I'm trying to guess,
then I'll have to open it up and mess with its variables, a very fragile practice.

### Is some idea getting tested in a multiple different places?

Do I find myself repeatedly testing the same thing?
Maybe I'm calculating some sort of presentation logic in multiple places.

This test pain is an indicator of a design problem.
There's an idea here which hasn't been centralized.
This, by the way, is what DRY (Don't Repeat Yourself) *actually* means.

I can use the test pain to extract the idea into one place,
test it there, and then show everywhere else that it goes through that logic.



## It's all about the Dependencies

So we've seen there are a lot of things I consider when thinking about design.
But at the end of the day, it's all about the dependencies.

What is a dependency? It's when some piece of code relies (depends) on something.

### Thinking about dependencies

I often think about dependencies by drawing them.

I would write down the classes or the ideas, draw circles around them,
and then draw arrows from the things that depend on other things, to the things they depend on.

Here is an example from when I was discussing [testing in SalesEngine](https://github.com/JoshCheek/fast_tests).

![Drawing Dependencies Example](https://s3.amazonaws.com/josh.cheek/images/scratch/dependencies_example.png)


### Hard dependencies
Often the thing it depends on is a class, for example
[this Numbermind CLI depends on the MessagePrinter](https://github.com/turingschool-examples/numbermind/blob/731af8c07d9f2f351ce3380ec214ec181dcd21cb/lib/cli.rb#L6).

That's a dependency. We say it's a hard dependency, because we directly reference the class.

### Soft dependencies

When we talk to a thing, but through a variable that the caller can set, that is a soft dependency.

For example, in Numbermind's Game class, [we can pass in the MessagePrinter](https://github.com/turingschool-examples/numbermind/blob/731af8c07d9f2f351ce3380ec214ec181dcd21cb/lib/game.rb#L8).
Then whenever we want to talk to it, we [reference the variable](https://github.com/turingschool-examples/numbermind/blob/731af8c07d9f2f351ce3380ec214ec181dcd21cb/lib/game.rb#L17).

Softe dependencies are generally better than hard dependencies, because we can swap them out.

For example, we can give it a MockPrinter instead of a MessagePrinter.
The MockPrinter doesn't actually print anything, it just records that it was told to print things.
Then we can assert against them in our tests like this:

```ruby
def test_game_prints_the_intro_once
  mock_printer = MockPrinter.new
  Game.new(mock_printer).play
  assert_includes mock_printer.things_printed, :intro
end
```

So we made a soft dependency in order to test better.
And what do we get for it? We now give it a ChineseMessagePrinter intead,
and now people in China can play it, because the messages are in a language they know.
No changes necessary to the Game class.

You see? By making things more testable, we improve our design,
which leads to other advantages we weren't even thinking about.

### Knowledge dependencies

Does your code know the location of a file? That's a knowledge dependency.
It's not code, but it's an idea, the code depends on that particular file in that particular place.

Does it know about the format of the file? Again, a knowledge dependency.
Try to keep these segregated from the rest of the system,
put them in one place whose responsibility is to deal with them.
Everyone else can be ignorant of this, making them inherently simpler.

### Generally anything you depend on can be drawn

Some people have [formalized](https://en.wikipedia.org/wiki/Unified_Modeling_Language)
these things, but don't worry about that.
Generally speaking, when testing is hard, or when change is hard, identify what is making it hard,
and draw that with a little circle around it.

Maybe it's "the filename" or "command-line environment".
Then draw that out.

### Injecting a dependency

Injecting a dependency is when you pass it in instead of directly talking to it.
This turns a hard dependency into a soft dependency.

This is a hard dependency:

```ruby
def print_message
  puts "hello"
end
print_message
```

Now we inject it to make it a soft dependency:

```ruby
def print_message(stream)
  stream.puts "hello"
end
print_message $stdout
```

### The big bad world

Here is a list of common dependencies you should *always* inject.

* `ENV` the variables you get from your environment (e.g. `ENV['PATH']` is `$PATH` from your shell)
* `ARGV` the arguments given to your program when you invoked it. e.g. `ruby your_program.rb --some-flag`, then `ARGV` would be `['--some-flag']`)
* `$stdin` the standard input stream, by default this is connected to your keyboard
* `$stdout` the standard output stream, by default this prints to your monitor (this is what you talk to when you call `puts`)
* `$stderr` the standard error stream, by default this also prints to your monitor, but it's for secondary information, like errors

[Here](https://github.com/JoshCheek/miniature-octo-ironman/blob/6d273d8ba5e0b8853053c0b86a991a57d085742f/bin/moi#L193-201)
is an example of some code I've recently been writing, ntoice that I inject all these varaibles and more.
This allows me to easily test them, and swap out the values when appropriate.

## Design based on Dependencies

### Dependencies point downwards

A general principle that will get you incredibly far is that dependencies should always point downwards.

You don't want things at the bottom to depend on things at the top,
or your ability to reason about your code will suffer,
your ability to use the code in different contexts will suffer,
your ability to change your code will suffer.

### Separate your code from the world

Code becomes difficult to test when it talks directly to the world.
If something calls `puts`, it is talking directly to the real standard output.
We see that in the [MessagePrinter](https://github.com/turingschool-examples/numbermind/blob/731af8c07d9f2f351ce3380ec214ec181dcd21cb/lib/message_printer.rb#L3).

If we tried to test our code, we would have difficulty.
It would start printing these messages all over our test output!
And how would we assert that it printed something?
We can't, because we aren't inbetween it and the standard output.

Instead, we should pass in the thing it is printing to. We'll see examples of that below.

### Move state as high as possible

Want to be able to easily test something? How about this code:

```ruby
def test_knows_correct_guess
  assert AnalyzeGuess.new("rrrr", "rrrr").correct?
  refute AnalyzeGuess.new("rrrr", "grrr").correct?
end
```

We removed the guess analysis from the middle of a game loop,
and now it's trivially easy to test it.

The game loop is state. We're in the middle of a loop.
That's hard to test. So by separating it out and having it depend on this code,
we can test this code utterly independently.

This code will also support a web-based interface,
because it does not assume it's in a loop or talking to a stream,
it's just pure logic.

### How to respond to test pain

When testing is difficult, draw the dependencies and look at what things are making it difficult.

Then figure out what you need to do to remove it.
Typically, that means moving the dependencies highter.
Often this is done by taking some variable you are initializing and having it be passed in.
Sometimes, it's by taking a responsibility and moving that responsibility up higher.

For example, what if our game did not have a loop?
How much easier would it be to test any one of these pieces?

### Considering a Mastermind Design

If we apply the ideas above, we can make it super easy to test mastermind,
because our design would change.

We might end up with a design like this:

```ruby
class Game
  attr_reader :sercet, :available_colors
  def initialize(secret, available_colors)
  def add_guess(guess)
  def last_guess
  def won?
  def valid_guess?(guess)
  def num_guesses
  def correct_colors
  def correct_positions
```

Which we could then use from the CLI like this:

```ruby
class CLI
  # ...

  def play_game
    secret = GenerateSecret.call(@available_colors)
    game   = Game.new(secret, @available_colors)
    loop do
      game.add_guess(input_stream.gets.chomp)
      break if game.won?
      output_stream.puts messages.incorrect_guess(game)
    end
    output_stream.puts messages.game_won(game, @available_colors)
  end
end
```

Want to add timing to that?

```ruby
def play_game
  start_time = Time.now
  # ...same logic ot play the game...
  stop_time = Time.now
  seconds_taken = stop_time - start_time
  output_stream.puts messages.game_won(game, @available_colors, seconds_taken)
end
```

## Last but not least

If you don't know what to do, then do the shitty thing that works.
Analysis paralysis will prevent you from making progress, and it's better to do something shitty than nothing at all.
Very often, doing the thing you know is shitty will get you far enough along the pat to see what is shitty about it and to then address that specifically.
And at the very least, you can see it show it to people and ask what they would do to improve it.

[Here](https://github.com/JoshCheek/seeing_is_believing/blob/1dcdbded221ebbd50e48d6690114d15b9ac0493e/lib/seeing_is_believing/binary/annotate_xmpfilter_style.rb)
is an example. Look at this giant pile of shit I wrote. "TODO"s littered everywhere.
Code that I can see belongs elsewhere. Giant piles of complex algorithmic crap.
But it gets my feature in place, and it lets me see what's wrong. It's a step on the staircase to better code.

So I don't worry about writing shit code, once its in place, I have something I can see and change. Get it working, then make it good.
