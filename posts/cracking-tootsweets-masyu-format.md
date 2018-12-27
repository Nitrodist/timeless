Title: Cracking TootSweet's Masyu Format
Author: judofyr

> <cite><a href="http://tootsweet.com/masyu">TootSweet's Masyu:</a></cite>
> *Masyu* is played on a grid. It has a simple goal: to draw a single
> non-intersecting loop through all of the circles in the grid. The Masyu grid
> contains two kinds of circles.  Each adds a constraint to the path of the
> loop.

Masyu is a type of logic puzzle designed by the same guys behind Sudoku.
Needless to say, it's very fun and extremely addicting. Especially when you
play it on your iPoid, thanks to TootSweet:

![TootSweet Masyu](http://timeless.judofyr.net/masyu/app.png)

As as programmer, I didn't like the thought of all those beautiful, hand-made
puzzles should be stuck on the iPod, so I wanted to see how TootSweet stored
the puzzles.

## Step 1: Find the puzzles

I've already jailbroken my iPod, so after SSH-ing in, a simple `find
/User/Applications/ -wholename *MasyuBug.app` revealed where the files to
MasyuBug.app were stored:

![MasyuBug.app](http://timeless.judofyr.net/masyu/terminal.png)

And right there, in a SQLite-file called `Documents/Puzzles.data` I found the
puzzles. Just seconds later I got the schema:

    CREATE TABLE properties ( 
      key text primary key,
      value text default null);

    CREATE TABLE puzzles ( 
      id integer primary key autoincrement,
      puzzle text,
      name text default null, 
      size integer default 0,
      volume integer default 0,
      board text default null,
      checkpoint text default null,
      status integer default 0,
      solution_date default null);

Let's have a look at the puzzles table:

    sqlite> SELECT id,puzzle,name,board,status FROM puzzles LIMIT 3;
     id | puzzle     | name    | board | status
      1 | 4:4:AgAQAA | Ladybug | ww:gq | 2
      2 | 4:4:CCQQQA | Aphid   | AA:IC | 1
      3 | 4:4:gAAECA | Cicada  | AA:AA | 0    

That seems very cryptic at the moment...

## Step 2: Let's play with the data!

Here comes the funny part: Updating the fields and see how the puzzles (in the
app) changes. I quickly verified that:

* The `puzzle` field contains the width, height and the puzzle itself.
* The `board` field contains the lines you've drawn.
* Status: 2 = solved, 1 = started, 0 = not started

The puzzle itself didn't really made any sense (AgAQAA?), so I started changing
the letters and once again noticed how MasyuApp displayed the puzzle:

* AAAAAA = empty board
* BAAAAA = white circle in top-left corner
* ABAAAA = the white circle moved three squares right
* AABAAA = the white circle moved three squares right (wrapped by the 4×4-board)
* AAABAA = the white circle moved another three squares right

Obviously, each letter represented three squares, but exactly how? After 52
UPDATE’s I ended up with this table:

![Puzzle table](http://timeless.judofyr.net/masyu/table.png)

Can you spot how TootSweet's converts letters to circles? It took me a while,
but suddenly I realized it: it’s base-4. It's in fact reversed base-4 where 0
and 3 = blank, 1 = white and 2 = black. Let me show you two examples:

The letter S is in the 18th position of our table (starting from zero), and 18
is 102 in base-4. Reverse it and you'll get 201: One black, one blank and one
white.

The letter X is in the 23th position, and 23 is 113 in base-4. Reverse it and
you'll get 311: One blank, two whites.

Ruby version:

    ALPHA = ('A'..'Z').to_a + ('a'..'z').to_a
    def letter(l)
      num = ALPHA.index(l)
      num.to_s(4).rjust(3, '0').reverse.split('').map{ |x| x.to_i % 3 }
    end
{: lang=ruby }

## Step 3: The Lines

Now let's see how TootSweet stores the lines you've drawn. I started drawing
lines in the app and watched how the `board` field changed in the database:

* AA:AA = no lines
* The first two letters changed when I draw a horizontal line
* The last two letters changed when I draw a vertical line
* The first letter changed when I draw a line on one of the six first places.
* Same with the second

It made sense that each letter represented six lines. And, since I now know how
TootSweet rolls, it made even more sense that it's reversed base-2. However,
when storing the circles, TootSweet only used 52 letters (A-Z and a-z). In
order to represent every possibilty for six lines, we need 64 (2 ** 6) letters.
After some drawing on the iPod, it turned out that 0-9 and + and / where the
missing letters TootSweet used:

0 is in the 52nd position of our new table (starting from zero), and 52 in
base-2 is 110100. Reversed it's 001011 and it would look like this (if we
assumed that the 0 was in the first letter):

![Lines](http://timeless.judofyr.net/masyu/numbering.png)

Ruby version:

    ALPHA = ('A'..'Z').to_a + ('a'..'z').to_a + ('0'..'9').to_a + %w(+ /)
    def line(l)
      num = ALPHA.index(l)
      num.to_s(2).rjust(6, '0').reverse.split('').map { |x| x.to_i }
    end
{: lang=ruby }

## That's it!

And there you have TootSweet's Masyu format. I reengineered this just as a
challenge, and it was a lot of fun. However, if you ever want to write Masyu
application, *don't* steal TootSweet's hand-made masyus. I'm sure they've spent
a lot of time on this, and beside, no-one wants to solve the same puzzle twice.

What you *could* use this information to, is importing puzzles from people's
iPhones, so they can continue to solve the puzzle on their computer. Or, you
can store your own puzzles in this format and this can end up as the standard
way to store Masyu puzzles. Even though it's a bit cryptic, the puzzles are
quite small when they're stored this way, and can then easily be shared over
IRC and forums.
