# CoMix
Common mix.exs code that all projects can benefit from

## Features
- `CoMix.version/0`: Enables easy versioning in any git project through git tags
  - One can run `git flow release start $VERSION; git flow release finish $VERSION` and be done
  - If git errors instead of getting a tag, and the environment variable `CO_MIX_TAGLESS` is set, a version suffixed with `-tagless` is used

## Installation
Add this to the top of your mix.exs file:
```elixir
unless Kernel.function_exported?(CoMix, :version, 0) do
  {:ok, _} = Application.ensure_all_started(:hex)
  Mix.install([{:co_mix, "~> 1.0", runtime: false}])
end
```
Then set `version = CoMix.version()` inside of the `def project`, instead of using a hardcoded version.

### Troubleshooting
#### Issue: Adding the `Mix.install` line to `mix.exs` causes `mix` commands like `mix deps.get` to error:
```
  (exit) exited in: GenServer.call(Hex.Registry.Server, {:open, []}, 60000)
     (EXIT) no process: the process is not alive or there’s no process currently associated with the given name, possibly because its application isn’t started
    (elixir 1.13.4) lib/gen_server.ex:1019: GenServer.call/3
    (hex 1.0.1) lib/hex/remote_converger.ex:28: Hex.RemoteConverger.converge/2
    (mix 1.13.4) lib/mix/dep/converger.ex:95: Mix.Dep.Converger.all/4
    (mix 1.13.4) lib/mix/dep/converger.ex:51: Mix.Dep.Converger.converge/4
    (mix 1.13.4) lib/mix/dep/fetcher.ex:16: Mix.Dep.Fetcher.all/3
    (mix 1.13.4) lib/mix/tasks/deps.get.ex:31: Mix.Tasks.Deps.Get.run/1
    (mix 1.13.4) lib/mix/task.ex:397: anonymous fn/3 in Mix.Task.run_task/3
    (mix 1.13.4) lib/mix/cli.ex:84: Mix.CLI.run_task/2
```

##### Solution
Be sure to include the call to `Application.ensure_all_started(:hex)` before calling `Mix.install/2`, to explicitly ensure hex is started.


#### Issue: mix commands fail with Hex 2.0
Since updating to Hex 2.0, Mix commands fail with an error similar to
```
** (Protocol.UndefinedError) protocol String.Chars not implemented for %Hex.Solver.Constraints.Range{min: %Version{major: 0, minor: 2, patch: 0}, max: %Version{major: 0, minor: 3, patch: 0, pre: [0]}, include_min: true, include_max: false} of type Hex.Solver.Constraints.Range (a struct). This protocol is implemented for the following type(s): Atom, BitString, Date, DateTime, Float, Integer, List, NaiveDateTime, Time, URI, Version, Version.Requirement
    (elixir 1.14.1) lib/string/chars.ex:3: String.Chars.impl_for!/1
    (elixir 1.14.1) lib/string/chars.ex:22: String.Chars.to_string/1
    (hex 2.0.0) lib/hex/mix.ex:168: Hex.Mix.registry_dep_to_def/1
    (elixir 1.14.1) lib/enum.ex:1658: Enum."-map/2-lists^map/1-0-"/2
    (hex 2.0.0) lib/hex/mix.ex:120: anonymous fn/1 in Hex.Mix.to_lock/1
    (elixir 1.14.1) lib/enum.ex:1658: Enum."-map/2-lists^map/1-0-"/2
    (elixir 1.14.1) lib/enum.ex:1658: Enum."-map/2-lists^map/1-0-"/2
    (elixir 1.14.1) lib/map.ex:263: Map.new_from_enum/2
```

##### Solution
This is due to a known bug that was fixed in Elixir 1.14.2, update and try again.

#### Issue: Commands like `mix local.hex --force` are failing (common for Travis):
```
** (MatchError) no match of right hand side value: {:error, {:hex, {'no such file or directory', 'hex.app'}}}
    mix.exs:1: (file)
    (mix 1.13.2) lib/mix/cli.ex:42: Mix.CLI.load_mix_exs/0
    (mix 1.13.2) lib/mix/cli.ex:29: Mix.CLI.proceed/1
The command "mix local.hex --force" failed and exited with 1 during .
```

##### Solution
This is because `mix.exs` needs to ensure hex is started, so the machine needs to install hex first before mix can run with the updated mix.exs
1. `cd` over to a directory without a `mix.exs`
    - Short and to the point, `cd` will take you to the home dir, which is unlikely to have this file
2. Run whichever mix commands were failing
3. Run `cd -` to return to your previous directory, and proceed as usual

In `.travis.yml`, this usually looks something like
```yaml
install:
  - cd
  - mix do local.rebar --force, local.hex --force
  - cd -
```

#### Issue: The release build is failing to upload from Travis:
##### Solution
First, make sure that everything is being run with `MIX_ENV` set the same (or defaulted)

There may be a script trying to pull the version by reading mix.exs
A good solution to this that we've found is to look for and update these scripts:
- If they already have hex installed, go ahead and use `mix app.version`
  - If you don't have this command yet, it's easy to implement, see [here](https://mintcore.se/blog/2017/11/getting-elixir-app-version-from-command-line) ([Archived](https://web.archive.org/web/20200920053411/https://mintcore.se/blog/2017/11/getting-elixir-app-version-from-command-line))
- In other cases, it may be cleaner to have the script call the binary, after it has been built, with a `version` argument
Such solutions can look like `VERSION=$(_build/prod/rel/my_app/bin/my_app version | cut -d ' ' -f2)`

## Testing
Testing is done by running `mix test`.

## Releasing
You should be using [git flow](https://github.com/petervanderdoes/gitflow-avh/wiki/Installation) here.

However, since this repo enables simplified versioning, we have chosen for the moment not to have it depend on itself.

As such, after running `git flow release start <X.Y.Z>`, you must create a commit bumping the version in mix.exs before running `git flow release finish <X.Y.Z>`.
