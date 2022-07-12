[![Ruby](https://github.com/ruby/debug/actions/workflows/ruby.yml/badge.svg?branch=master)](https://github.com/ruby/debug/actions/workflows/ruby.yml?query=branch%3Amaster) [![Protocol](https://github.com/ruby/debug/actions/workflows/protocol.yml/badge.svg)](https://github.com/ruby/debug/actions/workflows/protocol.yml)

# debug.rb

This library provides debugging functionality to Ruby (MRI) 2.6 and later. It has several advantages:

- Fast: No performance penalty on non-stepping mode and non-breakpoints.
- Native remote debugging support:
  - [UNIX domain socket](/docs/remote_debugging.md#invoke-program-as-a-remote-debuggee) (UDS)
  - [TCP/IP](/docs/remote_debugging.md#tcpip)
  - Integration with rich debugger frontends

     Frontend |  [Console](/docs/remote_debugging.md#debugger-console) | [VSCode](/docs/remote_debugging.md#vscode) | [Chrome DevTools](/docs/remote_debugging.md#chrome-devtool-integration) |
     ---|---|---|---|
     Connection | UDS, TCP/IP | UDS, TCP/IP | TCP/IP |
     Requirement | No | [vscode-rdbg](https://marketplace.visualstudio.com/items?itemName=KoichiSasada.vscode-rdbg) | Chrome |
  - See the [remote debugging guide](/docs/remote_debugging.md) for details
- Flexible: Users can use the debugger in multiple ways
  - Through requiring files - like `require "debug"`
  - Through the [`rdbg` executable](#the-rdbg-executable)
  - Through Ruby APIs
  - See [activate the debugger in your program](#activate-the-debugger-in-your-program) for details

# Installation

```
$ gem install debug
```
If you use Bundler, write the following line to your Gemfile.

```rb
gem "debug", ">= 1.0.0"
```

# Usage

The debugger is designed to support a wide range of use cases, so you have many ways to use it.

But a debugging session usually consists of 4 steps:

1. [Activate the debugger in your program](#activate-the-debugger-in-your-program)
1. [Set breakpoints](#set-breakpoints)
    - Through [`binding.break`](#the-bindingbreak-method)
    - Or [breakpoint commands](#breakpoint)
1. Execute/continue your program and wait for it to hit the breakpoints
1. [Start debugging](#start-debugging)
    - Here's the [full command list](#console-commands)
    - You can also type `help` or `help <command>` in the console to see commands

> **Note**
> You can connect the debugger to your program remotely through UNIX socket or TCP/IP.
> To learn more, please check the [remote debugging guide](docs/remote_debugging.md).

> **Note**
> If you want to use VSCode/Chrome integration, the steps will be slightly different. Please also check their dedicated sections:
> - [VSCode](/docs/remote_debugging.md#vscode)
> - [Chrome DevTools](/docs/remote_debugging.md#chrome-devtool-integration)

## Activate the debugger in your program

As mentioned earlier, you can use various ways to integrate the debugger with your program. Here's a simple breakdown:

Start at program start | `rdbg` | require | debugger API (after `require "debug/session"`)
---|---|---|---|
Yes | `rdbg` | `require "debug/start"` | `DEBUGGER__.start`
No | `rdbg --nonstop` | `require "debug"` | `DEBUGGER__.start(nonstop: true)`

But here are the 2 most common use cases:

### `require "debug"`

Similar to `byebug` or `pry`, once you've required `debug`, you can start setting breakpoints with the [`binding.break`](#the-bindingbreak-method) method.

### The `rdbg` executable

You can also start your program with the `rdbg` executable, which will enter a debugging session at the beginning of your program by default.

```shell
❯ rdbg target.rb
[1, 7] in target.rb
=>   1| def foo # stops at the beginning of the program
     2|   10
     3| end
     4|
     5| foo
     6|
     7| binding.break
=>#0    <main> at target.rb:1
(rdbg)
```

If you don't want to stop your program until it hits a breakpoint, you can use `rdbg --nonstop` instead (or `-n` for short).

```shell
❯ rdbg --nonstop target.rb
[2, 7] in target.rb
     2|   10
     3| end
     4|
     5| foo
     6|
=>   7| binding.break # stops at the first breakpoint
=>#0    <main> at target.rb:7
(rdbg)
```

If you want to run a command written in Ruby like like `rake`, `rails`, `bundle`, `rspec` and so on, you can use `rdbg -c` option.

- Without `-c` option, `rdbg <name>` expects `<name>` to be a Ruby script and invokes it like `ruby <name>` with the debugger.
- With `-c` option, `rdbg -c <name>` expects `<name>` be be command in `PATH` and simply invoke it with the debugger.

Examples:
- `rdbg target.rb`
- `rdbg -c -- rails server`
- `rdbg -c -- bundle exec ruby foo.rb`
- `rdbg -c -- bundle exec rake test`
- `rdbg -c -- ruby target.rb` is same as `rdbg target.rb`

> **Note**
> `--` is needed to separate the command line options for `rdbg` and invoking command. For example, `rdbg -c rake -T` is recognized like `rdbg -c -T -- rake`. It should be `rdbg -c -- rake -T`.

> **Note**
> If you want to use bundler (`bundle` command), you need to write `gem debug` line in your `Gemfile`.

## Set breakpoints

### The `binding.break` method

`binding.break` (and its aliases `binding.b` and `debugger`) set breakpoints at the written line.

```rb
❯ ruby -rdebug target.rb
[2, 7] in target.rb
     2|   10
     3| end
     4|
     5| foo
     6|
=>   7| binding.break
=>#0    <main> at target.rb:7
(rdbg)
```

#### Advanced usages

- If `do: 'command'` is specified, the debugger will

    1. Stop the program
    1. Run the `command` as a debug command
    1. Continue the program.

    It is useful if you only want to call a debug command and don't want to stop there.

    ```rb
    def initialize
      @a = 1
      binding.b do: 'watch @a'
    end
    ```

    In this case, the debugger will register a watch breakpoint for `@a` and continue to run.

- If `pre: 'command'` is specified, the debugger will
    1. Stop the program
    1. Run the `command` as a debug command
    1. Keep the console open

    It is useful if you have repeated operations to perform before the debugging at the breakpoint

    ```rb
    def foo
      binding.b pre: 'info locals'
      ...
    end
    ```

    In this case, the debugger will display local variable information automatically so you don't need to type it repeatedly.

## Start debugging

Once you're in the debugger console, you can start debugging with it. But here are some useful tips:

- `Enter` without any input repeats the last command (useful when repeating `step`s).
- `Ctrl-D` is the same as the `quit` command.

### Command & expression parsing

Because the debugger supports both Ruby expression evaluation and dozens of commands, name collision happens.

So here're a few rules explaining how the debugger interprets your input:

- If `<input>.split(" ")` does **NOT** start with any debugger command (e.g. `my_var` or `foo bar`), it will be evaluated as `pp <input>`
- If `<input>.split(" ")` starts with a command or a command shortcut (like `info` and `i`), it will be treated as a command instead.

Some examples:

  - `foo.bar` is same as `pp foo.bar`.
  - `info arg` are `i arg` are considered as `<info cmd> arg`
  - `info(arg)` is considered as `pp self.info(arg)`
  - `i` is considered as `<info cmd>`
  - `pp i` prints `i`


### Console commands

You can use the following debug commands. Each command should be written in 1 line.

The `[...]` notation means this part can be eliminate. For example, `s[tep]` means `s` or `step` are valid command. `ste` is not valid.
The `<...>` notation means the argument.

Here's a [Google sheet](https://docs.google.com/spreadsheets/d/1TlmmUDsvwK4sSIyoMv-io52BUUz__R5wpu-ComXlsw0/edit?usp=sharing) for comparing this and other Ruby debuggers' commands.

### Control flow

* `s[tep]`
  * Step in. Resume the program until next breakable point.
* `s[tep] <n>`
  * Step in, resume the program at `<n>`th breakable point.
* `n[ext]`
  * Step over. Resume the program until next line.
* `n[ext] <n>`
  * Step over, same as `step <n>`.
* `fin[ish]`
  * Finish this frame. Resume the program until the current frame is finished.
* `fin[ish] <n>`
  * Finish `<n>`th frames.
* `c[ontinue]`
  * Resume the program.
* `q[uit]` or `Ctrl-D`
  * Finish debugger (with the debuggee process on non-remote debugging).
* `q[uit]!`
  * Same as q[uit] but without the confirmation prompt.
* `kill`
  * Stop the debuggee process with `Kernel#exit!`.
* `kill!`
  * Same as kill but without the confirmation prompt.
* `sigint`
  * Execute SIGINT handler registered by the debuggee.
  * Note that this command should be used just after stop by `SIGINT`.

### Breakpoint

* `b[reak]`
  * Show all breakpoints.
* `b[reak] <line>`
  * Set breakpoint on `<line>` at the current frame's file.
* `b[reak] <file>:<line>` or `<file> <line>`
  * Set breakpoint on `<file>:<line>`.
* `b[reak] <class>#<name>`
   * Set breakpoint on the method `<class>#<name>`.
* `b[reak] <expr>.<name>`
   * Set breakpoint on the method `<expr>.<name>`.
* `b[reak] ... if: <expr>`
  * break if `<expr>` is true at specified location.
* `b[reak] ... pre: <command>`
  * break and run `<command>` before stopping.
* `b[reak] ... do: <command>`
  * break and run `<command>`, and continue.
* `b[reak] ... path: <path>`
  * break if the path matches to `<path>`. `<path>` can be a regexp with `/regexp/`.
* `b[reak] if: <expr>`
  * break if: `<expr>` is true at any lines.
  * Note that this feature is super slow.
* `catch <Error>`
  * Set breakpoint on raising `<Error>`.
* `catch ... if: <expr>`
  * stops only if `<expr>` is true as well.
* `catch ... pre: <command>`
  * runs `<command>` before stopping.
* `catch ... do: <command>`
  * stops and run `<command>`, and continue.
* `catch ... path: <path>`
  * stops if the exception is raised from a `<path>`. `<path>` can be a regexp with `/regexp/`.
* `watch @ivar`
  * Stop the execution when the result of current scope's `@ivar` is changed.
  * Note that this feature is super slow.
* `watch ... if: <expr>`
  * stops only if `<expr>` is true as well.
* `watch ... pre: <command>`
  * runs `<command>` before stopping.
* `watch ... do: <command>`
  * stops and run `<command>`, and continue.
* `watch ... path: <path>`
  * stops if the path matches `<path>`. `<path>` can be a regexp with `/regexp/`.
* `del[ete]`
  * delete all breakpoints.
* `del[ete] <bpnum>`
  * delete specified breakpoint.

### Information

* `bt` or `backtrace`
  * Show backtrace (frame) information.
* `bt <num>` or `backtrace <num>`
  * Only shows first `<num>` frames.
* `bt /regexp/` or `backtrace /regexp/`
  * Only shows frames with method name or location info that matches `/regexp/`.
* `bt <num> /regexp/` or `backtrace <num> /regexp/`
  * Only shows first `<num>` frames with method name or location info that matches `/regexp/`.
* `l[ist]`
  * Show current frame's source code.
  * Next `list` command shows the successor lines.
* `l[ist] -`
  * Show predecessor lines as opposed to the `list` command.
* `l[ist] <start>` or `l[ist] <start>-<end>`
  * Show current frame's source code from the line <start> to <end> if given.
* `edit`
  * Open the current file on the editor (use `EDITOR` environment variable).
  * Note that edited file will not be reloaded.
* `edit <file>`
  * Open <file> on the editor.
* `i[nfo]`
   * Show information about current frame (local/instance variables and defined constants).
* `i[nfo] l[ocal[s]]`
  * Show information about the current frame (local variables)
  * It includes `self` as `%self` and a return value as `%return`.
* `i[nfo] i[var[s]]` or `i[nfo] instance`
  * Show information about instance variables about `self`.
* `i[nfo] c[onst[s]]` or `i[nfo] constant[s]`
  * Show information about accessible constants except toplevel constants.
* `i[nfo] g[lobal[s]]`
  * Show information about global variables
* `i[nfo] ... /regexp/`
  * Filter the output with `/regexp/`.
* `i[nfo] th[read[s]]`
  * Show all threads (same as `th[read]`).
* `o[utline]` or `ls`
  * Show you available methods, constants, local variables, and instance variables in the current scope.
* `o[utline] <expr>` or `ls <expr>`
  * Show you available methods and instance variables of the given object.
  * If the object is a class/module, it also lists its constants.
* `display`
  * Show display setting.
* `display <expr>`
  * Show the result of `<expr>` at every suspended timing.
* `undisplay`
  * Remove all display settings.
* `undisplay <displaynum>`
  * Remove a specified display setting.

### Frame control

* `f[rame]`
  * Show the current frame.
* `f[rame] <framenum>`
  * Specify a current frame. Evaluation are run on specified frame.
* `up`
  * Specify the upper frame.
* `down`
  * Specify the lower frame.

### Evaluate

* `p <expr>`
  * Evaluate like `p <expr>` on the current frame.
* `pp <expr>`
  * Evaluate like `pp <expr>` on the current frame.
* `eval <expr>`
  * Evaluate `<expr>` on the current frame.
* `irb`
  * Invoke `irb` on the current frame.

### Trace

* `trace`
  * Show available tracers list.
* `trace line`
  * Add a line tracer. It indicates line events.
* `trace call`
  * Add a call tracer. It indicate call/return events.
* `trace exception`
  * Add an exception tracer. It indicates raising exceptions.
* `trace object <expr>`
  * Add an object tracer. It indicates that an object by `<expr>` is passed as a parameter or a receiver on method call.
* `trace ... /regexp/`
  * Indicates only matched events to `/regexp/`.
* `trace ... into: <file>`
  * Save trace information into: `<file>`.
* `trace off <num>`
  * Disable tracer specified by `<num>` (use `trace` command to check the numbers).
* `trace off [line|call|pass]`
  * Disable all tracers. If `<type>` is provided, disable specified type tracers.
* `record`
  * Show recording status.
* `record [on|off]`
  * Start/Stop recording.
* `step back`
  * Start replay. Step back with the last execution log.
  * `s[tep]` does stepping forward with the last log.
* `step reset`
  * Stop replay .

### Thread control

* `th[read]`
  * Show all threads.
* `th[read] <thnum>`
  * Switch thread specified by `<thnum>`.

### Configuration

* `config`
  * Show all configuration with description.
* `config <name>`
  * Show current configuration of <name>.
* `config set <name> <val>` or `config <name> = <val>`
  * Set <name> to <val>.
* `config append <name> <val>` or `config <name> << <val>`
  * Append `<val>` to `<name>` if it is an array.
* `config unset <name>`
  * Set <name> to default.
* `source <file>`
  * Evaluate lines in `<file>` as debug commands.
* `open`
  * open debuggee port on UNIX domain socket and wait for attaching.
  * Note that `open` command is EXPERIMENTAL.
* `open [<host>:]<port>`
  * open debuggee port on TCP/IP with given `[<host>:]<port>` and wait for attaching.
* `open vscode`
  * open debuggee port for VSCode and launch VSCode if available.
* `open chrome`
  * open debuggee port for Chrome and wait for attaching.

### Help

* `h[elp]`
  * Show help for all commands.
* `h[elp] <command>`
  * Show help for the given command.


# Configuration

You can configure the debugger's behavior with the `config` command and environment variables.

Every configuration has a corresponding environment variable, for example:

```
config set log_level INFO # RUBY_DEBUG_LOG_LEVEL=INFO
config set no_color true  # RUBY_DEBUG_NO_COLOR=true
```



- UI
  - `RUBY_DEBUG_LOG_LEVEL` (`log_level`): Log level same as Logger (default: WARN)
  - `RUBY_DEBUG_SHOW_SRC_LINES` (`show_src_lines`): Show n lines source code on breakpoint (default: 10)
  - `RUBY_DEBUG_SHOW_FRAMES` (`show_frames`): Show n frames on breakpoint (default: 2)
  - `RUBY_DEBUG_USE_SHORT_PATH` (`use_short_path`): Show shorten PATH (like $(Gem)/foo.rb) (default: false)
  - `RUBY_DEBUG_NO_COLOR` (`no_color`): Do not use colorize (default: false)
  - `RUBY_DEBUG_NO_SIGINT_HOOK` (`no_sigint_hook`): Do not suspend on SIGINT (default: false)
  - `RUBY_DEBUG_NO_RELINE` (`no_reline`): Do not use Reline library (default: false)
  - `RUBY_DEBUG_NO_HINT` (`no_hint`): Do not show the hint on the REPL (default: false)

- CONTROL
  - `RUBY_DEBUG_SKIP_PATH` (`skip_path`): Skip showing/entering frames for given paths
  - `RUBY_DEBUG_SKIP_NOSRC` (`skip_nosrc`): Skip on no source code lines (default: false)
  - `RUBY_DEBUG_KEEP_ALLOC_SITE` (`keep_alloc_site`): Keep allocation site and p, pp shows it (default: false)
  - `RUBY_DEBUG_POSTMORTEM` (`postmortem`): Enable postmortem debug (default: false)
  - `RUBY_DEBUG_FORK_MODE` (`fork_mode`): Control which process activates a debugger after fork (both/parent/child) (default: both)
  - `RUBY_DEBUG_SIGDUMP_SIG` (`sigdump_sig`): Sigdump signal (default: false)

- BOOT
  - `RUBY_DEBUG_NONSTOP` (`nonstop`): Nonstop mode (default: false)
  - `RUBY_DEBUG_STOP_AT_LOAD` (`stop_at_load`): Stop at just loading location (default: false)
  - `RUBY_DEBUG_INIT_SCRIPT` (`init_script`): debug command script path loaded at first stop
  - `RUBY_DEBUG_COMMANDS` (`commands`): debug commands invoked at first stop. commands should be separated by ';;'
  - `RUBY_DEBUG_NO_RC` (`no_rc`): ignore loading ~/.rdbgrc(.rb) (default: false)
  - `RUBY_DEBUG_HISTORY_FILE` (`history_file`): history file (default: ~/.rdbg_history)
  - `RUBY_DEBUG_SAVE_HISTORY` (`save_history`): maximum save history lines (default: 10000)

- REMOTE
  - `RUBY_DEBUG_PORT` (`port`): TCP/IP remote debugging: port
  - `RUBY_DEBUG_HOST` (`host`): TCP/IP remote debugging: host (default: 127.0.0.1)
  - `RUBY_DEBUG_SOCK_PATH` (`sock_path`): UNIX Domain Socket remote debugging: socket path
  - `RUBY_DEBUG_SOCK_DIR` (`sock_dir`): UNIX Domain Socket remote debugging: socket directory
  - `RUBY_DEBUG_LOCAL_FS_MAP` (`local_fs_map`): Specify local fs map
  - `RUBY_DEBUG_SKIP_BP` (`skip_bp`): Skip breakpoints if no clients are attached (default: false)
  - `RUBY_DEBUG_COOKIE` (`cookie`): Cookie for negotiation
  - `RUBY_DEBUG_OPEN_FRONTEND` (`open_frontend`): frontend used by open command (vscode, chrome, default: rdbg).
  - `RUBY_DEBUG_CHROME_PATH` (`chrome_path`): Platform dependent path of Chrome (For more information, See [here](https://github.com/ruby/debug/pull/334/files#diff-5fc3d0a901379a95bc111b86cf0090b03f857edfd0b99a0c1537e26735698453R55-R64))

- OBSOLETE
  - `RUBY_DEBUG_PARENT_ON_FORK` (`parent_on_fork`): Keep debugging parent process on fork (default: false)

## Initialization scripts

If you want to run certain commands or set configurations for every debugging session automatically, you can put them into the `~/.rdbgrc` file.

If you want to run additional initial scripts, you can also,

- Use `RUBY_DEBUG_INIT_SCRIPT` environment variable can specify the initial script file.
- Specify the initial script with `rdbg -x initial_script`.

Initial scripts are useful to write your favorite configurations.  For example,

```
config set use_short_path true # Use $(Gem)/gem_content to replace the absolute path of gem files
```

Finally, you can also write the initial script in Ruby with the file name `~/.rdbgrc.rb`.

## rdbg command help

```
exe/rdbg [options] -- [debuggee options]

Debug console mode:
    -n, --nonstop                    Do not stop at the beginning of the script.
    -e DEBUG_COMMAND                 Execute debug command at the beginning of the script.
    -x, --init-script=FILE           Execute debug command in the FILE.
        --no-rc                      Ignore ~/.rdbgrc
        --no-color                   Disable colorize
        --no-sigint-hook             Disable to trap SIGINT
    -c, --command                    Enable command mode.
                                     The first argument should be a command name in $PATH.
                                     Example: 'rdbg -c bundle exec rake test'

    -O, --open=[FRONTEND]            Start remote debugging with opening the network port.
                                     If TCP/IP options are not given, a UNIX domain socket will be used.
                                     If FRONTEND is given, prepare for the FRONTEND.
                                     Now rdbg, vscode and chrome is supported.
        --sock-path=SOCK_PATH        UNIX Domain socket path
        --port=PORT                  Listening TCP/IP port
        --host=HOST                  Listening TCP/IP host
        --cookie=COOKIE              Set a cookie for connection

  Debug console mode runs Ruby program with the debug console.

  'rdbg target.rb foo bar'                starts like 'ruby target.rb foo bar'.
  'rdbg -- -r foo -e bar'                 starts like 'ruby -r foo -e bar'.
  'rdbg -c rake test'                     starts like 'rake test'.
  'rdbg -c -- rake test -t'               starts like 'rake test -t'.
  'rdbg -c bundle exec rake test'         starts like 'bundle exec rake test'.
  'rdbg -O target.rb foo bar'             starts and accepts attaching with UNIX domain socket.
  'rdbg -O --port 1234 target.rb foo bar' starts accepts attaching with TCP/IP localhost:1234.
  'rdbg -O --port 1234 -- -r foo -e bar'  starts accepts attaching with TCP/IP localhost:1234.
  'rdbg target.rb -O chrome --port 1234'  starts and accepts connecting from Chrome Devtools with localhost:1234.

Attach mode:
    -A, --attach                     Attach to debuggee process.

  Attach mode attaches the remote debug console to the debuggee process.

  'rdbg -A'           tries to connect via UNIX domain socket.
                      If there are multiple processes are waiting for the
                      debugger connection, list possible debuggee names.
  'rdbg -A path'      tries to connect via UNIX domain socket with given path name.
  'rdbg -A port'      tries to connect to localhost:port via TCP/IP.
  'rdbg -A host port' tries to connect to host:port via TCP/IP.

Other options:
    -h, --help                       Print help
        --util=NAME                  Utility mode (used by tools)
        --stop-at-load               Stop immediately when the debugging feature is loaded.

NOTE
  All messages communicated between a debugger and a debuggee are *NOT* encrypted.
  Please use the remote debugging feature carefully.

```

# Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/ruby/debug.
This debugger is not mature so your feedback will help us.

Please also check the [contributing guideline](/CONTRIBUTING.md).

# Acknowledgement

- Some tests are based on [deivid-rodriguez/byebug: Debugging in Ruby 2](https://github.com/deivid-rodriguez/byebug)
- Several codes in `server_cdp.rb` are based on [geoffreylitt/ladybug: Visual Debugger](https://github.com/geoffreylitt/ladybug)
