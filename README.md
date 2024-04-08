# selection-path
A tool for generating complex randomized decision structures with flow charts to provide organic selection while still allowing for tight constraints.

### Basic components:
- Block that will choose true or false based on how many times the block has been called since the last time true has been chosen  
  - can define a function, but must define a max where true = 100%
- Block that will choose true or false randomly each time by a weight
- Block that will choose true or false randomly, with weights adjusted each time to enforce target distribution
  - has param for the max amount it can adjust the weight so it doesn't have overly large influence
  - has param for the range of error used to scale the weight of the adjustment
    - so it can make larger adjustments when the distribution is far off
  - has param for the size of the distribution, can choose to target the right distribution over 10 selections or 50
- Block that randomly selects a point under a curve
  - Can output x or y
- Block that selects a path based on an input point
  - define ranges for each path
  - define function/range to sample from around the input point
    - so it could select +-15% from the input with a normal distribution or you could make the range 0 and make this just a binary selection based off the input
- Block that randomly selects a point under a curve, but the curve can have input parameters to change it's shape
- Wave generator
  - outputs values of a sin wave
  - parameter for period, number of activations before repeating
  - parameters for range, the amplitude of the function and values at the peak/valley
- super wave generator
  - normal wave generator has no inputs and is meant to be global
  - this wave generator can be set to only tick forward on events or block making it past a certain point unless a certain event has happened this cycle
- Block that provides an output based on the number of real days since that last time a path was taken
- Blocks that stretch functions to change the range
- blocks that represent a selection from a set
  - not sure how to word this rn but most of these blocks are very micro like setting up an individual function but i want a block that selects 1 specific category based on the number of times each category has been selected or amount of time since each was last selected
  - so this is basically like a bunch of blocks each configured identically that represent different items
  - best example is imagining 12 blocks each doing a weight adjustment based on the number of runs since their last selection vs 1 block 

### How it should work:
The final step of each decision is represented as the number selected on a number line
this number line includes bars drawn to represent the ranges required to get each path
above this number line is a graph of the function used for the selection
to make a selection we randomly choose a point in the area under the function line
- so y=1 will give outputs evenly distributed on the range
- the graph of a normal distribution will give normally distributed outputs with the median at the center of the range
- y=0.5 or y=10 should give the same results as y=1

we have blocks to represent matrixes that split an input path and then join output paths
- so if you want to select from a 2d matrix with a different function for x and y you can split the input path into 2, put each path into a normal selection block, and then for a 5x5 you feed the 5 x paths and the 5 y paths into the matrix output block to get 25 paths
    - so it's basically a demultiplexer
      - not sure about the uses of a multiplexer
        - i could see using a multiplexer in order to launch a path to send a bunch of events or set a bunch of values if 1 specific row/path is chosen or something

there are 2 ways to change a percentage, change the function or move the fence

we have individual blocks for using the function generator to make a function

generated functions can be used as scalers, they take an x input and give the y output

we have blocks to combine generated functions
- they can be added together
- they can be multiplied
- they can be used as different axes of a matrix selection

when combining functions, the effect of each input function on the composition can be weighted

every function shows a graph that is colored to show the overall effect of each function
- so each function will be represented by a color
- and any areas under the curve that were added by a function will be that color
- and areas over the curve removed by a function will show crosshatches of that color

the most important goal is making it easy to understand the flow of the logic and how different functions come together
ideally we want to make it so that each transformation is done in a way that the order of the operations doesn't matter
- i have no clue if this is mathematically possible
we want to build everything in a way that makes functionality consistent and the impact of state obvious
and to do that we need to be able to represent the impact of each function graphically
and we need to be able to visualize higher dimensional data

really we can think about selections in 2 parts, the individual nodes representing all the different steps and filters required for 1 selection, and the macro groups of blocks representing the path from one selection to another

any individual part of the selection graph is called a block
- this includes standalone components like a global sin generator or a random number generator
any group of blocks which has an input to trigger it to run and multple output paths which can be activated by it's selection is called a node

every node is automatically turned into a custom block to use in more complex designs

so a node can contain other nodes, and you can take more complex steps in order to reach a selection

the overview only shows branching nodes. selection paths which just set global values or pass data are collapsed

each block is a contained unit with a class that handles all of it's internal state
- so no more bs trying to manage all the counters tracking the number of runs since they've been activated

### elements of a block

inputs:
- input parameters
  - these are values that change the behavior of a block, like an x value input for a function
  - these are meant to be temporary/local. the block will always use whatever parameters were given when the activation signal was sent. so reading blocks without an activation signal makes no sense
  - the idea is that blocks should work more like pure functions if they're not using any internal state and not referencing any global state
- scoped parameters
  - these are parameters driven by state
  - this is for stuff like driving a selection based on the total times a category has been run
  - this also would be how we would refer to stuff like adjusting the fence of a selection using a wave generator
  - really these are the same as input parameters but they should me distinct
    - parameters going directly from 1 block to another should be input params
    - params going from 1 to many blocks should generally be scoped params
    - scoped params are often represented visually by giving blocks which just output a copy of the main block, so you can have them in many places
  - they're called scoped params because their scope is above local but they don't necessarily have to be global
    - each param has it's own defined scope
    - by default the scope is inside the current node
    - nodes can be grouped to share scope/each parameter has a list of nodes it's scope can reach
    - so you can use a param across multiple blocks and for different things and set where it applies
