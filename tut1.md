<link href="http://kevinburke.bitbucket.org/markdowncss/markdown.css" rel="stylesheet"></link>

OMeta Tutorial
==============

Part 1 - Basics
---------------

### What Is It? ###

[OMeta][0] is a language for pattern-matching, somewhat similar to a
[parser generator][1]/[PEG][2] interpreter, only more powerful. There are several OMeta
implementations available, in this tutorial we'll be focusing on the Javascript implementation
of OMeta, OMeta/JS.

### A Simple Example ###

Okay, so that's all a bit abstract. Let's dive in and look at an example:-

    ometa Integer {
        Dig =
            '0' | '1' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9',
        Process =
            Dig Dig*
    }

    Integer.matchAll('42', 'Process');

Try it - copy and paste the above code into the [live OMeta workspace][3], highlight the code
and select 'print it' (or hit ctrl+p). The value '[2]' should be inserted into the workspace
(more on this result later).

What does this mean?

We write OMeta code inline with code in the host language (in our case, Javascript), defining a
grammar __Integer__ which expresses the pattern we want to match. OMeta translates this into a
Javascript object which we can use to match our pattern against input.

A key method provided by the generated object is __matchAll__. A typical usage of this method
is as follows:-

    ometaObject.matchAll(objectToMatchAgainst, 'nameOfRule');

Typically we define a 'toplevel' rule in our grammar which encompasses the overall pattern
we're matching against, in our example this rule is __Process__ which represents a valid
integer.

In our bid to understand the answer to the meaning of life, the universe and everything we
want to match __Integer__ against the string '42', so we end up with:-

    Integer.matchAll('42', 'Process');

### OMeta Grammars ###

Ok, so now we know how to *use* our grammar, let's take a look at the contents of the grammar
itself.

We define grammars using the following syntax:-

    ometa NameOfGrammar {
        ...
    }

Within the grammar we define *rules*. These rules express the pattern we are attempting to
match against.

Each rule consists of a name or *non-terminal*, followed by a *parsing expression*. We can
define as many rules as we like in a grammar, separated by commas, e.g.:-

    ometa NameOfGrammar {
        rule_1   = parsing_expression_1,
        rule_2   = parsing_expression_2,
        ...
        rule_n-1 = parsing_expression_n-1,
        rule-n   = parsing-expression-n
    }

In our example we define two rules, __Dig__ and __Process__. But what makes up each rule's
parsing expression?

Let's start with __Dig__:-

        Dig =
            '0' | '1' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9'

Here we specify that __Dig__ expects to see one of the range of characters '0' to '9'\*. This
demonstrates two possible components of a parsing expression - a character 'X' which literally
matches 'X' in the input, and *prioritised choice*, i.e. the __|__ operator.

Prioritised choice simply means Ometa iterates through the various possibilities and matches
against the first entry which fits the pattern. In this case, it iterates through the digits
0-9 until it finds the digit its looking for.

As an aside, prioritised choice is an idea taken from [PEG][2]s, which differs somewhat from
traditional parser frameworks such as [LALR][4], however that discussion is decidedly
off-topic.

Moving on to our second rule, __Process__, we encounter three new members of Ometa syntax,
namely *sequencing*, the *zero-or-more* operator (sometimes known as the [Kleene star][5]), and
*rule application*:-

    Process = Dig Dig*

Rule application is the use of rules within a rule. You can make *recursive* references,
i.e. references to the rule you are defining, but it is important to avoid so-called
[left recursion][6] - recursively referencing a rule in the left-most subexpression. The
current implementation of OMeta/JS cannot do this (however it does not result in an infinite
loop, rather it detects the situation and simply causes the rule to fail.)

Sequencing simply refers to stringing together a sequence of subexpressions to match against. In this
case we are saying that our __Process__ rule matches our __Dig__ rule, followed by our __Dig__
rule again, only this time followed by a star.

What is this second component? If you're at all familiar with [regular expressions][7] then it
will be familiar. The second component refers to our __Dig__ rule, then applies the
zero-or-more star operator which, as you might expect, indicates that we expect to see
zero-or-more __Dig__'s.

So what does this rule express overall? It states that an integer is represented by one digit
followed by zero or more digits, in other words - one or more digits.

As you might expect there is an operator available for this (as in regular expressions), the
*one-or-more* operator, __+__ so a sensible refactoring of this rule would be:-

    Process = Dig+

### Matchmaking ###

Let's look at our example program again:-

    ometa Integer {
        Dig =
            '0' | '1' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9',    
        Process =
            Dig Dig*
    }

    Integer.matchAll('42', 'Process');

We've talked about every part of this code, but we've missed something fundamental - what are
we trying to do here? The answer in this case is that we are simply checking whether our input
matches the expected pattern.

The method __Integer.matchAll()__ returns the *result* of our parse, in the case of our grammar
this is the last matched subexpression, i.e. __Dig*__.

Try the following in the [workspace][3]:-

    Integer.matchAll('421', 'Process');

The result this time is [2, 1], i.e. an array of values 2, 1. Again this is what was matched by
__Dig*__. Now try something invalid:-

    Integer.matchAll('xyz', 'Process');

You should see a popup alert indicating that the match failed.

### Semantic Actions ###

Our grammar isn't all that useful right now, so let's make it actually do something:-

    ometa Integer {
        Dig =
            '0' | '1' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9',    
        Process =
            Dig:d Dig*:r -> [d].concat(r)
    }

    Integer.matchAll('4210', 'Process');

The -> operator denotes *semantic actions*. These are actions (i.e. code in the host language)
which are executed when the rule is matched.

Note that we are assigning variable names to sub-expressions in our parsing expression by
suffixing them with a colon followed by the name. This enables us to reference the results of
these sub-expressions in our semantic actions and actually do things with these values.

In __Process__ we match the first digit and the zero-or-more digits which follow it. The
repetition operators return arrays of matched values. We then simply concatenate the first
digit with the digits which follow it. Each __Dig__ simply returns the result of its last
subexpression, which will simply be its digit.

If you go ahead and try this code in the [workspace][3] you'll see that it returns [4,2,1,0].

### Improving Matters ###

Note that OMeta has the __digit__ rule built in which matches individual digits, so we can
drastically simplify our grammar to:-

    ometa Integer {
        Process =
            digit+
    }

    Integer.matchAll('4210', 'Process');

And if we wanted to actually retrieve the integer as a number type, we could write:-

    ometa Integer {
        Process =
            digit+:n -> parseInt(n.join(''))
    }

    Integer.matchAll('4210', 'Process');

Note that it's a good idea to enclose complicated semantic actions in braces if they are
non-trivial, and especially if they contain arithmetic, as OMeta has a particular problem
parsing semantic actions which contain it, e.g.:-

    ometa IntegerPlusOne {
        Process =
            digit+:n -> (parseInt(n.join(''))+1)
    }

    Integer.matchAll('4210', 'Process');

Note that the parentheses can be of the round or curly variety here, it's a matter of which you
prefer.

### Notes ###

\* In Javascript there is no such thing as a separate character type, rather characters are
simply strings of length 1.

[0]:http://tinlizzie.org/ometa/
[1]:http://en.wikipedia.org/wiki/Parser_generator
[2]:http://en.wikipedia.org/wiki/Parsing_expression_grammar
[3]:http://www.tinlizzie.org/ometa-js/#foobarbaz
[4]:http://en.wikipedia.org/wiki/LALR
[5]:http://en.wikipedia.org/wiki/Kleene_star
[6]:http://en.wikipedia.org/wiki/Left_recursion
[7]:http://en.wikipedia.org/wiki/Regular_expressions
