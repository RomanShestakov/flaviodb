FlavioDB
========

Setup rebar riak_core template
------------------------------

.. code:: shell

    git clone https://github.com/basho/rebar_riak_core.git
    cd rebar_riak_core
    make install

Create project template
-----------------------

.. code:: shell

    mkdir flaviodb
    cd flaviodb

    # download rebar and set executable permissions
    wget http://cloud.github.com/downloads/basho/rebar/rebar && chmod u+x rebar

    # create project from riak_core template with app id set to flavio
    ./rebar create template=riak_core appid=flavio

Update riak_core version to 2.0.0
---------------------------------

we change the version on deps, riak_core and lager here:

https://github.com/marianoguerra/flaviodb/commit/6ae8f8b83c3d15346ea35a586dbe624aa3de6967

note we also updated lager version to 2.0.3 and removed the warnings_as_errors
flag on the last line.

Trying to build it
------------------

.. code:: shell

    make rel

if you are using Erlang 17 you will get some errors while compiling the
dependencies, the way to fix them most of the time is pretty ugly but I don't
know a better way.

the solution is to remove the warnings_as_errors config flag on rebar.config
directly on the rebar.config for that dependency, I did a script you can run
to fix this, run it once after you fetched the dependencies:

.. code:: shell

    ./util/fix_deps_warnings_as_errors.sh

if you ran the command above try building again:

.. code:: shell

    make rel

success!

now what?

Running a node
--------------

now that we managed to build it let's start a node:

.. code:: shell

    ./rel/flavio/bin/flavio console

but what can we do with it? well we can ping it:

.. code:: erlang

    (flavio@127.0.0.1)1> flavio:ping().
    {pong,1210306043414653979137426502093171875652569137152}

now you have a distributed, scalable and fault-tolerant ping service!

The road of the ping
--------------------

now that we have the basic riak_core project running let's follow the ping on
it's way from call to response.

it's entry point and public api is the flavio module, that means we have to
look into flavio.erl:

.. code:: erlang

    -module(flavio).
    -include("flavio.hrl").
    -include_lib("riak_core/include/riak_core_vnode.hrl").

    -export([ping/0]).

    -ignore_xref([ping/0]).

    %% Public API

    %% @doc Pings a random vnode to make sure communication is functional
    ping() ->
        DocIdx = riak_core_util:chash_key({<<"ping">>, term_to_binary(now())}),
        PrefList = riak_core_apl:get_primary_apl(DocIdx, 1, flavio),
        [{IndexNode, _Type}] = PrefList,
        riak_core_vnode_master:sync_spawn_command(IndexNode, ping, flavio_vnode_master).

we see we have our ping function there as the only public API and it does some
funny stuff.

I won't go into much riak_core details that are described elsewhere since this
is a talk that covers the practical aspects, there are many useful talks about
riak_core internals and theory around, you can watch them:

* http://vimeo.com/21772889
* http://vimeo.com/18758206

there are also some detailed articles about it:

* https://github.com/rzezeski/try-try-try
* https://github.com/basho/riak_core/wiki
* http://basho.com/where-to-start-with-riak-core/

but let's look at what it does line by line:

.. code:: erlang

        DocIdx = riak_core_util:chash_key({<<"ping">>, term_to_binary(now())}),

the line above hashes a key to decide to which vnode the call should go, a
riak_core app has a fixed number of vnodes that are distributed across all the
instances of your app, vnodes move from instance to instance when the number of
instances change to balance the load and have fault tolerance and scalability.

The call above will allow us to ask for vnodes that can handle that hashed key,
let's run it in the app console to see what it does:

.. code:: erlang

    (flavio@127.0.0.1)1> DocIdx = riak_core_util:chash_key({<<"ping">>, term_to_binary(now())}).
    <<207,185,91,89,64,167,168,83,113,154,212,211,27,36,113, 251,56,179,28,123>>

we seem to get a binary back, in the next line we ask for a list of vnodes that
can handle that hashed key:

.. code:: erlang

        PrefList = riak_core_apl:get_primary_apl(DocIdx, 1, flavio),

let's run it to see what it does:

.. code:: erlang

    (flavio@127.0.0.1)2> PrefList = riak_core_apl:get_primary_apl(DocIdx, 1, flavio).

    [{{1187470080331358621040493926581979953470445191168, 'flavio@127.0.0.1'}, primary}]

we get a list with one tuple that has 3 items, a long number, something that looks like a hist
and an atom, let's try changing the number 1:

.. code:: erlang

    (flavio@127.0.0.1)4> PrefList2 = riak_core_apl:get_primary_apl(DocIdx, 2, flavio).

    [{{1187470080331358621040493926581979953470445191168, 'flavio@127.0.0.1'}, primary},
     {{1210306043414653979137426502093171875652569137152, 'flavio@127.0.0.1'}, primary}]

now we get two tuples, the first one is the same, so what this does is to return
the number of vnodes that can handle the request from the hashed key by priority.

btw, the first number is the vnode id, it's what we get on the ping response :)

next line just unpacks the pref list to get the vnode id and ignore the other part:

.. code:: erlang

        [{IndexNode, _Type}] = PrefList,

and finally we ask riak_core to call the ping command on the IndexNode we got back:

.. code:: erlang

        riak_core_vnode_master:sync_spawn_command(IndexNode, ping, flavio_vnode_master).

let's try it on the console:

.. code:: erlang

    (flavio@127.0.0.1)5> [{IndexNode, _Type}] = PrefList.
    [{{1187470080331358621040493926581979953470445191168, 'flavio@127.0.0.1'}, primary}]

    (flavio@127.0.0.1)6> riak_core_vnode_master:sync_spawn_command(IndexNode, ping, flavio_vnode_master).
    {pong,1187470080331358621040493926581979953470445191168}

you can see we get IndexNode back in the pong response, now let's try passing the second IndexNode:

.. code:: erlang

    (flavio@127.0.0.1)7> [{IndexNode1, _Type1}, {IndexNode2, _Type2}] = PrefList2.
    [{{1187470080331358621040493926581979953470445191168, 'flavio@127.0.0.1'}, primary},
     {{1210306043414653979137426502093171875652569137152, 'flavio@127.0.0.1'}, primary}]

    (flavio@127.0.0.1)8> riak_core_vnode_master:sync_spawn_command(IndexNode2, ping, flavio_vnode_master).
    {pong,1210306043414653979137426502093171875652569137152}

we get the IndexNode2 back, that means that the request was sent to the second
vnode instead of the first one.

but where does the command go? the road is explained in this scientific chart::

    flavio.erl -> riak_core magic -> flavio_vnode.erl

let's see the content of flavio_vnode.erl (just the useful parts):