- input signals
  - these are events which drive internal state
  - these do not cause the whole block to activate
  - they do activate the blocks they need to
  - this is for stuff like adjusting based on number of runs
    - the input signal can connect to the activation for a counter while the activation signal can connect to the counter reset
- activation signals
  - these represent a specific node being selected

 outputs:
- possible signals
  - selection signal
    - a signal which can be used as activation or input for future blocks
  - not selected signal
    - signal to drive events when a path is not selected, for ticking counters and stuff
      - this needs to be it's own signal rather than using global counters for stuff like "runs since" counters
      - that way we can define scope
  - flag signals
    - signals sent based on internal state
    - like send a signal when it's been more than 10 runs
  - output signals
    - signals sent based on conditions that aren't part of internal state
    - so this would be like intermediate random selections or selections based on input params
- possible output values
  - a number
  - a multidimensional point
  - a vector(idk why)
  - a function
    - this is basically a whole visual programming language for creating composite functions
    - so passing around and transforming math functions is like 90% of what we do
  - the output of a function
  - some value from internal state
  - a python object, function, file, config, or dictionary, that can be used to invoke the selection. This is how you define the interface to connect the selection to the rest of the program.
 

### combining functions:
functions can be combined in many ways
you can do arithmetic or compose them
arithmetic is easy, `f(a+b) = f(a) + f(b)`
if we add, sub, mul, or div we can easily just follow order of operations and know that changing the order of selection won't change anything
but composing a function `f(c)=fa(fb(b))` depends on the order
we're going to end up composing functions as users ourselves anyway
  like it's so easy to just plug an output into an input
so we should give blocks to help compose functions and give helpful insight

you could take a function that outputs a fancy curve and create a new function by putting a sin wave function into the fancy function as an input and then you will have composed a function which draws that fancy curve periodically

you could multiple a function by the output of a sin wave shifted up to make the intensity of the output of the function cycle over time
- so you could do this to benchmarks to make them organically have times where they're very common and times where you rarely benchmark
  
you could add a sin wave to a function and then give it a floor to increase or decrease the output of the first function based on where in the cycle the sin wave is
- i think this is just a worse version of multiplying, addition doesn't make as much sense for comining a sin wave

we can detect function composition to an extent
- at least on the level of a node we can track this
- any chain of functions in a node can be displayed in the composition editor, where you can view all the different possible compositions and combinations of those functions.
- we'll see how far we can push our ability to parse the tree and visualize many massive changes
  - we need to see what happens if we change the order of functions in each node, and if we change the order of the nodes
  - we don't change functions across nodes, which helps a lot
  - actually no it doesn't. in either case it is n^2
  - i don't want to look throuhg n^2 graphs
  - it's not possible to test everything, so our best bet is to make dynamic systems and really rely on the functions that enforce a specific distribution

# Simulation
A good selection simulator is going to be crucial.
Every external input into the selection algorithm is represented by it's own block
that block has a set of configuration parameters defined which ensure type safety and either clamp the input to a specified range , refuse to handle input outside of that range, or send a signal to trigger a different path.

so really each input block is pretty much just a normal output block

it follows a path based on the type/state of the input
- (with wrong types sending activation to a path that sends an error activation signal)
- so you can get that signal from your program and handle it if needed
- can define err states

depending on how that goes it will send activation signals to pass an input value through a function to scale and clamp the output

then in addition to passing the value of the output it will also send activation signals depending on what range the output was in
- a normal input block will just always send an activation signal when it is passed a value, so if it is given a value that's defined in the accepted range then it will activate that signal

the logic of handling this input is given to the user
- interfaces will be defined which the selection program will expect to be implemented by the user
- there are interfaces for data and interfaces for function calls
- you just have to define the names of the inputs
- there are activation signals for calling a block, setting a value, getting a value, making a selection, and just about any other way that the user could call the lib
- this is all literally just a class. each input is it's own object. that class has a lot of flexibility when you instantiate it, and will set up all of the methods and parameters you need for your interface
- the custom methods of the class have 4 parts
  - an activation signal when the method is called
  - the data the method is passed
    - this is always stored as internal state to the class
    - so you can go and read the data that a method was passed last even if the method's activation signal isn't currently going and the params weren't directly saved
  - a block which will represent the return value of the method
  - a block which generates a signal on return
    - this way you can connect this return signal to the signals output interface along with the value so your program can process the result in an event driven manner
- all parameters/inputs should be changed/viewed using the set/get methods provided
  - these methods give signals when they are called
  - there are methods/options to set/get a parameter silently
    - there are blocks that provide a signal when they're called if you want to do something when a private method is accessed incorrectly
  - might implement custom setters/getters so that private attributes are actually private
type enforcement gets it's own block at the very end of the input chain
before that you're expected to do some of your own type checking
so you'll have blocks to check the type and shape of data to call your own signals based on the input
maybe there will be like a switch block at the start instead that defines the paths for types


##### simulated input interfaces
as described before, every input to the algorithm is defined in an interface and given a block to represent that interface as well as hooks to use it in your code.
in order to generate large amounts of data to see the performance of the algorithm you can simulate these inputs
to run a simulation you edit the input block and then from there you edit the input interface
this edit mode only has one block, the output
it is explicitly stated that when running in production the output of the interface block will just be whatever data it gets from the code and blocks on this screen will be ignored
but if you run the algorithm in simulation mode then the outputs of these interface blocks will be whatever you get from this code
by default it is just a function which gives a uniformly distributed output across the defined input range
you can do all the normal rng flowchart stuff to build your tests!~






















