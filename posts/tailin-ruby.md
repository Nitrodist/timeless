Title: Tailin' Ruby
Subtitle: Implementing tail call optimizations in Ruby
Author: judofyr

Ruby doesn't implement [tail call optimization][tail-call], so a while ago I
tried to figure out how I could fake it:

    RunAgain = Class.new(Exception)
    def fib(i, n = 1, result = 0)
      if i == -1
        result
      else
        raise RunAgain
      end
    rescue RunAgain
      i, n, result = i - 1, n + result, n
      retry
    end
{: lang=ruby }

It was *extremely* slow (see the benchmark at the end of this post) since it
has to build backtraces for each exception, but hey: It worked! I was proud;
*very* proud. Well, until I discovered [this awesome snippet][catch-tail]:

    class Class
      # Sweet stuff!
      def tailcall_optimize( *methods )
        methods.each do |meth|
          org = instance_method( meth )
          define_method( meth ) do |*args|
            if Thread.current[ meth ]
              throw( :recurse, args )
            else
              Thread.current[ meth ] = org.bind( self )
              result = catch( :done ) do
                loop do
                  args = catch( :recurse ) do
                    throw( :done, Thread.current[ meth ].call( *args ) )
                  end
                end
              end
              Thread.current[ meth ] = nil
              result
            end
          end
        end
      end
    end
    
    class TCOTest
      # tail-recursive factorial
      def fact( n, acc = 1 )
        if n < 2 then acc else fact( n-1, n*acc ) end
      end

      # length of factorial
      def fact_size( n )
        fact( n ).size
      rescue
        $!
      end   
    end

    t = TCOTest.new

    # normal method
    puts t.fact_size( 10000 )  # => stack level too deep

    # enable tail-call optimization
    class TCOTest
      tailcall_optimize :fact
    end

    # tail-call optimized method
    puts t.fact_size( 10000 )  # => 14808
{: lang=ruby }

Magical stuff. Fast and sweet. My (failed) attempt was sent to /dev/null immediately...

## The Ruby Programming Language arrives

A few months later, I received my copy of [The Ruby Programming Language][rpl]
and started reading it. Suddenly, on page 151:

> <cite>The Ruby Programming Language, page 151, 5.5.4 redo:</cite>
> The `redo` statement restarts the current iteration of a loop or iterator.
> This is not the same thing as `next`. `next` transfers control to the end of
> a loop or block so that the next iteration can begin, whereas `redo`
> transfers control back to the top of the loop or block so that the iteration
> can start over. If you come to Ruby from a C-like language, then `redo` is
> probably a new control structure for you. 

I’ve used Ruby for a long time, still I’ve never heard of redo... Here’s an
example from the book:

    puts "Please enter the first word you think of"
    words = %w(apple banana cherry)
    response = words.collect do |word|
      # Control returns here when redo is executed
      print word + "> "               # Prompt the user
      response = gets.chop            # Get a response
      if response.size == 0           # If user entered nothing
        word.upcase!                  # Emphasize the prompt with uppercase
        redo                          # And skip to the top of the block
      end
      response                        # Return the response
    end
{: lang=ruby }

So I started thinking: The reason I used raise/rescue in my implementation was
because I needed the `retry`-keyword... Maybe... What if?

    def fib(i, n = 1, result = 0)
      if i == -1
        result
      else
        i, n, result = i - 1, n + result, n
        redo
      end
    end
    
    fib(10000)
{: lang=ruby }

Unfortunately, "The `redo` statement restarts the currentiteration of a *loop*
or *iterator*", so it only throws a LocalJumpError. However, don't forget that
we're dealing with Ruby:

    ### Everything is possible in Ruby!

    define_method(:acc) do |i, n, result|
      if i == -1
        result
      else
        i, n, result = i - 1, n + result, n
        redo
      end
    end

    def fib(i)
      acc(i, 1, 0)
    end

    fib(10000)  # Yeah!
{: lang=ruby }

## Native Support in 1.9.2

It turns out that there's *yet* another way to achive tail call optimization in
Ruby: Ruby 1.9.2 actually ships with native support, but it's disabled by
default. You can either enable it through a compile-flag or by specifying the
compile options at runtime (thanks a lot to Roman Semenenko for showing me this
trick!):

    RubyVM::InstructionSequence.compile_option = {
      :tailcall_optimization => true,
      :trace_instruction => false
    }
    
    RubyVM::InstructionSequence.new(<<-EOF).eval
      def acc(i, n, result)
        if i == -1
          result
        else
          acc(i - 1, n + result, n)
        end
      end

      def fib(i)
        acc(i, 1, 0)
      end
    EOF
{: lang=ruby }

Notice that the compile options will only be used for the code you pass into
InstructionSequence, not any regular code. Also be aware of that these tail
call optimization are considered experimental and there might be bugs related
to them (like [#4082][4082]). However, they perform pretty good (see the
TCO-graph below).

## The Benchmark

So, how fast is it? Take a look below:

<iframe src="http://judofyr.github.com/recursive/" style="width:660px;height:460px;border:0">&nbsp;</iframe>

* You can zoom by marking in any of the graphs.
* When you un-tick a graph, mark in the lower graph to zoom in.
* Un-tick everything else than "Redo" and "Iterative" and notice how close they are to each other.
* Look how bad "Rescue" performs,
* and how early "Regular" fails.
* All the code is available at [GitHub][source]

Nifty, eh?

[tail-call]: http://en.wikipedia.org/wiki/Tail_call_optimization
[catch-tail]: http://blade.nagaokaut.ac.jp/cgi-bin/scat.rb/ruby/ruby-talk/145593
[rpl]: http://www.amazon.com/Ruby-Programming-Language-David-Flanagan/dp/0596516177
[source]: http://github.com/judofyr/recursive
[4082]: http://redmine.ruby-lang.org/issues/show/4082

