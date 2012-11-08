# Introduction

This library implements a tuple store with a query and subscribe mechanism.
A subscribe is effectively a standing query which executes a block whenever
a newly added tuple together with the store's content fullfills the filter
specification.

The store holds same sized tuples with each value being assigned a name and
type at creation to support varous convenience functions to create and retrieve
tuples.

The following code snippet creates a simple RDF store and adds a few triplets
to it.

    store = OMF::Rete::Store.new(:subj => String, :pred => String, :obj => Object)
    store.add('myFridge', 'contains', 'milk')
    store.addNamed(:subj => 'myFridge', :obj => 'broken', :pred => 'status')

A filter consists of an array of tuple +patterns+ and a +block+ to be called when the store
contains a set of tuples matching the +pattern+.

The following filter only looks for a single, specific tuple. The supplied block is called
immediately if the tuple already exists in the store, or when such a tuple would be added at a later
stage.

    store.subscribe(:report_problem, [
      ['myFridge', 'status', 'broken']
    ]) do |m|
      puts "My fridge is broken"
    end

The following filter contains two +patterns+ and therefore both need to be matched at the same
time in order for the block to fire. Note, that the order these tuples are added to the store
or the interval between is irrelevant.

    store.subscribe(:save_milk, [
      ['myFridge', 'status', 'broken'],
      ['myFridge', 'contains', 'milk'],
    ]) do |m|
      puts "Save the milk from my fridge"
    end


So far the filter pattern were fully specified. The <tt>:\_</tt> symbol can be used as a wildcard identifier.
The following code snippet reports anything which is broken.

    store.subscribe(:something_broken, [
      [:_, 'status', 'broken']
    ]) do |m|
      puts "Something is broken"
    end

Similar to OMF::Rete::Store#addNamed we can describe a pattern with a hash. Any value not named is automatically
wildcarded. Therefore, an alternative represenation of the previous filter is as follows:

    store.subscribe(:something_broken, [
      {:pred => 'status', :obj => 'broken'}
    ]) do |m|
      puts "Something is broken"
    end

The +match+ argument to the block holds the context of the match and specifically, the tuples involved
in the match.

    store.subscribe(:something_broken, [
      [:_, 'status', 'broken']
    ]) do |match|
      what = match.tuples[0][:subject]
      puts "#{what} is broken"
    end

<tt>match.tuples</tt> returns an area of tuples one for each pattern. The matched tuple for the first pattern is at index 0,
the second one at index 1, and so on. Individual values of a tuple can be retrieved through the initially declared
value name (see OMF::Rete::Tuple#[]).

Let us assume we are monitoring many fridges, so if we want to report broken ones with milk inside, we need to ensure
that the +subject+ in both patterns in our second example are identical. Or in more technical terms, we need to +bind+ or +join+
values across patterns. A binding variable is identified by a symbol with a trailing <b>?</b>.

    store.subscribe(:save_milk, [
      [:fridge?, 'status', 'broken'],
      [:fridge?, 'contains', 'milk'],
    ]) do |match|
      fridge = match[:fridge]
      puts "Save the milk from #{fridge}"
    end

<tt>match[bindingName]</tt> (without the '?') returns the value bound to <tt>:fridge?</tt> for this match.
Obviously <tt>match.tuples[0][:subject]</tt> will return the same value.

# Functions

Pattern matches alone are not always sufficient. For instance, let us assume that we have also stored the age in years
of each monitored fridge and want to replace each broken one which is older than 10 years. To describe such a filter
we introduce functions (or what in SPARQL is refered to as a FILTER) which allow us to restrict bound values.

Functions are identified by the <tt>:PROC</tt> symbol in the first position of a pattern, followed by the function
name, and the list of parameters. Effectively, a function filters the values previosuly bound to a variable to those
for which the function returns true.

    store.subscribe(:replace_old_ones, [
      [:fridge?, 'status', 'broken'],
      [:fridge?, 'age', :age?],
      [:PROC, :greater, :age?, 10]
    ]) do |match|
      puts "Replace #{match[:fridge]}"
    end

<b>Design Note:</b> A more generic solution based on a 'lambda' is most likely cleaner. This is effectively
identical to the final block, except that the block should return +true+ for tuples passing the filter,
and +false+ for all others. To further simplify this and also reduce the search space, we can define a
+filter+ function which takes a list of bound variables and calls the associated block with specific bindings.

    store.subscribe(:replace_old_ones, [
      [:fridge?, 'status', 'broken'],
      [:fridge?, 'age', :age?],
      filter(:age?) do |age| age > 10 end
    ]) do |match|
      puts "Replace #{match[:fridge]}"
    end

# Set Operators

Let us assume we want the store to not only reflect the current facts but the entire history of a system. We
can achieve that by adding a timestamp to each fact and never retract facts.

    store = OMF::Rete::Store.new(:subj => String, :pred => String, :obj => Object, :tstamp => Time)

This now allows us to capture that a fridge broke on a specific date and was fixed some times later.

    store.add('myFridge', 'status', 'broken', '2008-12-20')
    store.add('myFridge', 'status', 'ok', '2008-12-22')

However, how can we now determine that a specific fridge is CURRENTLY broken? The pattern
<tt>[:f?, 'status' 'broken']</tt> will identify all fridges which are currently broken, as well as those
which broke in the past but are ok now. What we need is a way to describe sets and a filter to select a single tuple
from each set. In our example, each set would contain all the status messages for a specific fridge, while
the filter picks the one with the most recent timestamp.

The current syntax achieves this through special match values. For instance, <tt>:LATEST</tt> for <tt>Time</tt>
types picks the most recent fact.

    [:fridge?, 'status', :_, :LATEST]

To find all currently broken fridges we need to bind this to all broken status facts.

    store.subscribe(:broken_lately, [
      [:fridge?, 'status', :_, :LATEST],
      [:fridge?, 'status', 'broken']
    ]) do |match|
      puts "#{match[:fridge]} is broken"
    end

<b>Design Note:</b> This seems to be a fairly ad-hoc syntax. Is there a better one? This assumes that there is no join
on any of the bound variables, they are simply keys for the sets. But overloading functionality always adds complexity.

# Negated Conditions

Now let us consider we know that our fridge is broken and we want to monitor any future status updates.
There may be many different status types and we are interested in all of them as long as they are
different to 'broken'. In other words, we need a way to describe what is refered to as a 'negated
condition' and is defined by a leading <tt>:NOT</tt>, followed by one or multiple patterns describing
what should NOT be in the store.

    store.subscribe(:find_latest, [
      ['My Fridge', :status, :_, :LATEST],
      [:NOT, ['My Fridge', 'status', 'broken']]
    ]) do |match|
      puts "Status for my fridge changed to '#{match.tuples[0][:obj]}."
    end

Please note that the above example fails to report when my fridge is reported as broken again.

# Implementation

