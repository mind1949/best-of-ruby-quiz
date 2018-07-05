# Madlibs (#28)

This week's Ruby Quiz is to write a program that presents the user with Madlibs. The script should ask the user for a series of words, then fill in the proper places in the story using the user's answers.

We'll keep our story format very simple, using a ((...)) notation for placeholders. Here's an example:

>  Our favorite language is ((a gemstone)).

If your program is fed that template, it should ask you to enter "a gemstone" and then display your version of the story:

> Our favorite language is Ruby.

That covers the simple cases, but in some instances we may want to reuse an answer. For that, we'll introduce a way to name them:

>  Our favorite language is ((gem:a gemstone)).  We think ((gem)) is better than ((a gemstone)).

With the above story, your program should ask for two gemstones, then substitute the one designated by ((gem:...)) at ((gem)). That would give results like:

> Our favorite language is Ruby.  We think Ruby is better than Emerald.

You can choose any interface you like, as long as person can interact with the end result. You can play around with my solution here:

[Ruby Quiz Madlibs](http://rubyquiz.com/cgi-bin/madlib.cgi)

And here are the two Madlib files I'm using, to get you started:

[Lunch Hungers](http://rubyquiz.com/madlibs/Lunch_Hungers.madlib)

[Gift Giving](http://rubyquiz.com/madlibs/Gift_Giving.madlib)

------

# Quiz Summary

Well, if nothing else these are a fun little distraction, eh? Actually, I was surprised to discover (when writing the quiz), how practical this challenge is. Madlibs are really just a templating problem and that comes up in many aspects of programming. Have a look at the "views" in Rails, for a strong real-world example.

Looking at the problem that way got me to thinking, doesn't Ruby ship with a templating engine? Yes, it does. We could use that to build our solution:

```ruby
 #!/usr/local/bin/ruby -w

    # use Ruby's standard template engine
    require "erb"

    # storage for keyed question reuse
    $answers = Hash.new

    # asks a madlib question and returns an answer
    def q_to_a( question )
        question.gsub!(/\s+/, " ")       # noramlize spacing

        if $answers.include? question    # keyed question
            $answers[question]
        else                             # new question
            key = if question.sub!(/^\s*(.+?)\s*:\s*/, "") then $1 else nil end

            print "Give me #{question}:  "
            answer = $stdin.gets.chomp

            $answers[key] = answer unless key.nil?

            answer
        end
    end

    # usage
    unless ARGV.size == 1 and test(?e, ARGV[0])
        puts "Usage:  #{File.basename(__FILE__)} MADLIB_FILE"
        exit
    end

    # load Madlib, with title
    madlib = "\n#{File.basename(ARGV[0], '.madlib').tr('_', ' ')}\n\n" +
             File.read(ARGV[0])
    # convert ((...)) to <%= q_to_a('...') %>
    madlib.gsub!(/\(\(\s*(.+?)\s*\)\)/, "<%= q_to_a('\\1') %>")
    # run template
    ERB.new(madlib).run
```

The main principal here is to convert ((...)) to <%= ... %>, so we can use Ruby's own template engine. Of course, <%= a noun %> isn't going to be valid Ruby code, so a helper method is needed. That's where q_to_a() comes in. It takes the Madlib replacements as an argument and returns the user's answer. To use that we actually need to convert ((...)) to <%= q_to_a('...') %>. From there, ERb does the rest of the work for us.

Now for simple Madlibs, you don't really need something as robust as ERb. It's easy to roll your own solution and most people did just that. Let's examine Sean E. McCardell's code:

```ruby
   class Madlib

      # Given the madlib text as a string, builds a list of questions and
      # a map of questions to "blanks"
      def initialize(txt)
        @questions = []
        @story_parts = []
        @answer_list = []
        @answers = []

        stored = {}

        txt.split(/\((\([^)]*\))\)/).each do |item|
          if item[0] == ?(
            item = item[1..-2].gsub("\n", ' ')
            if item.index(':')
              name, question = item.split(':')
              stored[name] = @questions.length
              @questions << question
            else
              name, question = item, item
            end
            @answer_list << (stored[name] || @questions.length)
            @questions << question unless stored[name]
          else
            @story_parts << item
          end
        end
      end

      # Calls a block with the index and text of each question
      def list_questions(&block)
        @questions.each_index do |i|
          yield(i, @questions[i])
        end
      end

      # Stores the answer for a given question index
      def answer_question(i, answer)
        @answers[i] = answer
      end

      # Returns a string with the answers filled-in to their respective blanks
      def show_result
        real_answers = @answer_list.collect {|i| @answers[i]}
        @story_parts.zip(real_answers).flatten.compact.join
      end
    end

    # Example that reads the madlib text from a file specified on the
    # command line

    madlib = Madlib.new(IO.read(ARGV.shift))
    answers = []
    madlib.list_questions do |i, q|
      print "Give me " + q + ": "
      answers[i] = gets.strip
    end
    answers.each_index {|i| madlib.answer_question(i, answers[i]) }
    puts madlib.show_result

```

The Madlib object handles the heavy lifting here. initialize() really does a lot of the work. It breaks the story down into an internal format which is primarily a list of @story_parts, @questions, and @answers. Since the answer to a question may be used in more than one place, an @answer_list is also built as a mapping between the actual answers and all their replacements.

You can see this chunking process in the bottom half of initialize(). It basically split()s the story around ((...)) replacement sections. The split() Regexp uses capturing parentheses to ensure that the replacements themselves are returned, in addition to the story parts.

Inside the iterator, the outer if branches to handle either questions (starting with a "(" character) or story parts. Each item is added to the correct list. Questions are also examined for the extra label and the stored Hash resolves these repeats as they occur.

The next method, list_questions(), provides iteration over the list of questions. (Note that the &block parameter isn't used in the method and could be removed.) The block is yielded an index and the current question. The index can be used to feed an answer to the sister method, answer_question(), which just stores answers.

The final method of the class, show_result(), uses the @answer_list map to construct a list of real_answers. That list is zip()ed with @story_parts to produce the final output.

The final chunk of code just puts the class to work. An object is constructed from the file passed as a command-line argument. Next, the code walks the questions, asking each one in turn and collecting answers. Those answers are passed to answer_question(), and the final results are printed. I believe you could do away with the extra Array in this section and simplify a little:

```ruby
    madlib = Madlib.new(IO.read(ARGV.shift))
    madlib.list_questions do |i, q|
      print "Give me " + q + ": "
      madlib.answer_question(i, gets.strip)
    end
    puts madlib.show_result
```

Well, there's a look at a couple of the solutions. Other solutions involved CGI, PDF output (very cool!), and even a little golf action. Don't miss looking over them.

My thanks to all the spongy Madlibers who took the time to fill out my fire-hose.

Tomorrow we'll use the quiz to start a new library for Ruby that will hopefully ease the ins and outs of common coding...