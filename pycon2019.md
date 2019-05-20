# PyCon 2019 Notes

<details>
  <summary>Other pycon2019 notes</summary>

  * walrus operator: temp saving / naming of var to lessen lines of code
  * other suggested talks: lambda calc good speaker, break cycle automate tasks, take out rubbish garbage collector, 8 things that happen after the dot, the perils of inheritance, python security tools
  * other non-pycon2019 suggested talks: 2016 russia concurrency talk raymond h, pycon2018 logging mario corchero
  * per open spaces maintainer: virtualenv more maintained than venv
</details>

## Talk: Exceptional Exceptions
`except my_exc_func():` is valid

Be careful of returns in try/except/else/finally blocks - return in finally can hide all errors

Use `repr` of exeception not `str` to get type, `str(ValueError(1)) = '1'` `repr(ValueError(1)) = 'ValueError(1)'`

This code fails to handle first exception, now 2 errors: `"During handling of the above exception..."`:

```
try:
    ...
except Exception:
    logger.debug("...")
    raise Exception
```

Can handle 1st error by just `raise` or if adding text / changing type then use `from None`:

```
>>> try:
...     1/0
... except ZeroDivisionError as exc:
...     logging.info("...", exc_info=True)
...     raise ValueError(f'Bad value in process X: {exc!r}') from None
... 
Traceback (most recent call last):
  File "<stdin>", line 5, in <module>
ValueError: Bad value in process X: ZeroDivisionError('division by zero')
```

<details>
  <summary>Other talk notes</summary>

  * if `except non_existing_var:` not syntax error only runtime error

  * `finally` runs after any good/bad run, `else` runs if no exceptions

  * `logging.info(“...” exc_info=True)` or `logging.exception(“...”)` - both log traceback but diff log level

  * Many use `NotImplemented` but that is not error, easy to miss

  * Python3 uses `exc.args` no longer `exc.message`
</details>


## Keynote: Steering Council

* new governance chat
* pypi considering staging step before publish


## Talk: Eliot Tool - Itamar Turner-Trauring

Summary: Eliot is ~stack trace generated via logging (made for sci computing), not debuger/profiler. Shows causal trace with standard software execution.

Add a `@log_call` decorator to functions, and run `$ eliot-tree out.log` to see tree of function calls with inputs and outputs (profiler may show function but not inputs)


## Talk: Types / Taints for Security - Shannon Zhu

Problem of user input making way into execution like `os.system(cmd)`, some solutions examples with pyre (facebook team gave talk)

Type checking - can make `SafeString` type, and ensure os `cmd` uses subclass of `SafeString`. Scalable but limited since content can still be dangerous w/ double escaping or higher order shell injection

Taint analysis - can help above cases, but need manual work to define these:
* sources - of data like user input, etc
* taint - follow objects tainted with source
* sinks - sensitive functions like shell execution
* sanitizers - functions that remove taint from an object

<details>
  <summary>Other talk notes</summary>
    * ok if type checking: `command = "wget -q {}".secure_format(url)`
    * not ok, double escaping: `command = "wget -q '{}''".secure_format(url)`
    * not ok, higher order: `command = "bash -c {}".secure_format(url)`
    * not ok, if set cmd: `command = "{} -arg1 -arg2".secure_format(url)`
</details>

## Talk: Dependency hell

Example of pip installing 2 packages which depend on same package but with incompatible version ranges - for user not clear how to resolve (hope it works or give up)

7 of top 100 pypi packages conflict with each another

Causes of diamond dependency issue:
1. author not following semver "rules" introduces breaking changes in minor release

    - cause users to pin and not get latest `==`, or `>1.2.0,<1.8.0` but `1.9.0` already released, which causes...
1. use of outdated dependencies

    - X doesn't support latest but all newer packages do, so they will conflict (also miss bug fixes or security fixes)

Best practices to make authors and users play nice
1. author - follow semantic versioning "rules" to tell users about backward-incompatible changes
1. author - avoid api churn, buffer breaking changes with deprecation warnings
1. user - if using version `1.11.0` then always support the latest within major version `1` by using specifier range `>=1.11.0,<2`
1. user - when testing, test with earliest version in range (pip would get latest by default)
 
Note: A has `httplib2>=0.9.2,<1` and B has`httplib2>=0.8,<=0.11.3`
* if pip install A before B, you get latest (0.12.0) and B is not compatible
* if pip install B before A, you get 0.11.3 and all are compatible
* all OK if both ended in `,<1`

<details>
  <summary>Other notes</summary>

  Confuses some users who asked for X and Y but get warning about Q, but "hell" since even with user who understands packaging, it is not clear how to resolve
  
  requirements.txt does not guarantee the order of dependencies installed, per the docs it is arbitrary, but in practice is consistent
  
  pip check for dependency checks in virtual env  
  
</details>


## Talk: Modern Solvers - Raymond Hettinger

