Title: Guardrails
Date: 2024-08-10
Tags: clojure, restaurant

# Putting down the footgun

After the trials and tribulations of [my last post](6-months-1-line-of-code.html), this one was somewhat simpler. The
last item on the checklist is "turn on all error messages". In
[Code That Fits in Your Head](https://www.oreilly.com/library/view/code-that-fits/9780137464302/) the author presents
2 primary ways ... turning warnings into errors and using analysers, such as linters and code formatters.

# Reflections

Since Clojure is a dynamic language it doesn't really have the concept of compilation warnings that can be turned into
errors. So while there aren't any errors I could turn on it's generally considered good practice to be [warned about,
and remediate, the use of reflection](https://cuddly-octo-palm-tree.com/posts/2022-02-20-opt-clj-6/). Reflection is
generally a slow process in the JVM, so I want to avoid it when I can. I added the following s-expression to my
`dev/user.clj` file.

```clojure
(alter-var-root #'*warn-on-reflection* (constantly true))
```

Now, whenever I'm developing and the JVM needs to perform reflection I'll be warned about it and can add the appropriate
type hints to avoid performing reflection. The only slight annoyance here is that it looks like
[nrepl](https://github.com/nrepl/nREPL), which my editor uses, uses reflection and so I get a few warnings when I start
my REPL.

# Analysers

The Clojure ecosystem comes with a multitude code formatters and linters. For this project I chose to use
[cljfmt](https://github.com/weavejester/cljfmt) to format my code and
[clj-kondo](https://github.com/clj-kondo/clj-kondo), [eastwood](https://github.com/jonase/eastwood) and
[splint](https://github.com/noahtheduke/splint) for linting. In general, I prefer these to be integrated into my editor
to provide the fastest possible feedback loop as well as in my CI/CD pipeline to ensure consistency as my code base
evolves. 

## Code formatting

Code formatting can be a bit of a touchy subject. Some engineers prefer to trade off the consistency of automated code
formatting for the freedom to express themselves as they wish. Personally I like to have some sensible defaults and then
forget about code formatting. When writing Clojure my editor of choice is
[Intellij Idea](https://www.jetbrains.com/idea/) with the [Cursive plugin](https://cursive-ide.com/). Cursive has its
own opinions on code formatting. They're a bit different to the [Clojure style guide](https://guide.clojure.style/), but
I've happily working with them for the last few years. Selecting the option to "format on save" and I pretty much never
have to think about code formatting.

Even though this seems pretty safe, I do like to have a check in my CI/CD pipelines. While there are several code
formatting options, I went with `cljfmt` simply because it has an option to match the Cursive formatting style. To do
this I created a `.cljfmt.clj` file in the root of the project and added the following

```clojure
{:function-arguments-indentation :cursive
 :sort-ns-references?            true}
```

Since Cursive includes references in alphabetical order when it automatically imports them I added a check for this for
those occasions when I have to add references by hand.

To run `cljfmt` during CI/CD I included it from `setup-clojure` and ran it in its own step.

```yaml
...

- name: Install clojure tools
  uses: DeLaGuardo/setup-clojure@12.5
  with:
    ...
    cljfmt: 0.12.0
    ...

- name: Check formatting
  run: cljfmt check deps.edn src dev infra

...
```

## Linting

Again, there are several linters in the Clojure ecosystem and in this case I tend to find it's a case of the more, the
merrier. As long as they are decently configurable.

### clj-kondo

The OG of Clojure linters and it's able to be integrated into my editor using the
[Clojure Extras plugin](https://github.com/brcosta/clj-extras-plugin).

### eastwood and splint

Unfortunately neither of these linters are able to be integrated into Cursive. The best way I could think to add them
to my workflow was to create a script that calls them (as well as `clj-kondo` and `cljfmt`) that I could run by hand. I
run this script just before every commit I make. I created the script, `verify.sh` and placed it in the `dev` folder.
Due to the `build` alias having a `:paths` key rather than `:extra-paths` it's not possible to run the Clojure CLI tool
over the entire code base in one pass. I broke linting into "application" (including `dev`) and "infrastructure"
linting. `clj-kondo` and `cljfmt` are both available as pre-built binaries, which allows them to start up much faster
than when they are run by the Clojure CLI. This script assumes they are both available on the users' path.

```shell
#!/usr/bin/env bash

set -e

clj-kondo --lint deps.edn src dev infra
clojure -M:dev:linters -m eastwood.lint
clojure -M:dev:linters -m noahtheduke.splint
clojure -M:build:linters -m eastwood.lint
clojure -M:build:linters -m noahtheduke.splint
cljfmt check deps.edn src dev infra
```

`splint` did pick up on the fact that `restaurant.clj` is a single segment namespace. This means that that namespace
can't be called by any other Java package. Since I intend (and, yeah, this is likely to come back and bite me) for this
only to be used as an application and never as a library I created a `splint.edn` configuration file in the root of the
project and removed the warning.

```clojure
{naming/single-segment-namespace {:enabled false}}
```

Having fixed that issue, I configured my CI/CD pipeline in a similar manner to the `verify.sh` script above.

```yaml
...
- name: Install clojure tools
  uses: DeLaGuardo/setup-clojure@12.5
  with:
    ...
    clj-kondo: 2024.08.01
...
- name: Lint application
  run: |
    clj-kondo --lint deps.edn src dev
    clojure -M:dev:test:linters -m eastwood.lint
    clojure -M:dev:test:linters -m noahtheduke.splint

- name: Lint infrastructure
  run: |
    clj-kondo --lint infra
    clojure -M:build:linters -m eastwood.lint
    clojure -M:build:linters -m noahtheduke.splint
...
```

Now every change that is pushed to `main` is linted and checked for correct formatting.

# Summary

In this post I described the guardrails, linters and code formatter, I put in place to keep my code tidy, idiomatic and
to reduce the likelihood of bugs. I also talked about the development time settings I use to keep my code performant.

Next time there'll hopefully be a bit more application development and a little less yak shaving as I'll be building the
first vertical slice of the restaurant reservation system. See you then. If you've got comments or questions feel free
to reach out via Twitter/X (linked above) or on the Clojurians Slack channel.

**Previous:** [6 months, 1 line of code](./6-months-1-line-of-code.html)
