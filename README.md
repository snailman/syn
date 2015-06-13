# Syn
**Syn** (short for _synonym_) is a global process registry for Erlang.

## Introduction
Syn is a process registry that has the following features:

 * Global (i.e. a process is uniquely identified with a Key across all the nodes of a cluster).
 * Any term can be used as Key.
 * Fast writes.
 * Automatically handles net splits.


## Notes
In any distributed system you are faced with a consistency challenge, which is often resolved by having one master arbiter performing all write operations (chosen with a mechanism of [leader election](http://en.wikipedia.org/wiki/Leader_election)), or through [atomic transactions](http://en.wikipedia.org/wiki/Atomicity_(database_systems)).

Syn was born for applications of the [IoT](http://en.wikipedia.org/wiki/Internet_of_Things) field. In this context, Keys used to identify a process are often the physical object's unique identifier (for instance, its serial or mac address), and are therefore already defined and unique _before_ hitting the system.  The consistency challenge is less of a problem in this case, since the likelihood of concurrent incoming requests that would register processes with the same Key on different nodes is extremely low and, in most cases, acceptable.

In addition, write speeds were a determining factor in the architecture of Syn.

Therefore, Availability has been chosen over Consistency and Syn is [eventually consistent](http://en.wikipedia.org/wiki/Eventual_consistency).


## Install

If you're using [rebar](https://github.com/rebar/rebar) , add `syn` as a dependency in your project's `rebar.config` file:

```erlang
{syn, ".*", {git, "git://github.com/ostinelli/syn.git", "master"}}
```

Then, get and compile your dependencies:

```bash
$ rebar get-deps
$ rebar compile
```

## Usage

### Setup
Ensure to start Syn from your application. This can be done by either providing it as a dependency in your `.app` file, or by starting it manually:

```erlang
syn:start().
```

### Basic
To register a process:

```erlang
syn:register(Key, Pid) -> ok | {error, Error}.

Types:
	Key = any()
	Pid = pid()
	Error = taken
```

To retrieve a Pid from a Key:

```erlang
syn:find_by_key(Key) -> Pid | undefined.

Types:
	Key = any()
	Pid = pid()
```

To retrieve a Key from a Pid:

```erlang
syn:find_by_pid(Pid) -> Key | undefined.

Types:
	Pid = pid()
	Key = any()
```

To unregister a previously a Key:

```erlang
syn:unregister(Key) -> ok | {error, Error}.

Types:
	Key = any()
	Error = undefined
```

To retrieve the count of total registered processes running in the cluster:

```erlang
syn:count() -> non_neg_integer().
```

To retrieve the count of total registered processes running on a specific node:

```erlang
syn:count(Node) -> non_neg_integer().

Types:
	Node = atom()
```

Processes are automatically monitored and removed from the registry if they die.

### Options
Options can be set at runtime using the `syn:options/1` method.

```erlang
syn:options(SynOptions) -> ok.

Types:
	SynOptions = [SynOption]
	SynOption = ProcessExitCallback | NetsplitConflictingMode
	ProcessExitCallback = {process_exit_callback, function() | undefined}
	NetsplitConflictingMode = {netsplit_conflicting_mode, kill | {send_message, any()}}
```

#### Callbacks
You can set a callback to be triggered when a process exits. This callback will be called only on the node where the process was running.

Define a callback:
```erlang
CallbackFun = fun(Key, Pid, Reason) -> any().

Types:
	Key = any()
	Pid = pid()
	Reason = any()
```
The arguments Key and Pid are the ones of the process that exited with Reason.

For instance, if you want to print a log when a process exited:

```erlang
%% define the callback
CallbackFun = fun(Key, Pid, Reason) ->
	error_logger:info_msg("Process with Key ~p and Pid ~p exited with reason ~p~n", [Key, Pid, Reason])
end,

%% set the option
syn:options([
	{process_exit_callback, CallbackFun}
]).
```

#### Conflict resolution
After a net split, when nodes reconnect, Syn will merge the data from all the nodes in the cluster.

If the same Key was used to register a process on different nodes during a net split, then there will be a conflict. By default, Syn will discard the processes running on the node the conflict is being resolved on, and will kill it by sending a `kill` signal with `exit(Pid, kill)`.

If this is not desired, you can change the option `netsplit_conflicting_mode` to instruct Syn to send a message to the discarded process, so that you can trigger any actions on that process (such as a graceful shutdown).

For example, if you want the message `shutdown` to be send to the discarded process:

```erlang
syn:options([
	{netsplit_conflicting_mode, {send_message, shutdown}}
]).
```

If instead you want to ensure that an `exit(Pid, kill)` signal is sent to the discarded process:

```erlang
syn:options([
	{netsplit_conflicting_mode, kill}
]).
```

This is the default, so you do not have to specify this behavior if you haven't changed it.

## Internals
Under the hood, Syn performs dirty reads and writes into a distributed in-memory Mnesia table, synchronized across nodes.

To automatically handle net splits, Syn implements a specialized and simplified version of the mechanisms used in Ulf Wiger's [unsplit](https://github.com/uwiger/unsplit) framework.


## Contributing
So you want to contribute? That's great! Please follow the guidelines below. It will make it easier to get merged in.

Before implementing a new feature, please submit a ticket to discuss what you intend to do. Your feature might already be in the works, or an alternative implementation might have already been discussed.

Do not commit to master in your fork. Provide a clean branch without merge commits. Every pull request should have its own topic branch. In this way, every additional adjustments to the original pull request might be done easily, and squashed with `git rebase -i`. The updated branch will be visible in the same pull request, so there will be no need to open new pull requests when there are changes to be applied.

Ensure to include proper testing. To run Syn tests you simply have to be in the project's root directory and run:

```bash
$ make tests
```