.. code:: erlang

    -module(flavio_vnode).
    -behaviour(riak_core_vnode).

    -export([start_vnode/1,
             init/1,
             terminate/2,
             handle_command/3,
             is_empty/1,
             delete/1,
             handle_handoff_command/3,
             handoff_starting/2,
             handoff_cancelled/1,
             handoff_finished/2,
             handle_handoff_data/2,
             encode_handoff_item/2,
             handle_coverage/4,
             handle_exit/3]).

    -record(state, {partition}).

    %% API
    start_vnode(I) ->
        riak_core_vnode_master:get_vnode_pid(I, ?MODULE).

    init([Partition]) ->
        {ok, #state { partition=Partition }}.

    %% Sample command: respond to a ping
    handle_command(ping, _Sender, State) ->
        {reply, {pong, State#state.partition}, State};
    handle_command(Message, _Sender, State) ->
        ?PRINT({unhandled_command, Message}),
        {noreply, State}.

ok, let's go by parts, first we declare our module:

.. code:: erlang

    -module(flavio_vnode).

then we specify that we want to implement the riak_core_vnode behaviour:

.. code:: erlang

    -behaviour(riak_core_vnode).

behaviours in erlang are like interfaces, a set of functions that a module must
implement to satisfy the behaviour specification, you can read more here:

http://www.erlang.org/doc/design_principles/des_princ.html

in this case riak_core defines a behaviour with a set of functions we must
implement to be a valid riak_core vnode, you can get an idea of the kind of
functionality we need by looking at the exported functions:

.. code:: erlang

    -export([start_vnode/1,
             init/1,
             terminate/2,
             handle_command/3,
             is_empty/1,
             delete/1,
             handle_handoff_command/3,
             handoff_starting/2,
             handoff_cancelled/1,
             handoff_finished/2,
             handle_handoff_data/2,
             encode_handoff_item/2,
             handle_coverage/4,
             handle_exit/3]).

for the moment most of them have a "dummy" implementation where they just to
the minimal amount of work to satisfy the behaviour and not more, it's our job
to change the default implementation to fit our needs.

we will have a record called state to keep info between callbacks, this is
typical erlang way of managing state so I won't cover it here:

.. code:: erlang

    -record(state, {partition}).

then we implement the api to start the vnode, nothing fancy:

.. code:: erlang

    %% API
    start_vnode(I) ->
        riak_core_vnode_master:get_vnode_pid(I, ?MODULE).

note that on init we store the Partition value on state so we can use it later,
this is what I referred above as vnode id, it's the big number you saw before:

.. code:: erlang

    init([Partition]) ->
        {ok, #state { partition=Partition }}.

and now for the interesting part, here we have our ping command implementation,
we match for ping in the Message position (the first argument):

.. code:: erlang

    handle_command(ping, _Sender, State) ->

and return a reply response with the second item in the tuple being the actual
response that the caller will get where we reply with the atom pong and the
partition number of this vnode, the last item in the tuple is the new state we
want to have for this vnode, since we didn't change anything we pass the
current value:

.. code:: erlang

        {reply, {pong, State#state.partition}, State};

and then we implement a catch all that will just print the unknown command and
give no reply back:

.. code:: erlang

    handle_command(Message, _Sender, State) ->
        ?PRINT({unhandled_command, Message}),
        {noreply, State}.

so this is the roundtrip of the ping call, our task to add more commands will be:

* add a function on flavio.erl that hides the internal work done to distribute the work
* add a new match on handle_command to match the command we added on flavio.erl and provide a reply

but before adding a new command let's play with the distribution part of
riak_core.

in our case we have all the vnodes on the same instance and on the same machine
so that's not that distributed, let's try running more than one node.

Creating a local cluster
------------------------

to create a local cluster we will need to create and start N different builds
and instances with slightly different configurations given the fact that all
instances are running on the same machine and share the same resources.

you can read more about devrel here:

https://github.com/basho/rebar_riak_core#devrel

first stop your running instance if you still have it running, then run:

.. code:: shell

    make devrel

you can see at the end of the output something similar to this:

.. code:: shell

    mkdir -p dev
    rel/gen_dev dev1 rel/vars/dev_vars.config.src rel/vars/dev1_vars.config
    Generating dev1 - node='flavio1@127.0.0.1' http=10018 handoff=10019
    (cd rel && /home/mariano/src/rct/flaviodb/rebar generate target_dir=../dev/dev1 overlay_vars=vars/dev1_vars.config)
    ==> rel (generate)

    mkdir -p dev
    rel/gen_dev dev2 rel/vars/dev_vars.config.src rel/vars/dev2_vars.config
    Generating dev2 - node='flavio2@127.0.0.1' http=10028 handoff=10029
    (cd rel && /home/mariano/src/rct/flaviodb/rebar generate target_dir=../dev/dev2 overlay_vars=vars/dev2_vars.config)
    ==> rel (generate)

    mkdir -p dev
    rel/gen_dev dev3 rel/vars/dev_vars.config.src rel/vars/dev3_vars.config
    Generating dev3 - node='flavio3@127.0.0.1' http=10038 handoff=10039
    (cd rel && /home/mariano/src/rct/flaviodb/rebar generate target_dir=../dev/dev3 overlay_vars=vars/dev3_vars.config)
    ==> rel (generate)

    mkdir -p dev
    rel/gen_dev dev4 rel/vars/dev_vars.config.src rel/vars/dev4_vars.config
    Generating dev4 - node='flavio4@127.0.0.1' http=10048 handoff=10049
    (cd rel && /home/mariano/src/rct/flaviodb/rebar generate target_dir=../dev/dev4 overlay_vars=vars/dev4_vars.config)
    ==> rel (generate)

you can see it generated 4 builds (dev1 ... dev4) and that it assigned different names
(flavio1 ... flavio4) and assigned different ports for http and handoff.

now let's start them:

.. code:: shell

    for d in dev/dev*; do $d/bin/flavio start; done

now instead of starting and connecting to a console as before we just started
the nodes, but how do we know they are running?

welp, we can ping them from the command line tool that the template kindly provides to us:

.. code:: shell

    for d in dev/dev*; do $d/bin/flavio ping; done

we should see 4 individual pong replies::

    pong
    pong
    pong
    pong

but we don't have a cluster yet, because each instance is running unaware of the others, to make them
an actual cluster we have to make them aware of each other.

you can see that they aren't aware by asking any of them about the status of its members:

.. code:: shell

    $ dev/dev1/bin/flavio-admin member-status

    ================================= Membership ==================================
    Status     Ring    Pending    Node
    -------------------------------------------------------------------------------
    valid     100.0%      --      'flavio1@127.0.0.1'
    -------------------------------------------------------------------------------
    Valid:1 / Leaving:0 / Exiting:0 / Joining:0 / Down:0

you see flavio1 "cluster" has only one node in it (itself), try with another:

.. code:: shell

    $ dev/dev4/bin/flavio-admin member-status

    ================================= Membership ==================================
    Status     Ring    Pending    Node
    -------------------------------------------------------------------------------
    valid     100.0%      --      'flavio4@127.0.0.1'
    -------------------------------------------------------------------------------
    Valid:1 / Leaving:0 / Exiting:0 / Joining:0 / Down:0

note dev4 instead of dev1 in the command.

now we will ask nodes 2, 3 and 4 to join node 1 on a cluster:

.. code:: shell

    $ for d in dev/dev{2,3,4}; do $d/bin/flavio-admin cluster join flavio1@127.0.0.1; done

    Success: staged join request for 'flavio2@127.0.0.1' to 'flavio1@127.0.0.1'
    Success: staged join request for 'flavio3@127.0.0.1' to 'flavio1@127.0.0.1'
    Success: staged join request for 'flavio4@127.0.0.1' to 'flavio1@127.0.0.1'

check again the cluster status:

.. code:: shell

    $ dev/dev1/bin/flavio-admin member-status

    ================================= Membership ==================================
    Status     Ring    Pending    Node
    -------------------------------------------------------------------------------
    joining     0.0%      --      'flavio2@127.0.0.1'
    joining     0.0%      --      'flavio3@127.0.0.1'
    joining     0.0%      --      'flavio4@127.0.0.1'
    valid     100.0%      --      'flavio1@127.0.0.1'
    -------------------------------------------------------------------------------
    Valid:1 / Leaving:0 / Exiting:0 / Joining:3 / Down:0dev/dev1/bin/flavio-admin member-status

they are joining, because we have to approve cluster changes, let's look what's
the plan:

.. code:: shell

    $ dev/dev1/bin/flavio-admin cluster plan

    =============================== Staged Changes ================================
    Action         Details(s)
    -------------------------------------------------------------------------------
    join           'flavio2@127.0.0.1'
    join           'flavio3@127.0.0.1'
    join           'flavio4@127.0.0.1'
    -------------------------------------------------------------------------------


    NOTE: Applying these changes will result in 1 cluster transition

    ###############################################################################
                             After cluster transition 1/1
    ###############################################################################

    ================================= Membership ==================================
    Status     Ring    Pending    Node
    -------------------------------------------------------------------------------
    valid     100.0%     25.0%    'flavio1@127.0.0.1'
    valid       0.0%     25.0%    'flavio2@127.0.0.1'
    valid       0.0%     25.0%    'flavio3@127.0.0.1'
    valid       0.0%     25.0%    'flavio4@127.0.0.1'
    -------------------------------------------------------------------------------
    Valid:4 / Leaving:0 / Exiting:0 / Joining:0 / Down:0

    Transfers resulting from cluster changes: 48
      16 transfers from 'flavio1@127.0.0.1' to 'flavio4@127.0.0.1'
      16 transfers from 'flavio1@127.0.0.1' to 'flavio3@127.0.0.1'
      16 transfers from 'flavio1@127.0.0.1' to 'flavio2@127.0.0.1'

looks good to me, let's commit that plan so it actually happens:

.. code:: shell

    $ dev/dev1/bin/flavio-admin cluster commit

    Cluster changes committed

let's see the cluster status again:

.. code:: shell

    $ dev/dev1/bin/flavio-admin member-status

    ================================= Membership ==================================
    Status     Ring    Pending    Node
    -------------------------------------------------------------------------------
    valid      25.0%      --      'flavio1@127.0.0.1'
    valid      25.0%      --      'flavio2@127.0.0.1'
    valid      25.0%      --      'flavio3@127.0.0.1'
    valid      25.0%      --      'flavio4@127.0.0.1'
    -------------------------------------------------------------------------------
    Valid:4 / Leaving:0 / Exiting:0 / Joining:0 / Down:0


now the cluster has 4 nodes which have the ring distributed equally :)

to just be sure it's not all a lie, let's connect to some nodes and run the
ping again, first from node 1:

.. code:: shell

    $ dev/dev1/bin/flavio attach
    Attaching to /tmp//home/mariano/src/rct/flaviodb/dev/dev1/erlang.pipe.1 (^D to exit)

.. code:: erlang

    (flavio1@127.0.0.1)1> flavio:ping().
    {pong,822094670998632891489572718402909198556462055424}
    (flavio1@127.0.0.1)2> [Quit]

now from node 3:

.. code:: shell

    $ dev/dev3/bin/flavio attach
    Attaching to /tmp//home/mariano/src/rct/flaviodb/dev/dev3/erlang.pipe.1 (^D to exit)

.. code:: erlang

    (flavio3@127.0.0.1)1> flavio:ping()
    (flavio3@127.0.0.1)1> .
    {pong,1438665674247607560106752257205091097473808596992}
    (flavio3@127.0.0.1)2> [Quit]

note that we got the reply from a different vnode the second time.

Adding a command
----------------

first let's add a simple command to get the workflow right.

we will build a calculation command first and then we will add some state
tracking to it.

our command will start simply by adding two numbers and returning the result
and the vnode that calculated the result.

let's start by defining our new command from the user's perspective, we want to
be able to run:

.. code:: erlang

    flavio:add(2, 5).

and get our result back, so let's add the add function to the flavio module,
first we add it to the list of the exported functions:

.. code:: erlang

    -export([ping/0, add/2]).

and then we add our implementation starting from the ping version:

.. code:: erlang

    add(A, B) ->
        DocIdx = riak_core_util:chash_key({<<"add">>, term_to_binary({A, B})}),
        PrefList = riak_core_apl:get_primary_apl(DocIdx, 1, flavio),
        [{IndexNode, _Type}] = PrefList,
        riak_core_vnode_master:sync_spawn_command(IndexNode, {add, A, B}, flavio_vnode_master).

the changes are, the name (of course), the parameters it accepts, in our case it accepts two numbers,
but more subtle changes are in the following line:

.. code:: erlang

        DocIdx = riak_core_util:chash_key({<<"add">>, term_to_binary({A, B})}),

we change the name of the command (the first item in the tuple) and we also
changed the content of the arguments to term_to_binary, we could leave now()
there so the call will generate a new number on each call producing a different hash and therefore routing to a different vnode each time, but in our case we want a little more predictability.

we will pass the numbers we want to add as the second item in the tuple, this
means that if we want to add the same two numbers we will be routed to the same
vnodes every time, this is part of the "consistent hashing" you may have heard
about riak_core, we will try it in action later, but for now let's move to the next lines.

this two stay the same:

.. code:: erlang

        PrefList = riak_core_apl:get_primary_apl(DocIdx, 1, flavio),
        [{IndexNode, _Type}] = PrefList,

but the last one changed slightly:

.. code:: erlang

        riak_core_vnode_master:sync_spawn_command(IndexNode, {add, A, B}, flavio_vnode_master).

instead of passing ping as second parameter we pass our "command", that is,
which operation we want to perform and the parameters, this may seem familiar
if you ever implemented something like gen_server, if not, we basically send a message
with the information of the command we want to call and the other side matches
the message with the commands it understands and acts accordingly.

in our case now we must match this message/command on the vnode implementation,
this should be really easy, on flavio_vnode.erl we add the following clause to
the existing handle_command function:

.. code:: erlang

    handle_command({add, A, B}, _Sender, State) ->
        {reply, {A + B, State#state.partition}, State};

you can see we match the command on the first argument and as reply on the
second position of the tuple we send the response back, which contains the
addition as first item and the partition on as seconds, this just to keep track
of the routing, it's not needed to return it.

now stop your current instance if you have one running and build a new release::

    rm -rf rel/flavio
    make rel


now let's play a little with it::

    $ ./rel/flavio/bin/flavio console

.. code:: erlang

    (flavio@127.0.0.1)1> flavio:add(2, 5).
    {7,959110449498405040071168171470060731649205731328}

    (flavio@127.0.0.1)2> flavio:add(2, 5).
    {7,959110449498405040071168171470060731649205731328}

    (flavio@127.0.0.1)3> flavio:add(2, 5).
    {7,959110449498405040071168171470060731649205731328}

    (flavio@127.0.0.1)4> flavio:add(3, 5).
    {8,91343852333181432387730302044767688728495783936}

    (flavio@127.0.0.1)5> flavio:add(3, 5).
    {8,91343852333181432387730302044767688728495783936}

    (flavio@127.0.0.1)6> flavio:add(2, 5).
    {7,959110449498405040071168171470060731649205731328}

    (flavio@127.0.0.1)7> flavio:add(2, 9).
    {11,1255977969581244695331291653115555720016817029120}

    (flavio@127.0.0.1)8> flavio:add(2, 9).
    {11,1255977969581244695331291653115555720016817029120}

    (flavio@127.0.0.1)9> flavio:add(2, 5).
    {7,959110449498405040071168171470060731649205731328}

you can see that the same addition gets sent to the same vnode each time, if
the parameters change then it's sent to another one, but consistently.

this is how we handle scaling and distribution, by deciding which information
of our command is part of the hash key, this varies with each problem so it's
a design decision you have to make.

the full change is here: https://github.com/marianoguerra/flaviodb/commit/8e0fb2460791651fcc1aa5cd957b535437d07095

Keeping some state
------------------

this operations are stateless so it doesn't make much sense to route them
consistently, but now we will add some state tracking to count how many
additions each vnode made.

for this we will increment a operations counter on each vnode when an operation
is made and we will provide a way to retrieve this information as another
command.

first let's start by adding a new field to our state record to keep the count:

.. code:: erlang

    -record(state, {partition, ops_count=0}).

and then when we receive an addition command we increment the count and return
the new state in the 3 item tuple so that this new state becomes the vnode
state:

.. code:: erlang

    handle_command({add, A, B}, _Sender, State=#state{ops_count=CurrentCount}) ->
        NewCount = CurrentCount + 1,
        NewState = State#state{ops_count=NewCount},
        {reply, {A + B, State#state.partition}, NewState};

line by line, first we match the current ops_count:

.. code:: erlang

    handle_command({add, A, B}, _Sender, State=#state{ops_count=CurrentCount}) ->

then calculate the new count:

.. code:: erlang

        NewCount = CurrentCount + 1,

then create the new state record that is the same as the old one but with the
new count:

.. code:: erlang

        NewState = State#state{ops_count=NewCount},

and then we reply as before but we pass NewState as third item:

.. code:: erlang

        {reply, {A + B, State#state.partition}, NewState};

rebuild and run::

    $ rm -rf rel/flavio && make rel && ./rel/flavio/bin/flavio console

.. code:: erlang

    (flavio@127.0.0.1)1> flavio:add(2, 5).
    {7,959110449498405040071168171470060731649205731328}
    (flavio@127.0.0.1)2> flavio:add(2, 6).
    {8,1278813932664540053428224228626747642198940975104}

the full change is here: https://github.com/marianoguerra/flaviodb/commit/3b8a789308767f735ce45590f4d1887e2dbdb1b4

nothing different because we need a way to get that count, for that we will
implement a new command, get_stats, but how do we tell to which vnode?
can we ask all vnodes for this info?

well yes we can, it's called a coverage call, and it's a call that involves all
the vnodes

first we add the stats function to the export list:

.. code:: erlang

    -export([ping/0, add/2, stats/0]).

now we add the implementation:

.. code:: erlang

    stats() ->
        Timeout = 5000,
        flavio_coverage_fsm:start(stats, Timeout).

well, that was easy... but what is this flavio_coverage_fsm:start thing?

the high level description of a coverage call is that we do a coverage call for
all the vnodes and collect the results until we have all of them or until
timeout happens, this coverage call is implemented in the vnode by adding a clause
on the handle_coverage function to match the command sent to it, in our case,
we pass the atom "stats".

but someone has to take care of making the calls to all the vnodes, accumulating
the results and timing out if necessary.

for that riak_core provides a behaviour called riak_core_coverage_fsm, which
provides some callbacks we must implement and everything else will be handled
by riak_core, the callbacks we must implement are needed to init the state of
the process, to process each individual result and to do an action when the
collection is finished.

for the most basic case we will just initialize with some configured values,
init the state, on each individual result we will accumulate it and maybe
summarize it in some way and on finalization we return the result, we may also
do some summarization or cleanup if needed.

the code of flavio_coverage_fsm and flavio_coverage_fsm_sup (it's supervisor)
are really straight forward if you ever implemented something like a gen_fsm,
if not you can live by copying and pasting it and tweaking some details but at
some point you should go over and read about gen_fsm and OTP in general to get
a better sense of what's happening there.

but before we go to the vnode implementation other than creating this two new
modules to help us with our coverage call we need to register this new supervisor
in the our supervisor tree, this is also an OTP thing that you should investigate
on your own, there's a lot of useful information about it on the Erlang docs, books
and in Learn You Some Erlang.

to add this supervisor to the supervisor tree we must edit the file
flavio_sup.erl and add the following:

.. code:: erlang

    init(_Args) ->
        VMaster = { flavio_vnode_master,
                      {riak_core_vnode_master, start_link, [flavio_vnode]},
                      permanent, 5000, worker, [riak_core_vnode_master]},

        CoverageFSMs = {flavio_coverage_fsm_sup,
                        {flavio_coverage_fsm_sup, start_link, []},
                        permanent, infinity, supervisor, [flavio_coverage_fsm_sup]},
        {ok,
            { {one_for_one, 5, 10},
              [VMaster, CoverageFSMs]}}.

we added the CoverageFSMs definition and we added it to the list on the last
line.

the part that's interesting to us is the api call and the callback that must be
implemented in the vnode, which goes as follows:

.. code:: erlang

    handle_coverage(stats, _KeySpaces, {_, RefId, _}, State=#state{ops_count=OpsCount}) ->
        {reply, {RefId, [{ops_count, OpsCount}]}, State};
    handle_coverage(Req, _KeySpaces, _Sender, State) ->
        lager:warning("unknown coverage received ~p", [Req]),
        {norepl, State}.

we redefine the whole handle_coverage function to avoid it from stopping the
vnode in case it gets a coverage call it doesn't know about and change it so
that it only logs a warning and ignores it.

but the interesting function clause is the first one where we match the RefId
that is passed to us from flavio_coverage_fsm, which uses it to differentiate
all the calls and we also get from our state the info we are going to reply.

we reply with a two item tuple where the first item is the RefId we got and the
second is the coverage call response.

in this case I return a `proplist <http://www.erlang.org/doc/man/proplists.html>`_ just
to future proof this call and allow to return more information in the future.

now we rebuild and run the release to play with it::

    $ rm -rf rel/flavio && make rel && ./rel/flavio/bin/flavio console

.. code:: erlang

    (flavio@127.0.0.1)1> flavio:stats().
    {ok,[ lot of output here]}

    % let's use the api a little

    (flavio@127.0.0.1)2> flavio:add(2, 5).
    {7,959110449498405040071168171470060731649205731328}
    (flavio@127.0.0.1)3> flavio:add(2, 6).
    {8,1278813932664540053428224228626747642198940975104}
    (flavio@127.0.0.1)4> flavio:add(2, 6).
    {8,1278813932664540053428224228626747642198940975104}
    (flavio@127.0.0.1)5> flavio:add(2, 6).
    {8,1278813932664540053428224228626747642198940975104}
    (flavio@127.0.0.1)6> flavio:add(3, 6).
    {9,182687704666362864775460604089535377456991567872}
    (flavio@127.0.0.1)7> flavio:add(3, 6).
    {9,182687704666362864775460604089535377456991567872}
    (flavio@127.0.0.1)8> flavio:stats().
    {ok,[ lot of output here, maybe you can see some with ops_count > 0]}

    % let's filter the output to see interesting info

    (flavio@127.0.0.1)9> {ok, Stats} = flavio:stats().
    {ok,[ again lot of output here]}

    (flavio@127.0.0.1)10> lists:filter(fun ({_, _, [{ops_count, OpsCount}]}) -> OpsCount > 0 end, Stats).
    [{1278813932664540053428224228626747642198940975104, 'flavio@127.0.0.1', [{ops_count,3}]},
     {959110449498405040071168171470060731649205731328, 'flavio@127.0.0.1', [{ops_count,1}]},
     {182687704666362864775460604089535377456991567872, 'flavio@127.0.0.1', [{ops_count,2}]}]

we can see in the last call that there are 3 nodes that have ops_count set to
a value bigger than 0 and that matches the calls we did above.

the full change is here: https://github.com/marianoguerra/flaviodb/commit/9b6ef0ea2b9f0257733024b1468016a5d96b713c

Tolerating faults in our additions (?)
--------------------------------------

you know computers cannot be trusted, so we may want to run our commands in
more than one vnode and wait for a subset (or all of them) to finish before
considering the operation to be successful, for this when a command is ran we
will send the command to a number of vnodes, let's call it W and wait for a
number of them to succeed, let's call it N.

to do this we will need to do something similar than what we did with coverage
calls, we will need to setup a process that will send the command to a number
of vnodes and accumulate the responses or timeout if it takes to long, then
send the result back to the caller. We will also need a supervisor for it and
to register this supervisor in our main supervisor tree.

again I won't go into details on the fsm and supervisor implementations, maybe
I will add an annex later or comment the code heavily in case you want to
understand how it works, but just for you to know, I tend to copy those fsms from
other projects and adapt them to my needs, just don't tell anybody ;)

here is a diagram of how it works::

    +------+    +---------+    +---------+    +---------+              +------+
    |      |    |         |    |         |    |         |remaining = 0 |      |
    | Init +--->| Prepare +--->| Execute +--->| Waiting +------------->| Stop |
    |      |    |         |    |         |    |         |              |      |
    +------+    +---------+    +---------+    +-------+-+              +------+
                                                  ^   | |                    
                                                  |   | |        +---------+ 
                                                  +---+ +------->|         | 
                                                                 | Timeout | 
                                          remaining > 0  timeout |         | 
                                                                 +---------+ 


the code for the "caller/accumulator/waiter/replier" is in
flavio_io_fsm_sup.erl I did it as generic as I could so you can reuse it
easily, you have to pass an operation to it by calling flavio_op_fsm:op(N, W,
Op), where N and W are described above and where Op is a two item tuple, for
example for addition it would be {add, {A, B}}, it has to be that way so the
hashing is generic.

this fsm will then generate a RefId and will call our vnode with a command like
this: {RefId, Op} where Op is the two item tuple we passed to flavio_op_fsm:op.

flavio_op_fsm_sup is as generic as any fsm supervisor can get.

finally we register this new supervisor in our main supervisor tree in flavio_sup.erl:

.. code:: erlang

    init(_Args) ->
        VMaster = { flavio_vnode_master,
                      {riak_core_vnode_master, start_link, [flavio_vnode]},
                      permanent, 5000, worker, [riak_core_vnode_master]},

        CoverageFSMs = {flavio_coverage_fsm_sup,
                        {flavio_coverage_fsm_sup, start_link, []},
                        permanent, infinity, supervisor, [flavio_coverage_fsm_sup]},

        OpFSMs = {flavio_op_fsm_sup,
                     {flavio_op_fsm_sup, start_link, []},
                     permanent, infinity, supervisor, [flavio_op_fsm_sup]},
        {ok,
            { {one_for_one, 5, 10},
              [VMaster, CoverageFSMs, OpFSMs]}}.

as before we add the OpFSMs definition and we add it to the list in the last
line.

we need to modify our vnode handle_command to handle the new command
format, that includes the RefId and has the parameters inside a tuple:

.. code:: erlang

    handle_command({RefId, {add, {A, B}}}, _Sender, State=#state{ops_count=CurrentCount}) ->
        NewCount = CurrentCount + 1,
        NewState = State#state{ops_count=NewCount},
        {reply, {RefId, {A + B, State#state.partition}}, NewState};

and now instead of calculating the vnode ourselves we let out new flavio_op_fsm
take care of the call by changing the flavio:add implementation:

.. code:: erlang

    add(A, B) ->
        N = 3,
        W = 3,
        Timeout = 5000,

        {ok, ReqID} = flavio_op_fsm:op(N, W, {add, {A, B}}),
        wait_for_reqid(ReqID, Timeout).

in this case we require 3 vnodes to run the command and we wait for the 3 to
consider the request successful, if the operation takes more than 5000 ms then
we fail with a timeout error.

the following line sends the desired N, W and the command in the new format, we
get back a request id we must wait for, we will receive a message to this
process with that ReqID and the result when all the requests finished or with
the error in case it failed or timed out:

.. code:: erlang

        {ok, ReqID} = flavio_op_fsm:op(N, W, {add, {A, B}}),

to wait for the result we implement a function to do it for use:

.. code:: erlang

    wait_for_reqid(ReqID, Timeout).

which is implemented as follows:

.. code:: erlang

    wait_for_reqid(ReqID, Timeout) ->
        receive {ReqID, Val} -> {ok, Val}
        after Timeout -> {error, timeout}
        end.

let's rebuild and use it::

    $ rm -rf rel/flavio && make rel && ./rel/flavio/bin/flavio console

.. code:: erlang

    (flavio@127.0.0.1)1> flavio:add(2, 4).
    {ok,[{6,433883298582611803841718934712646521460354973696},
         {6,388211372416021087647853783690262677096107081728},
         {6,411047335499316445744786359201454599278231027712}]}

    (flavio@127.0.0.1)2> flavio:add(12, 4).
    {ok,[{16,68507889249886074290797726533575766546371837952},
         {16,45671926166590716193865151022383844364247891968},
         {16,22835963083295358096932575511191922182123945984}]}

as you can see we get the 3 results back, it's our job to decide what to do
with them, we can pick one and return that one or we can compare all the
results to be sure that all vnodes got the same result, this is part of
conflict resolution and it should be part of the design decisions of your app.

the full change is here: https://github.com/marianoguerra/flaviodb/commit/dde9698c821055512b59fc54c25dbc5b223e8afe

what about handoff?
-------------------

it seems you know a lot about riak_core do you?

well, the thing about `handoff <https://github.com/basho/riak_core/wiki/Handoffs>`_
is that it's used to move data between vnodes during ring resizing and until
this moment we don't have data to move around.

but this is about to change, let's implement a data store, but what will we
store? short messages.

one problem I have with social networks is that I have several interests and I
post in more than one language, and I hate having some people have to go
through my rants about things that they aren't interested in just because they
want to know about other aspects of my life.

this is about to change, let's disrupt some industries while we learn
riak_core.

how will it work? simple, each user has a set of streams he can post short
messages to, a stream is created when the user posts for the first time there.

let's think about the problem in a riak_core way, you have seen that the key
hashing until now is done with a two item tuple, here we have users that have
streams, that fits perfectly with our problem, what a coincidence!

so when a new message is posted we will hash {Username, Stream} and send the
message to W vnodes and wait confirmation from N of them that they stored the
message.

Writing
.......

We are going to add a new function to flavio's API like this:

.. code:: erlang

    flavio:post_msg(Username, Stream, Msg)

only if there was a library to write short messages to disk, let see...

another coinsidence! here's one: https://github.com/marianoguerra/fixstt

so we start adding an entry to rebar.config to add this new dependency:

.. code:: erlang

    {fixstt, ".*", {git, "git://github.com/marianoguerra/fixstt", {branch, "master"}}}

and we will ask rebar to fetch the new deps::

    ./rebar get-deps

then we need to actually implement post_msg, it will be really similar to
add since we want to write to W vnodes and wait for N confirmations:

.. code:: erlang

    post_msg(Username, Stream, Msg) ->
        N = 3,
        W = 3,
        Timeout = 5000,

        {ok, ReqID} = flavio_op_fsm:op(N, W, {post_msg, {Username, Stream, Msg}},
                                       {Username, Stream}),
        wait_for_reqid(ReqID, Timeout).

you may have noticed that we passed an extra parameter to flavio_op_fsm:op,
that's because I added an extra parameter to be used as explicit key for the
hashing function in case the operation has more than 2 items.

to start let's implement a really naive way of writting the messages:

.. code:: erlang

    handle_command({RefId, {post_msg, {Username, Stream, Msg}}}, _Sender,
                   State=#state{partition=Partition}) ->
        PartitionStr = integer_to_list(Partition),
        StreamPath = filename:join([PartitionStr, Username, Stream, "msgs"]),
        ok = filelib:ensure_dir(StreamPath),
        {ok, StreamIo} = fixsttio:open(StreamPath),
        Entry = fixstt:new(Msg),
        {ok, _NewStream, EntryId} = fixsttio:append(StreamIo, Entry),
        EntryWithId = fixstt:set(Entry, id, EntryId),
        {reply, {RefId, {EntryWithId, State#state.partition}}, State};

let's disect the interesting lines:

.. code:: erlang

        StreamPath = filename:join([PartitionStr, Username, Stream, "msgs"]),

here we create the path were we are going to store the messages, it's built
by joining the partition id, username, stream and the string msgs.

why the partition id? because one server instance will have more than one vnode
on it and it may get a request to write a message for the same Username and
Stream, in that case if we didn't use the Partition to differentiate them then
more than one vnode will try to open and/or write to the same file causing
interesting results, also, later when we move one vnode to another server we
want to just move the data from that vnode.

.. code:: erlang

        ok = filelib:ensure_dir(StreamPath),

here we make sure the directory for the msgs file exists.

.. code:: erlang

        {ok, StreamIo} = fixsttio:open(StreamPath),

then we open our stream with the path we built before.

.. code:: erlang

        Entry = fixstt:new(Msg),
        {ok, _NewStream, EntryId} = fixsttio:append(StreamIo, Entry),
        EntryWithId = fixstt:set(Entry, id, EntryId),

then we create a new entry, append it to the stream and set the returned id to it.

.. code:: erlang

        {reply, {RefId, {EntryWithId, State#state.partition}}, State};

finally we return the received RefId as first item and as second a pair with
the entry we wrote and the partition that handled the request.

now let's try everything together:

.. code:: erlang

    (flavio@127.0.0.1)1> flavio:post_msg(<<"mariano">>, <<"english">>, <<"hello world!">>).

    {ok,[{{fixstt,1,9001,9001,12,1416928004032,0,0, <<"hello world!">>},
          981946412581700398168100746981252653831329677312},
         {{fixstt,1,9001,9001,12,1416928004032,0,0, <<"hello world!">>},
          959110449498405040071168171470060731649205731328},
         {{fixstt,1,9001,9001,12,1416928004032,0,0, <<"hello world!">>},
          1004782375664995756265033322492444576013453623296}]}

    (flavio@127.0.0.1)2> flavio:post_msg(<<"mariano">>, <<"spanish">>, <<"hola mundo!">>).
    {ok,[{{fixstt,1,9001,9001,11,1416928004035,0,0, <<"hola mundo!">>},
          890602560248518965780370444936484965102833893376},
         {{fixstt,1,9001,9001,11,1416928004035,0,0,<<"hola mundo!">>},
          867766597165223607683437869425293042920709947392},
         {{fixstt,1,9001,9001,11,1416928004035,0,0,<<"hola mundo!">>},
          844930634081928249586505293914101120738586001408}]}

it worked we can see the 3 responses have the record stored on it, we can make
sure it worked by going to the server folder and searching for a file names msgs::

    cd rel/flavio
    $ find -name msgs

in my case this is the ouput, in your case it may vary::

    ./890602560248518965780370444936484965102833893376/mariano/spanish/msgs
    ./844930634081928249586505293914101120738586001408/mariano/spanish/msgs
    ./1004782375664995756265033322492444576013453623296/mariano/english/msgs
    ./959110449498405040071168171470060731649205731328/mariano/english/msgs
    ./867766597165223607683437869425293042920709947392/mariano/spanish/msgs
    ./981946412581700398168100746981252653831329677312/mariano/english/msgs

we can see that there are 3 instances of spanish and 3 of english.

the full change is here: https://github.com/marianoguerra/flaviodb/commit/62ad84faa81d94c4057522d9da3b3c82df911dbb

Cleanup
.......

just to do some cleanup we will create the partition folders inside a base
directory so we don't fill the base rel/flaviodb directory with partition
folders, later we can make this base directory configurable, the change is
here: https://github.com/marianoguerra/flaviodb/commit/b33841758f254d8eb7a5e08c245e0274d74eb994

Reading
.......

now we need to be able to read the messages for a given Username and Stream,
for that we will implement a new function in the API that does something like:

.. code:: erlang

    flavio:get_msgs(Username, Stream, Id, Count).

here we tell we want to read Count messages from the stream Stream from user
Username starting from id Id.

the code will be really similar to the one of the write, we can choose to read
from only one vnode but to keep it simple and consistent we will read from N,
this could be used to implement something like consistency checks on read.

since there's a lot of shared code with post_msg I will refactor the commong code
to a utility function and add some error checking, the resulting code for both
post_msg and get_msgs on flavio_vnode is:

.. code:: erlang

    handle_command({RefId, {post_msg, {Username, Stream, Msg}}}, _Sender, State) ->
        {ok, StreamIo} = get_stream(State, Username, Stream),
        Entry = fixstt:new(Msg),
        Result = case fixsttio:append(StreamIo, Entry) of
                     {ok, StreamIo1, EntryId} ->
                         {ok, _StreamIo2} = fixstt:close(StreamIo1),
                         {ok, fixstt:set(Entry, id, EntryId)};
                     Other -> Other
                 end,
        {reply, {RefId, {Result, State#state.partition}}, State};

    handle_command({RefId, {get_msgs, {Username, Stream, Id, Count}}}, _Sender, State) ->
        {ok, StreamIo} = get_stream(State, Username, Stream),
        Result = case fixsttio:read(StreamIo, Id, Count) of
                     {ok, StreamIo1, Entries} ->
                         {ok, _StreamIo2} = fixstt:close(StreamIo1),
                         {ok, Entries};
                     Other -> Other
                 end,
        {reply, {RefId, {Result, State#state.partition}}, State};

now let's play with it, let's write 3 entries to 2 streams and try reading them back:

.. code:: erlang

    (flavio@127.0.0.1)1> flavio:post_msg(<<"mariano">>, <<"english">>, <<"hello world!">>).
    {ok,[{{ok,{fixstt,1,9001,9001,12,1416930241029,0,0, <<"hello world!">>}},
          981946412581700398168100746981252653831329677312},
         {{ok,{fixstt,1,9001,9001,12,1416930241029,0,0, <<"hello world!">>}},
          1004782375664995756265033322492444576013453623296},
         {{ok,{fixstt,1,9001,9001,12,1416930241029,0,0, <<"hello world!">>}},
          959110449498405040071168171470060731649205731328}]}

    (flavio@127.0.0.1)2> flavio:post_msg(<<"mariano">>, <<"english">>, <<"second post">>).
    {ok,[{{ok,{fixstt,2,9001,9001,11,1416930252869,0,0, <<"second post">>}},
          981946412581700398168100746981252653831329677312},
         {{ok,{fixstt,2,9001,9001,11,1416930252868,0,0, <<"second post">>}},
          1004782375664995756265033322492444576013453623296},
         {{ok,{fixstt,2,9001,9001,11,1416930252868,0,0, <<"second post">>}},
          959110449498405040071168171470060731649205731328}]}

    (flavio@127.0.0.1)3> flavio:post_msg(<<"mariano">>, <<"english">>, <<"eating something">>).
    {ok,[{{ok,{fixstt,3,9001,9001,16,1416930260533,0,0, <<"eating something">>}},
          1004782375664995756265033322492444576013453623296},
         {{ok,{fixstt,3,9001,9001,16,1416930260533,0,0, <<"eating something">>}},
          981946412581700398168100746981252653831329677312},
         {{ok,{fixstt,3,9001,9001,16,1416930260533,0,0, <<"eating something">>}},
          959110449498405040071168171470060731649205731328}]}

    (flavio@127.0.0.1)4> flavio:post_msg(<<"mariano">>, <<"spanish">>, <<"hola mundo!">>).
    {ok,[{{ok,{fixstt,1,9001,9001,11,1416930275765,0,0, <<"hola mundo!">>}},
          890602560248518965780370444936484965102833893376},
         {{ok,{fixstt,1,9001,9001,11,1416930275765,0,0, <<"hola mundo!">>}},
          867766597165223607683437869425293042920709947392},
         {{ok,{fixstt,1,9001,9001,11,1416930275765,0,0, <<"hola mundo!">>}},
          844930634081928249586505293914101120738586001408}]}

    (flavio@127.0.0.1)5> flavio:post_msg(<<"mariano">>, <<"spanish">>, <<"segundo post">>).
    {ok,[{{ok,{fixstt,2,9001,9001,12,1416930280219,0,0, <<"segundo post">>}},
          867766597165223607683437869425293042920709947392},
         {{ok,{fixstt,2,9001,9001,12,1416930280218,0,0, <<"segundo post">>}},
          844930634081928249586505293914101120738586001408},
         {{ok,{fixstt,2,9001,9001,12,1416930280218,0,0, <<"segundo post">>}},
          890602560248518965780370444936484965102833893376}]}

    (flavio@127.0.0.1)6> flavio:post_msg(<<"mariano">>, <<"spanish">>, <<"comiendo algo">>).
    {ok,[{{ok,{fixstt,3,9001,9001,13,1416930284791,0,0, <<"comiendo algo">>}},
          844930634081928249586505293914101120738586001408},
         {{ok,{fixstt,3,9001,9001,13,1416930284791,0,0, <<"comiendo algo">>}},
          890602560248518965780370444936484965102833893376},
         {{ok,{fixstt,3,9001,9001,13,1416930284791,0,0, <<"comiendo algo">>}},
          867766597165223607683437869425293042920709947392}]}

nothing new under the sun there, now let's try reading some of those, just as a
little help, the returned value is a record called fixstt, the second item on
it is the entry id, you can see it starts from 1 and goes to 3, we will use it
to query them:

.. code:: erlang

    (flavio@127.0.0.1)8> % query from mariano/spanish from id 1, get 1 post
    (flavio@127.0.0.1)8> flavio:get_msgs(<<"mariano">>, <<"spanish">>, 1, 1).
    {ok,[{{ok,[{fixstt,1,9001.0,9001.0,11,1416930275765,0,0, <<"hola mundo!">>}]},
          867766597165223607683437869425293042920709947392},
         {{ok,[{fixstt,1,9001.0,9001.0,11,1416930275765,0,0, <<"hola mundo!">>}]},
          890602560248518965780370444936484965102833893376},
         {{ok,[{fixstt,1,9001.0,9001.0,11,1416930275765,0,0, <<"hola mundo!">>}]},
          844930634081928249586505293914101120738586001408}]}

    (flavio@127.0.0.1)9> % same but get 3 posts
    (flavio@127.0.0.1)9> flavio:get_msgs(<<"mariano">>, <<"spanish">>, 1, 3).
    {ok,[{{ok,[{fixstt,1,9001.0,9001.0,11,1416930275765,0,0, <<"hola mundo!">>},
               {fixstt,2,9001.0,9001.0,12,1416930280218,0,0, <<"segundo post">>},
               {fixstt,3,9001.0,9001.0,13,1416930284791,0,0, <<"comiendo algo">>}]},
          890602560248518965780370444936484965102833893376},
         {{ok,[{fixstt,1,9001.0,9001.0,11,1416930275765,0,0, <<"hola mundo!">>},
               {fixstt,2,9001.0,9001.0,12,1416930280218,0,0, <<"segundo post">>},
               {fixstt,3,9001.0,9001.0,13,1416930284791,0,0, <<"comiendo algo">>}]},
          844930634081928249586505293914101120738586001408},
         {{ok,[{fixstt,1,9001.0,9001.0,11,1416930275765,0,0, <<"hola mundo!">>},
               {fixstt,2,9001.0,9001.0,12,1416930280219,0,0, <<"segundo post">>},
               {fixstt,3,9001.0,9001.0,13,1416930284791,0,0, <<"comiendo algo">>}]},
          867766597165223607683437869425293042920709947392}]}

    (flavio@127.0.0.1)10> % query but starting from some weird id
    (flavio@127.0.0.1)10> flavio:get_msgs(<<"mariano">>, <<"spanish">>, 10, 3).
    {ok,[{{error,outofbound}, 844930634081928249586505293914101120738586001408},
         {{error,outofbound}, 890602560248518965780370444936484965102833893376},
         {{error,outofbound}, 867766597165223607683437869425293042920709947392}]}

we can see we get what we ask for and we handle errors correctly when we ask
for some nonsense.

that's all for now, there's lot to improve on this, for example we could avoid
opening and closing the file for each request, but that doesn't add anything
useful to this guide, it may be implemented later as an optimization.

full changes here: https://github.com/marianoguerra/flaviodb/commit/b0b74fbac07b542479ef8453434715c317251d4f

Listing streams from a user
...........................

now that we have data on disc we can make use of the coverage calls for
something more interesting, listing a user's streams, the call will be quite
simple:

.. code:: erlang

    flavio:list_streams(Username).

should return a list of all streams for that Username.

again we are reusing a lot of code we already wrote so the api implementation is
simply:

.. code:: erlang

    list_streams(Username) ->
        Timeout = 5000,
        flavio_coverage_fsm:start({list_streams, Username}, Timeout).

and the implementation:

.. code:: erlang

    handle_coverage({list_streams, Username}, _KeySpaces, {_, RefId, _}, State) ->
        Streams = lists:sort(list_streams(State, Username)),
        {reply, {RefId, {ok, Streams}}, State};

the implementation of list_streams is pretty straightforward you can see it in
the commit.

just to make the output less noisy, let's remove the responses that are empty lists so we only get the useful information:

.. code:: erlang

    list_streams(Username) ->
        Timeout = 5000,
        case flavio_coverage_fsm:start({list_streams, Username}, Timeout) of
            {ok, Responses} ->
                {ok, lists:filter(fun ({_Partition, _Node, {ok, []}}) -> false;
                                 ({_Partition, _Node, _Streams}) -> true
                             end, Responses)};
            Other -> Other
        end.

full change here: https://github.com/marianoguerra/flaviodb/commit/5a2ca66103313541af605021428345fdf28d7336

Listing users
.............

this one will be really easy based on the last change, let's go straight to the code:

.. code:: erlang

    list_users() ->
        Timeout = 5000,
        case flavio_coverage_fsm:start(list_users, Timeout) of
            {ok, Responses} ->
                {ok, lists:filter(fun filter_empty_responses/1, Responses)};
            Other -> Other
        end.

I refactored the filtering to reuse it on both calls.

the implementation on the vnode again is simple:

.. code:: erlang

    handle_coverage(list_users, _KeySpaces, {_, RefId, _}, State) ->
        Users = lists:sort(list_users(State)),
        {reply, {RefId, {ok, Users}}, State};

playing with both functions on the console:

.. code:: erlang

    (flavio@127.0.0.1)1> flavio:post_msg(<<"mariano">>, <<"spanish">>, <<"comiendo algo">>).
    ...
    (flavio@127.0.0.1)2> flavio:post_msg(<<"mariano">>, <<"english">>, <<"eating something">>).
    ...
    (flavio@127.0.0.1)3> flavio:post_msg(<<"bob">>, <<"english">>, <<"eating something too">>).
    ...

    (flavio@127.0.0.1)4> flavio:list_users().
    {ok,[{981946412581700398168100746981252653831329677312,
          'flavio@127.0.0.1', {ok,[<<"mariano">>]}},
         {890602560248518965780370444936484965102833893376,
          'flavio@127.0.0.1', {ok,[<<"mariano">>]}},
         {959110449498405040071168171470060731649205731328,
          'flavio@127.0.0.1', {ok,[<<"mariano">>]}},
         {867766597165223607683437869425293042920709947392,
          'flavio@127.0.0.1', {ok,[<<"mariano">>]}},
         {844930634081928249586505293914101120738586001408,
          'flavio@127.0.0.1', {ok,[<<"mariano">>]}},
         {365375409332725729550921208179070754913983135744,
          'flavio@127.0.0.1', {ok,[<<"bob">>]}},
         {342539446249430371453988632667878832731859189760,
          'flavio@127.0.0.1', {ok,[<<"bob">>]}},
         {319703483166135013357056057156686910549735243776,
          'flavio@127.0.0.1', {ok,[<<"bob">>]}},
         {1004782375664995756265033322492444576013453623296,
          'flavio@127.0.0.1', {ok,[<<"mariano">>]}}]}

    (flavio@127.0.0.1)6> flavio:list_streams(<<"mariano">>).
    {ok,[{981946412581700398168100746981252653831329677312,
          'flavio@127.0.0.1', {ok,[<<"english">>]}},
         {867766597165223607683437869425293042920709947392,
          'flavio@127.0.0.1', {ok,[<<"spanish">>]}},
         {959110449498405040071168171470060731649205731328,
          'flavio@127.0.0.1', {ok,[<<"english">>]}},
         {844930634081928249586505293914101120738586001408,
          'flavio@127.0.0.1', {ok,[<<"spanish">>]}},
         {890602560248518965780370444936484965102833893376,
          'flavio@127.0.0.1', {ok,[<<"spanish">>]}},
         {1004782375664995756265033322492444576013453623296,
          'flavio@127.0.0.1', {ok,[<<"english">>]}}]}

    (flavio@127.0.0.1)7> flavio:list_streams(<<"bob">>).
    {ok,[{319703483166135013357056057156686910549735243776,
          'flavio@127.0.0.1', {ok,[<<"english">>]}},
         {342539446249430371453988632667878832731859189760,
          'flavio@127.0.0.1', {ok,[<<"english">>]}},
         {365375409332725729550921208179070754913983135744,
          'flavio@127.0.0.1', {ok,[<<"english">>]}}]}

full change here: https://github.com/marianoguerra/flaviodb/commit/21d4fa819aa0429cab7d19dffb6240f9aeb66391

Implementing Handoff
--------------------

Finally we have all the pieces to implement
`handoff <https://github.com/basho/riak_core/wiki/Handoffs>`_, this is a
complex topic that is described in detail in other places, but still it's hard
to understand, so I will do my best.

the reasons to start a handoff are:


* A ring update event for a ring that all other nodes have already seen.
* A secondary vnode is idle for a period of time and the primary, original owner of the partition is up again.

when this happen riak_core will inform the vnode that handoff is starting, calling
handoff_starting, if it returns false it's cancelled, if it returns true it calls
is_empty, that must return false to inform that the vnode has something to handoff (it's not empty)
or true to inform that the vnode is empty, if it returns true the handoff is considered finished, if
false then a call is done to handle_handoff_command passing as first parameter
an opaque structure that contains two fields we are insterested in, foldfun and
acc0, they can be unpacked with a macro like this:

.. code:: erlang

    handle_handoff_command(?FOLD_REQ{foldfun=Fun, acc0=Acc0}, _Sender, State) ->

this function must iterate through all the keys it stores and for each of them
call foldfun with the key as first argument, the value as second argument and
the latest acc0 value as third, like this:

.. code:: erlang

    AccIn1 = Fun(Key, Value, AccIn0),

the result of the function call is the new acc0 you must pass to the next call
to foldfun, the last acc0 must be returned by the handle_handoff_command
function like this:

.. code:: erlang

    {reply, AccFinal, State};

for each call to Fun(Key, Entry, AccIn0) riak_core will send it to the new vnode, to do that
it must encode the data before sending, it does this by calling encode_handoff_item(Key, Value),
where you must encode the data before sending it, it's common to do something like:

.. code:: erlang

    term_to_binary({Key, Value}).

when the value is received by the new vnode it must decode it and do something
with it, this is done by the  function handle_handoff_data, where we decode
the received data and do the appropriate thing with it:

.. code:: erlang

    handle_handoff_data(BinData, State) ->
        TermData = binary_to_term(BinData),
        {Key, Value} = TermData,
        % do something with it here

when we sent all the key/values handoff_finished will be called and then delete
so we cleanup the data on the old vnode:

.. code:: erlang

    handoff_finished(_TargetNode, State=#state{partition=Partition}) ->
        lager:info("handoff finished ~p", [Partition]),
        {ok, State}.

    delete(State) ->
        Path = partition_path(State),
        remove_path(Path),
        {ok, State}.

you can decide to handle other commands sent to the vnode while the handoff is
running, you can choose to do one of the followings:

* handle it in the current vnode
* forward it to the vnode we are handing off
* drop it

what to do depends on the design of you app, all of them have tradeoffs.

the signature of all the responses is:

.. code:: erlang

    -callback handle_handoff_command(Request::term(), Sender::sender(), ModState::term()) ->
    {reply, Reply::term(), NewModState::term()} |
    {noreply, NewModState::term()} |
    {async, Work::function(), From::sender(), NewModState::term()} |
    {forward, NewModState::term()} |
    {drop, NewModState::term()} |
    {stop, Reason::term(), NewModState::term()}.

an advanced diagram of the flow is as follows::
                                                                    
     +-----------+      +----------+        +----------+                
     |           | true |          | false  |          |                
     | Starting  +------> is_empty +--------> fold_req |                
     |           |      |          |        |          |                
     +-----+-----+      +----+-----+        +----+-----+                
           |                 |                   |                      
           | false           | true              | ok                   
           |                 |                   |                      
     +-----v-----+           |              +----v-----+     +--------+ 
     |           |           |              |          |     |        | 
     | Cancelled |           +--------------> finished +-----> delete | 
     |           |                          |          |     |        | 
     +-----------+                          +----------+     +--------+ 
                                                                    
the pseudocode for the handoff implementation would be something like:

.. code:: python

    handle_handoff_command(Fun, Acc, _Sender, State):
        for Stream in AllUserStreams:
            for Key, Entry in get_entries(Stream):
                # pardon the mutability, it's just to make the code smaller
                Acc = Fun(Key, Entry, Acc)

        return reply, Acc, State

the complete code for the handoff can be seen in the commit for this feature.

to test this we will need to build a cluster by parts, for this we will build a
devrel again, start a node, put some data in it and then join some other nodes,
and watch the handoff running.

I've added a lot of logging to the functions involved in the handoff so we can
follow them from the console.

Let's start by building the devrel and start one node:

.. code:: shell

    rm -rf dev && make devrel
    ./dev/dev1/bin/flavio console

we will generate some data for it, just paste it on the node's console:

.. code:: erlang

    Nums = lists:seq(1, 10).
    Users = [<<"bob">>, <<"sandy">>, <<"patrick">>, <<"gary">>].
    Topics = [<<"english">>, <<"spanish">>, <<"erlang">>, <<"riak_core">>].
    MakeUsersAndTopics = fun (User) -> lists:map(fun (Topic) -> {User, Topic} end, Topics) end.
    UsersAndTopics = lists:flatmap(MakeUsersAndTopics, Users).
    MakeMsg = fun (User, Topic, I) -> list_to_binary(io_lib:format("~s says ~p in ~s", [User, I, Topic])) end.
    MakeMsgs = fun ({User, Topic}) -> lists:map(fun (I) -> {User, Topic, MakeMsg(User, Topic, I)} end, Nums) end.
    Msgs = lists:flatmap(MakeMsgs, UsersAndTopics).

    lists:foreach(fun ({Username, Topic, Msg}) -> flavio:post_msg(Username, Topic, Msg) end, Msgs).

we can check that it worked by listing users and buckets:

.. code:: erlang

    flavio:list_users().
    flavio:list_streams(<<"bob">>).

now that we have data let's start a second node in another terminal:

.. code:: shell

    ./dev/dev2/bin/flavio console

in yet another terminal we ask the second node to join the first one:

.. code:: shell

    ./dev/dev2/bin/flavio-admin cluster join flavio1@127.0.0.1

and we commit the plan

.. code:: shell

    dev/dev1/bin/flavio-admin cluster plan
    dev/dev1/bin/flavio-admin cluster commit
    
you can see the progress by looking at the consoles or by printing the cluster status, which at some point will reach 50% for each ring::

    $ dev/dev1/bin/flavio-admin member-status

    ================================= Membership ==================================
    Status     Ring    Pending    Node
    -------------------------------------------------------------------------------
    valid      50.0%      --      'flavio1@127.0.0.1'
    valid      50.0%      --      'flavio2@127.0.0.1'
    -------------------------------------------------------------------------------
    Valid:2 / Leaving:0 / Exiting:0 / Joining:0 / Down:0

we should see logs appearing on both consoles informing about the handoff
progress, here are some excerpts from mine, yours will differ obviously:

.. code:: shell

    (flavio1@127.0.0.1)12> 10:53:57.316 [info] 'flavio2@127.0.0.1' joined cluster with status 'joining'

    10:54:26.600 [info] handoff starting 45671926166590716193865151022383844364247891968
    10:54:26.602 [info] handoff is empty? false 22835963083295358096932575511191922182123945984
    10:54:26.603 [info] handoff cancelled 114179815416476790484662877555959610910619729920
    10:54:26.619 [info] Starting ownership_transfer transfer of flavio_vnode from 'flavio1@127.0.0.1' 22835963083295358096932575511191922182123945984 to 'flavio2@127.0.0.1' 22835963083295358096932575511191922182123945984
    10:54:26.620 [info] fold req 45671926166590716193865151022383844364247891968
    10:54:26.620 [info] handling handoff for patrick/spanish
    10:54:26.667 [info] ownership_transfer transfer of flavio_vnode from 'flavio1@127.0.0.1' 45671926166590716193865151022383844364247891968 to 'flavio2@127.0.0.1' 45671926166590716193865151022383844364247891968 completed: sent 1.08 KB bytes in 10 of 10 objects in 0.05 seconds (23.45 KB/second)
    10:54:26.668 [info] handoff finished 22835963083295358096932575511191922182123945984
    10:54:26.681 [info] handoff delete flavio_data/45671926166590716193865151022383844364247891968
    10:54:26.683 [info] terminate 45671926166590716193865151022383844364247891968: normal

multiplicate that for approx 32 and you will get an idea of the amount of logs generated :)

on the receiving side we get logs like this:

.. code:: shell

    (flavio2@127.0.0.1)1> 10:54:21.864 [info] 'flavio2@127.0.0.1' changed from 'joining' to 'valid'
    10:54:26.620 [info] Receiving handoff data for partition flavio_vnode:45671926166590716193865151022383844364247891968 from {"127.0.0.1",34478}
    10:54:26.669 [info] Handoff receiver for partition 22835963083295358096932575511191922182123945984 exited after processing 10 objects from {"127.0.0.1",32835}
    10:54:36.614 [info] Receiving handoff data for partition flavio_vnode:137015778499772148581595453067151533092743675904 from {"127.0.0.1",53206}
    10:55:23.619 [info] handoff starting 68507889249886074290797726533575766546371837952
    10:55:23.639 [info] handoff is empty? true 1370157784997721485815954530671515330927436759040
    10:55:23.639 [info] handoff delete flavio_data/1370157784997721485815954530671515330927436759040
    10:55:23.640 [info] terminate 890602560248518965780370444936484965102833893376: normal


in this case this vnode also starts a handoff to the node 1 but since it has no
data it will finish all the vnode handoffs right away.

to be sure that the handoff happened you can list the folders in each node, this
is what I got:

.. code:: shell

    $ tree dev/dev1/flavio_data
    dev/dev1/flavio_data
    ├── 0
    │   └── patrick
    │       └── spanish
    │           └── msgs
    ├── 1073290264914881830555831049026020342559825461248
    │   └── gary
    │       └── english
    │           └── msgs
    ├── 1164634117248063262943561351070788031288321245184
    │   ├── bob
    │   │   └── riak_core
    │   │       └── msgs
    │   └── gary
    │       └── spanish
    │           └── msgs

    ...

    ├── 707914855582156101004909840846949587645842325504
    │   └── sandy
    │       └── erlang
    │           └── msgs
    └── 91343852333181432387730302044767688728495783936
        └── sandy
            └── english
                └── msgs

    63 directories, 22 files

    $ tree dev/dev2/flavio_data
    dev/dev2/flavio_data
    ├── 1118962191081472546749696200048404186924073353216
    │   ├── bob
    │   │   └── riak_core
    │   │       └── msgs
    │   └── gary
    │       └── english
    │           └── msgs

    ...

    ├── 662242929415565384811044689824565743281594433536
    │   ├── patrick
    │   │   └── english
    │   │       └── msgs
    │   └── sandy
    │       └── erlang
    │           └── msgs
    └── 685078892498860742907977265335757665463718379520
        ├── patrick
        │   └── english
        │       └── msgs
        └── sandy
            └── erlang
                └── msgs

    67 directories, 26 files

as you can see the handoff actually happened :)

you can keep playing by adding more messages, and adding more nodes to the
cluster and see the handoff happen.

all the changes for handoff are here: https://github.com/marianoguerra/flaviodb/commit/4c259862b9b8407e83a88c4566c337d88e59c430

Providing an API
----------------

Setup
.....

first we need to add our new dependencies to rebar.config, we need a web server
and a way to parse json, we will use cowboy and jsx for that:

.. code:: erlang

    {cowboy, "1.0.0", {git, "https://github.com/ninenines/cowboy", {tag, "1.0.0"}}},
    {bullet, "0.4.1", {git, "https://github.com/extend/bullet", {tag, "0.4.1"}}},
    {jsxn, ".*", {git, "https://github.com/talentdeficit/jsxn", {tag, "v2.1.1"}}}

then we need to add a new app to the list our application depends on, in this case
cowboy, we add this to flavio.app.src in the applications section.

if you are using erlang R17 you will get an error compiling bullet, the quick way to fix it for the moment is
to run this command from the root of the project:

.. code:: shell

    printf '0a\n%%%% coding: latin-1\n.\nw\n' | ed deps/bullet/src/bullet_handler.erl

it will add a header to that file specifying the encoding to let it compile.

Creating Messages
.................

then we need to start the web server and register some handlers, we will start
by implementing a way to post a new message, this will expose the flavio:post_msg
function through a HTTP API for this we need to add some code to flavio_app.erl
when the server starts to register and start the server, the start function will
end up looking like this:

.. code:: erlang

    start(_StartType, _StartArgs) ->
        Dispatch = cowboy_router:compile([
            {'_', [{"/msgs/:user/:topic", handler_flavio_msgs, []}]}
        ]),
        ApiPort = 8080,
        ApiAcceptors = 100,
        {ok, _} = cowboy:start_http(http, ApiAcceptors, [{port, ApiPort}], [
            {env, [{dispatch, Dispatch}]}
        ]),

        case flavio_sup:start_link() of
            {ok, Pid} ->
                ok = riak_core:register([{vnode_module, flavio_vnode}]),

                ok = riak_core_ring_events:add_guarded_handler(flavio_ring_event_handler, []),
                ok = riak_core_node_watcher_events:add_guarded_handler(flavio_node_event_handler, []),
                ok = riak_core_node_watcher:service_up(flavio, self()),

                {ok, Pid};
            {error, Reason} ->
                {error, Reason}
        end.

.. code:: erlang

        Dispatch = cowboy_router:compile([
            {'_', [{"/msgs/:user/:topic", handler_flavio_msgs, []}]}
        ]),

here we compile our routes, we will call handler_flavio_msgs handler when a
request is done to /msgs/:user/:topic, here :user and :topic are placeholders
and will allow use to retrieve its content when we handle the request.

.. code:: erlang

        {ok, _} = cowboy:start_http(http, ApiAcceptors, [{port, ApiPort}], [
            {env, [{dispatch, Dispatch}]}
        ]),

then we start the server with our routes and some parameters.

now we have to implement handler_flavio_msgs, it's documented in detail in the
`cowboy documentation <http://ninenines.eu/docs/en/cowboy/1.0/>`_, I will just cover
the interesting parts here.

.. code:: erlang

    -record(state, {username, topic}).

    init({tcp, http}, _Req, _Opts) -> {upgrade, protocol, cowboy_rest}.

    rest_init(Req, []) ->
        {Username, Req1} = cowboy_req:binding(username, Req),
        {Topic, Req2} = cowboy_req:binding(topic, Req1),

        {ok, Req2, #state{username=Username, topic=Topic}}.

we declare a state record that will contain the information about a request,
then on init we tell cowboy this is a rest handler.

on rest_init we extract the :username and :topic placeholders and set it to
state so we can access it later.

.. code:: erlang

    allowed_methods(Req, State) -> {[<<"POST">>], Req, State}.

we only handle POST in this handler

.. code:: erlang

    content_types_accepted(Req, State) ->
        {[{{<<"application">>, <<"json">>, '*'}, from_json}], Req, State}.

we only handle requests where content type is application/json, when that's the
content type we want the from_json function to be called.

and now, the interesting part, the from_json function:

.. code:: erlang

    from_json(Req, State=#state{username=Username, topic=Topic}) ->
        {ok, Body, Req1} = cowboy_req:body(Req),
        case jsx:is_json(Body) of
            true ->
                Data = jsx:decode(Body),
                Msg = proplists:get_value(<<"msg">>, Data, nil),

                if is_binary(Msg) ->
                       {ok, [FirstResponse|_]} = flavio:post_msg(Username, Topic, Msg),
                       {{ok, Entity}, _Partition} = FirstResponse,
                       EntityPList = fixstt:to_proplist(Entity),
                       EntityJson = jsx:encode(EntityPList),
                       response(Req, State, EntityJson);
                   true ->
                       bad_request(Req1, State, <<"{\"type\": \"no-msg\"}">>)
                end;
            false ->
                bad_request(Req1, State, <<"{\"type\": \"invalid-body\"}">>)
        end.

we do some extra error checking to provide better error reporting, I think
some of this code should go in other cowboy callbacks, but to keep it simple
let's leave it as is.

we extract the body and check if it's json, if it is we decode it and get the
msg field from the object, if the msg field is a string (binary here) then we
proceed to post the message using the username and topic we extracted before
from the url.

remember that the response contains N responses from N vnodes, we just get
the first one (no consistency checking), extract the entity, convert it to
JSON and return it as the body of the response.

now build a new release and let's play with it:

.. code:: shell

    $ curl -X POST http://localhost:8080/msgs/mariano/english -H "Content-Type: application/json" -d '{"msg": "hello world"}'
    {"id":1,"lat":9001,"lng":9001,"date":1417081176410,"ref":0,"type":0,"msg":"hello world"}

    $ curl -X POST http://localhost:8080/msgs/mariano/english -H "Content-Type: application/json" -d '{"msg": "hello world again"}'
    {"id":2,"lat":9001,"lng":9001,"date":1417081185756,"ref":0,"type":0,"msg":"hello world again"}

    $ curl -X POST http://localhost:8080/msgs/mariano/spanish -H "Content-Type: application/json" -d '{"msg": "hola mundo"}'
    {"id":1,"lat":9001,"lng":9001,"date":1417081201062,"ref":0,"type":0,"msg":"hola mundo"}

    $ curl -X POST http://localhost:8080/msgs/mariano/spanish -H "Content-Type: application/json" -d '{"msg": "hola mundo nuevamente"}'
    {"id":2,"lat":9001,"lng":9001,"date":1417081204533,"ref":0,"type":0,"msg":"hola mundo nuevamente"}


the happy path works fine, let's try the unhappy ones:

.. code:: shell

    curl -X POST http://localhost:8080/msgs/mariano/spanish -H "Content-Type: application/xml" -d '{"msg": "hola mundo nuevamente"}' -v

    * Connected to localhost (127.0.0.1) port 8080 (#0)
    > POST /msgs/mariano/spanish HTTP/1.1
    > User-Agent: curl/7.37.1
    > Host: localhost:8080
    > Accept: */*
    > Content-Type: application/xml
    > Content-Length: 32
    >
    < HTTP/1.1 415 Unsupported Media Type

    curl -X PUT http://localhost:8080/msgs/mariano/spanish -H "Content-Type: application/json" -d '{"msg": "hola mundo nuevamente"}' -v
    > PUT /msgs/mariano/spanish HTTP/1.1
    > User-Agent: curl/7.37.1
    > Host: localhost:8080
    > Accept: */*
    > Content-Type: application/json
    > Content-Length: 32
    >
    < HTTP/1.1 405 Method Not Allowed

    curl -X POST http://localhost:8080/msgs/mariano/spanish -H "Content-Type: application/json" -d 'this is not json'
    {"type": "invalid-body"}

    curl -X POST http://localhost:8080/msgs/mariano/spanish -H "Content-Type: application/json" -d '{}' -v
    > POST /msgs/mariano/spanish HTTP/1.1
    > User-Agent: curl/7.37.1
    > Host: localhost:8080
    > Accept: */*
    > Content-Type: application/json
    > Content-Length: 2
    >
    < HTTP/1.1 400 Bad Request

    {"type": "no-msg"}

cowboy takes care of handling the other cases for us :)

the full change is here: https://github.com/marianoguerra/flaviodb/commit/416528e6f8a1cd1cfd2f789dd87d1afc761485c6

Querying Messages
.................

This will be similar to what we did in the previous section, we will start by adding
two new fields to our state, from and limit, which will be query parameters
we will extract on init and will be used if the request is a GET request, this
two parameters will be used to specify from where and how many entries the user
wants to query

.. code:: erlang

    -record(state, {username, topic, from, limit}).

    rest_init(Req, []) ->
        {Username, Req1} = cowboy_req:binding(username, Req),
        {Topic, Req2} = cowboy_req:binding(topic, Req1),

        {FromStr, Req3} = cowboy_req:qs_val(<<"from">>, Req2, <<"">>),
        {LimitStr, Req4} = cowboy_req:qs_val(<<"limit">>, Req3, <<"1">>),

        From = to_int_or(FromStr, nil),
        Limit = to_int_or(LimitStr, 1),

        {ok, Req4, #state{username=Username, topic=Topic, from=From, limit=Limit}}.

from can be left out and it means "query limit items from the end".

.. code:: erlang

    allowed_methods(Req, State) -> {[<<"POST">>, <<"GET">>], Req, State}.

    content_types_provided(Req, State) ->
        {[{{<<"application">>, <<"json">>, '*'}, to_json}], Req, State}.

we say we also support GET now and that when a request is done that accepts
application/json we will handle it in the to_json function.

.. code:: erlang

    to_json(Req, State=#state{username=Username, topic=Topic, from=From, limit=Limit}) ->
        {ok, [FirstResponse|_]} = flavio:get_msgs(Username, Topic, From, Limit),
        {{ok, Entities}, _Partition} = FirstResponse,
        EntitiesPList = lists:map(fun fixstt:to_proplist/1, Entities),
        EntitiesJson = jsx:encode(EntitiesPList),

        {EntitiesJson, Req, State}.

finally the to_json function where we simply fall the flavio function, get the
first response, and encode it to json.

now let's play with it:

.. code:: shell

    $ curl -X POST http://localhost:8080/msgs/mariano/spanish -H "Content-Type: application/json" -d '{"msg": "hola mundo"}'
    {"id":1,"lat":9001,"lng":9001,"date":1417084202384,"ref":0,"type":0,"msg":"hola mundo"}

    $ curl -X POST http://localhost:8080/msgs/mariano/spanish -H "Content-Type: application/json" -d '{"msg": "hola mundo nuevamente"}'
    {"id":2,"lat":9001,"lng":9001,"date":1417084204320,"ref":0,"type":0,"msg":"hola mundo nuevamente"}

    $ curl http://localhost:8080/msgs/mariano/spanish\?from\=1\&limit\=1
    [{"id":1,"lat":9001.0,"lng":9001.0,"date":1417084202384,"ref":0,"type":0,"msg":"hola mundo"}]

    $ curl http://localhost:8080/msgs/mariano/spanish\?from\=1\&limit\=2
    [{"id":1,"lat":9001.0,"lng":9001.0,"date":1417084202384,"ref":0,"type":0,"msg":"hola mundo"},
     {"id":2,"lat":9001.0,"lng":9001.0,"date":1417084204320,"ref":0,"type":0,"msg":"hola mundo nuevamente"}]

    $ curl http://localhost:8080/msgs/mariano/spanish\?from\=1\&limit\=20
    [{"id":1,"lat":9001.0,"lng":9001.0,"date":1417084202384,"ref":0,"type":0,"msg":"hola mundo"},
     {"id":2,"lat":9001.0,"lng":9001.0,"date":1417084204320,"ref":0,"type":0,"msg":"hola mundo nuevamente"}]

    $ curl http://localhost:8080/msgs/mariano/spanish\?limit\=20
    [{"id":1,"lat":9001.0,"lng":9001.0,"date":1417084202384,"ref":0,"type":0,"msg":"hola mundo"},
     {"id":2,"lat":9001.0,"lng":9001.0,"date":1417084204320,"ref":0,"type":0,"msg":"hola mundo nuevamente"}]

    $ curl http://localhost:8080/msgs/mariano/euskera\?limit\=20
    []

full change here: https://github.com/marianoguerra/flaviodb/commit/4dfcf6ad49250d87bdb1356df3b490826c04fc24