* Generic tools to solve complex problems outlasted custom tools
* His examples for puzzle solving (chess, go) and reinforement learning seem more simpler in a solver than in neural network deep learning frameworks
* With solver important to define well the problem in state, moves, and winning state and then it can be used
* SAT useful for dependency resolving


## Talk: API Evolution the Right Way

Overall discusses how users interact with change and get impacted by change

Ex of change: in py2 `bool(datetime.time(9, 30) == True` but `bool(datetime.time(0, 0)) == False` - fixed in py3

### Removing a method
1. `1.0` stable release, use semantic versioning to communicate pace of change
1. `1.1` add new method, mark old as deprecated
1. `2.0` remove method

### Parameter edits
1. For new parameter use default to preserve old behavior
1. If delete parameter, ensure behavior does not change silently
 * if override defaults positionally then something removed, changes behavior with no error
 * use `*` parameter syntax to fail loudly - all params after `*` can only be passed by name `def move(direction, *, x=_x_default)` - but do early, if use later it is a breaking change
 * can deprecate a param by using a sentinal value to warn if used

### Changing behavior safely
1. In minor release, add flag to opt-in to new behavior (default False, deprecate + warn if it's False)
1. In next major release, change default to True, deprecate + warn any usage of flag
1. In another major release, remove the flag entirely (Ex: `os.stat` changing return value from `int` to `float`)

Deprecation suggestions:
 * Add `warnings.warn("...", stacklevel=2)` to mark old method as deprecated with instructive error, `stacklevel` will show line number to change
 * IDE should show old method with strikethrough
 * User can use `python3 -Werror::DeprecationWarning script.py` to turn warnings into errors to ensure not using any deprecated methods
 * Tell users to upgrade to last minor release, run deprecation checks, then safe to upgrade to major

<details>
  <summary>Other notes</summary>

  * Summary
   * Evolve cautiously: Minimize features, keep features narrow, mark experimental as provisional
   * Write natural history rigorously: version scheme, change log (tell what new, what deprecated, when deprecated leaving), upgrade guide
   * Change gradually and loudly: delete features gently, add params compatibly, change behavior gradually
  * Careful new feature isn't so powerful or too generic it can do too many things
  * Con of deleting feature if user must think about how to change, versus simple mechanical find and replace
  * Version number is language to communicate pace of change for library
  * Pre-delete, give deprecation warning on usage of param via sentinal value:
   
   ```
   _x_default = object()
   
   def move(direction, x=_x_default):
      ...
      if x is not _x_default:
         warnings.warn(...)
   ```
  
</details>


## Talk: Practical Decorators

When defining decorator with func inside func pattern, outside func runs onces, inside func runs each time a decorated function runs. return value is the callable assigned back to the decorated function's name (if `@mydeco` on `add()` effectively does a `add = mydeco(add)`)

```
def mydeco(func):
    def wrapper(*args, **kwargs):
        ...
        return func(*args, **kwargs)
    return wrapper
```

Examples

1. Timing - add a time.time() before and after func, write to log, apply to functions as needed
1. Once per min - can use a decorator make sure a func runs only once per min else exception: can carry info across different runs of function with `nonlocal` - define `last_invoked = 0` in outer function and `nonlocal last_invoked` in inside wrapper function to check and update.
1. Once per n - add argument to decorator, `@once_per_n(5)` decorator to have 3 func nested, `once_per_n`, `middle`, `wrapper`
1. Memoization - caching results... if have seen same function inputs, return cached value instead of running function, `cache` dictionary in decorator outer function:
    ```
    def memoize(func):
        cache = {}
        def wrapper(*args, **kwargs):
            t = (pickle.dump(args), pick.dumps(kwargs))
            if t not in cache:
                cache[t] = func(*args, **kwargs)
            return cache[t]
        return wrapper        
    ```

1. Attributes - if want to have consistent attributes across classes that are very different (so inheritance is not preferred) can use decorator on class for this. Can do at class level and/or instance level


## Talk: Mocking and Patching Pitfalls- Mock Hell

beware over mocking. beware if tests need to know too much of implementation detail of class, violating information hiding / encapsulation

mocks are not stubs. mocks are not a tool for isolation - but a tool for exploratory design and discovery (allow testing side effects)

alternatives: fake patch, fake injection (pass dependency into method and in test substitute)

mock roles not objects (i.e. source, parser, sink) - mock source not requests and sessions

patching - should be rare and last tool used

__test double options - substitutions in tests:__
* mock - records calls to the object
* stub - returns canned data, no logic
* fake - implements fake version of production logic
* dummy - does nothing
* spy - records and delegates to the real thing


<details>
  <summary>Other notes</summary>

  * summary / opinions
    * always be refactoring
    * consider other test doubles
    * patching - should be rare and last tool used
    * mocks (if you use them)
      * should target roles not objects
      * are not just for test isolation
  * ask self: want to test on side effects on side effects or just return values?
</details>























