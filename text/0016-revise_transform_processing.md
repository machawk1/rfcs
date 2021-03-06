- Feature Name: Revise Transform Processing
- Start Date: 2018-11-15
- RFC PR: [qri-io/rfcs#24](https://github.com/qri-io/rfcs/pull/24)
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

Change transformation script execution, replacing the notion of a pipeline of uniform transform functions with set of _special functions_ that are fit-to-purpose, and a canonical _transform_ responsible for mutating the input dataset.

# Motivation
[motivation]: #motivations

While building out transform scripts, I've come to believe we need to revise our approach to resolve a design tension.

> Is the dataset that the download function is passed the original dataset in Qri? If so, then in the case when you want to combine it with an external dataset that you grab over http, is this a valid way to get them both to the transform function?
-- [source](https://github.com/qri-io/rfcs/pull/21#issuecomment-429575144)

> when I was writing a transformation, it wasn't super clear what needed to happen in the sky file and what needed to happen in the dataset.yml. I started out with both and had to deal with a bunch of errors which stemmed from (I think) me accidentally specifying things in both places or in the wrong place. In particular I wasn't sure what runs first. For example, does the ds in the transformation know the schema specified in dataset.yml or does qri check the output of transform against the specified schema?

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Current Transform Script Execution
As Qri currently exists, the primitive unit of a _transform script_ is a _transform function_, which always had the same signature:
```python
def transform_function(input_dataset):
  return result_dataset
```

A _transform script_ is a script that defines one or more transform functions, each of which is chained together in a predetermined order. The function names in order are `download`, `transform`. The reason for the separation is to separate capiblities & concerns into matching predeclared functions, establishing a declaritive sandboxing model. By chaining uniform functions we minimize side effects & establish a clear mental model of order. Here's an example:

```python
load("http.sky", "http")

def download(ds):
  res = http.get("https://api.example.com/cats.json")
  ds.set_body(res.json())
  return ds

def transform(ds):
  ds.set_meta("title", "a list of cats from example.com")
  return ds
```

This script first calls `download`, and the return value of `download` is passed as the `ds` argument to `transform`. The `http` module only allows network access within the `download` transform function, access to local dataset info is only available in the `transform` function. Sandboxing comes from use of the "hollywood principle" ("don't call us we call you"). If a `download` function is defined, the Qri environment running the transform will activate network access, call the user-defined `download` transform function, turn off network access, and move on to the next transform function. Access to the user's local repository is _not_ available during the `download` step, but is during the `transform` step.

Two important questions have been posed by early Qri users about this model:
* _What is the initial input dataset to `download`?_ -- @dustmop
* _How do I get state that isn't part of a dataset from one transform function to another?_ -- paraphrased from [@stuartlynn](https://github.com/qri-io/rfcs/pull/21#issuecomment-429575144)

These are both very good questions that point to an opportunity for improvement on our current model. 

### What is the initial input dataset to `download`?
The initial input to download is the user-supplied dataset details (eg all the things provided as arguments to the command line, or files provided over the HTTP API). This is, well, bad, because it flies directly in the face of our sandbox model. If the user is using a script on a private dataset (note that private datasets currently don't exist, but we plan on building support for them), this will lead to bad things.

Ideally the download function would only have access to the transformation configuration, and secrets. Secrets are unfortunately required as it's the only way to safely provide things like API keys and other priviledged information for constructing network requests, and transformation configuration details are important for driving logic around network request construction.

Both the required signature and the required order are getting in the way here. `download` has to be called before `transform` to give transform access to the result of network requests, and the requirement that functions have the same signature means we can't properly constrain the download function. We could pass an empty dataset with only config & secrets made available, but to me this creates confusion. Ideally download is passed nothing at all.

### How do I get state that isn't part of a dataset from one transform function to another?

This extra-textual state is a question of _routing return values_. By requiring only datasets to be passed in and out, the user is forced to compromise, returning either the user input _or_ the result of network requests. In a worst-case scenario, script authors may abuse the dataset object as a kind of transitive context, stuffing download results into, say, the `meta` dataset component, and deleting it in a later. This is highly error prone.

Ideally, consumers of the return value of `download` have access to whatever the author of the download function deems is relevant, and are not required to make any tradeoffs when delivering the results of network requests.


## Proposed Change:
I think these questions point to problems with the requirement of uniform transform function signatures, and a strict sequential call order. I have concerns about how these requirements will prevent us from adapting to new challenges in the future, and don't provide enough payoff to warrant keeping them around.

Let's have more support for the basic idea that functions that have different signatures _do different things_. That isn't really honored here. It's not clear from the above example that download and transform have any predeclared order or capabilities. Transform functions as implemented appear to differ in name only, and the predeclared call order runs the risk of becoming arbitrary. 

To solve both of these problems, I'm proposing the following changes:
* replace the notion of a transform function with a set of predefined _special functions_, where each function has a required signature that fits it's needs
* replace the "chain of transform functions" with a single function that applies programmitic changes & returns the finalized dataset
* all _special functions_ accept a passed `context` object for moving state across special function calls
* remove the requirement that the final transform function return a dataset

Here's a an example that re-writes the above transform script using the propsosed changes:

```python
load("http.sky", "http")

def download(context):
  res = http.get("https://api.example.com/cats.json")
  return res.json()

def transform(ds, context):
  cats = context.download
  ds.set_body(cats)
  ds.set_meta("title", "a list of cats from example.com")
```

Here the notion of chaining transform functions is gone. Instead, there is now one point entry, a function, `transform` that accepts a dataset. `transform` acts as a "main" function that Qri calls, passing in the user-supplied dataset snapshot (whatever the user provided from either the command line or API). `transform` retains is previous sandbox qualities: access to the user's local qri repository, but no network access, and is now always provided with the user-input dataset.

`download` is an example of a _special function_. Special functions are predefined function names and signatures the Qri environment knows to look for. Unlike our current transform functions, special functions have varying signatures, which must be conveyed through documentation. If a transform script defines a special function, Qri will call it and place the return value of the function at `context.[special_function_name]` before calling the main `transform` function. The main `transform` now also has access to the result of all _special function_ calls via the passed in `context`.

`context` is designed to solve the issue of moving state around during a transform. All special functions will accept a `context` argument. The Qri environment will populate `context` with necessary transform state such as `transform.config` and `transform.secret` values. `context` also has an API for passing arbitrary user data via `context.set("key", Value)` and `context.get("key")`.

_Note: Most special functions will not be able to reference reference state that another function procduces–even via context–because many special functions and will be called in parallel._

As a special function, `download` now has a unique signature that more closely matches it's intention. Download is intended to _download things_. It has no access to the input dataset, which is denoted by the signature. Download should do network stuff, then return the results for further processing in `transform`. Transform should modify the dataset, so it's provided a dataset param. All of this will need to be thoroughly documented, (and ideally made as a "tab completion" in the Qri frontend editor), but this change helps convey intended use.

Making the return value of `download` available at `context.download` also helps clarify to the user that by the time `transform` is executing, the qri environment has already called download. This clarifies both what is doing the calling, and when it's been called.

A side effect of these changes, transform scripts now must define a `transform` function if they wish to have any effect on a dataset. Reducing a transform to this single requirement will make for a consistent point of entry that's easier for both humans and machines to reason about.

### Remove `return ds` requirement

An additional change I'd like to make is to make the return value of `transform` optional. Users should be free to simply mutate the passed in `ds` dataset, and have that committed as the result. We'll keep the explicit `return ds` as an option in the event that scripts wish to return a distinct dataset object from the one passed in. (This currently isn't possible in Qri as we haven't provided a way to construct a dataset within a transform script, but there's a high likelihood this functionality will be introduced in the future.)

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

I'm intentionally keeping this short. Qri staff will implement this feature, and what's important is agreeing upon userland changes. The short list:

* Remove the notion of _transform functions_ from documentation
* Document all supported special functions
* New `context` object & API
* Update our tutorials

# Drawbacks
[drawbacks]: #drawbacks

### Proliferation of Function Signatures
Losing the uniform function definition requires clarification & documentation of _every_ predefined function available in a transform script. This creates additional overhead for the end user, who now needs to memorize multiple function signatures. This isn't "all bad", however, as functions whose signatures map well to their intended purpose should be easier to remember.

We plan to mitgate this in a few ways:
* good documentation
* code completion
* introducing predefined functions slowly

### "Thicker" transform functions
Using this approach moves much of the burden of doing transforms into one place. This can lead to lots of spaghetti code and not-so-encapsulated concens. One of the key ways Qri fights this is by confining bad code to a single dataset.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Global state object
A close-second alternative considered by the Qri team is to use a _global object_ to pass state around between predefined functions:

```python
load("http.sky", "http")
load("qri.sky", "qri")

def download():
  res = http.get("https://api.example.com/cats.json")
  context.set("download", res.json())

def transform(ds):
  cats = qri.results.download
  ds.set_body(cats)
  ds.set_meta("title", "a list of cats from example.com")
```

While arguably more elegant, this violates a core skylark principle:
```python
load("qri.sky", "qri")    # after this load call, skylark "freezes" all values in qri.sky 
                          # the "qri" value loaded should now be immutable

def download():
  return ["hello", "world"]

def transform(ds):
  qri.results.download     # <- this implies a value in the qri object is mutated :(
```

For this reason we switched to a passed context.

### Renaming the 'transform' function
We've also considered a number of alternative names for the function called `transform` in all of the above examples:
* `save`
* `update`
* `main`

I'm not a huge fan of `main` because this isn't code that'll work outside of the Qri environment. `save` and `update` feel like functions / methods that a user would call, not define.

# Prior art
[prior-art]: #prior-art

I haven't had time to look up prior art. We did consider stuff like [go's context package](https://golang.org/pkg/context/).

# Unresolved questions
[unresolved-questions]: #unresolved-questions

What errors (if any) should be presented to the user if:
* they define a special function with the wrong signature?
* they attempt to reference `qri.results.[special_function_name]`, but don't define the function?
* they define a special function, but never make use of `qri.results.[special_function_name]`?
Most people will do at least _some_ learning by punching the wrong stuff into Qri and negotiating the error. Providing usable error output is more important here.

In this model special functions have no access to each other's state. We often throw around the notion of defining `map` and `reduce` functions for distributed computation. This change removes the predefined dataset input, and allows for a more useful function signature for any future `map` and `reduce` functions. However, there may be some ambiguity to the user (and to use until we have a good think on it) about how exactly the `map` and `reduce` function might relate, as we have explicitly kept the state of each special function separate. 