Title: Modern software engineering for a small team
Date: 2024-01-16
Tags: clojure, restaurant

# Why, oh why, oh why?

I'm a software engineer with two decades in the software industry. Over those two decades I've spent most of my time in
"brown field" code bases, code bases where I wasn't around when they were first started. I've also been working inside
larger ecosystems that provide a lot, if not all, of the surrounding infrastructure (e.g. integration and deployment
pipelines, databases, logging, etc.). This means that I rarely spin up new projects, nor do I fully understand the
fundamentals that underpin the tooling that is required to support those products. That tooling is abstracted away and
my use of it is reduced to an API call or copy, paste and small modification of a config file. This is great for working
in that particular ecosystem, but doesn't help me understand what is going on under the hood. While this understanding
clearly isn't a requirement to do my job (or I'm doing fantastically well to "faking it until I make it") it's gotten to
the point where I really would like to understand what is required and what is going on.

To explore these cracks (or more likely crevasses) in my understanding my plan is to work through a project from
scratch using what I believe to be modern software engineering techniques and processes as if I were bootstrapping a new
product with a small team. I hope to take the simplest (but not always the easiest) route possible and eschewing certain
technologies until their introduction reduces the complexity of the product. For example, I've skirted the edges of more
infrastructure focused technology (e.g. Terraform and Kubernetes) and so will start without them and intend to bring
them in only if the additional complexity simplifies the product overall.

A lot of my recent thinking about software engineering is underpinned by
[Accelerate](https://itrevolution.com/product/accelerate/) by Nicole Forsgren, Jez Humble and Gene Kim. I expect I will
often refer to the capabilities described there during this series of blog posts.

My years at the coal face have also exposed me to many ideas that would like to try, but which I haven't found space to
try out in a professional setting. I intend to experiment with them here, as and when appropriate, no doubt making
mistakes and hopefully rectifying them in future installments.

# What am I building?

Normally when I'm trying out new languages, processes and/or tools I'll build the requisite contacts or todo-list
application. The problem with those specific projects is that the vast majority of the code ends up being request
and response handling, authentication, database wrangling and the like, and very little actual business logic. While
this is fine for experimenting with new languages or tools, modern software engineering is about building software that
continuously evolves. That non-business logic scaffolding tends to be the most static part of the application from an
engineering perspective, so I'm looking for a product with plenty of business logic that can evolve over time.

I've just started reading Mark Seemann's book
[Code That Fits in Your Head](https://www.oreilly.com/library/view/code-that-fits/9780137464302/) which apparently
includes a restaurant reservation system that he builds throughout the book, starting simply and modifying and adding
new features as new use cases present themselves. Exactly the sort of product I'm looking for.

# Presumably there's a plan?

The example application in Code That Fits in Your Head is written in C#. My current tool of choice is
[Clojure](https://clojure.org/) so as I read through each chapter I will build a Clojure application using the requirements defined in
that chapter. While the code for this blog series will be in Clojure I hope to explore some wider software engineering
thoughts and opinions I have. Hopefully this means this series will be useful to more than just Clojurists.

As an aside, this isn't a beginners guide to Clojure. If you're new to Clojure and need some resources to get started
then [Practicalli](https://practical.li/clojure/), the
[Getting Started section of the Clojure website](https://clojure.org/guides/getting_started) and
[Clojure for the Brave and True](http://www.braveclojure.com/) are all excellent starting points.

# Check it out

The first practical advice in Code That Fits in Your Head provides is to use checklists. Having read Atul Gawande's
[The Checklist Manifesto](http://atulgawande.com/book/the-checklist-manifesto/) and agreed with most of it, I'm all on
board for this. The book recommends the following checklist for starting a project
- [ ] Use Git
- [ ] Automate the build
- [ ] Turn on all error messages

A checklist isn't a detailed list of everything that needs to be done, nor is it a list of requirements. It's just a
reminder of the high level, important things that need to be done. Nor is it static, checklists are expected to be
personalised and evolved over time. To me, this seems like a perfectly reasonable starting point.

# First things first

Code That Fits in Your Head starts off by creating a web server that simply returns "Hello World!" to every request, so
that's where I'll start.

## Project management

Of the 3 main project management tools available for Clojure, ([deps.edn](https://clojure.org/guides/deps_and_cli),
[leiningen](https://leiningen.org/) and [boot](https://boot-clj.github.io/)) the most I've had experience with recently
is deps.edn, so I'll go with that. Sean Corfield's [deps-new](https://github.com/seancorfield/deps-new) is one option for
generating projects from templates, but in this instance I'm going to do it by hand.

## A minimal web server

To start with I'll need a minimal directory layout

```shell
restaurant/
├── deps.edn
├── src
│   └── restaurant.clj
```

There are two primary mechanisms for server side applications in Clojure, [ring](https://github.com/ring-clojure/ring)
and [pedestal](https://github.com/pedestal/pedestal). I haven't used pedestal in anger and the ring ecosystem is rich
and vibrant so that's the one I'll be working with. With that in mind, I'll add the following to my `deps.edn` file.

```clojure
{:paths ["src"]
 :deps {org.clojure/clojure {:mvn/version "1.11.1"}
        ring/ring-jetty-adapter {:mvn/version "1.11.0"}}}
```

As mentioned above, the current requirement is for a web server that simply returns "Hello World!" as the response to
any requests. The easiest way to do this is as follows

```clojure
(ns restaurant
  (:require [ring.adapter.jetty :as jetty])
  (:gen-class))

(defn -main [& _args] (jetty/run-jetty (fn [_] {:status 200 :body "Hello World!"}) {}))
```

While this code will work in production (once I've built and deployed the service) I can't run it locally as a normal
user, either in the REPL or on the command line, because it will attempt to bind to port 80 which normal users cannot
do. Assuming I'm not running anything else on port 80, I can escalate my privileges and check that it works. On my linux
box that's

```shell
$ sudo clojure -M -m restaurant
```

and I can then access the server using curl (or your command line web client of choice).

```shell
$ curl localhost
Hello World!
```

Although it works, this is a pretty hostile situation for a developer, especially as complexity increases.

## Getting Git'ty with it

Now that I've got something working I check my checklist and the first thing I find is "Use Git". With that in mind I do
the following:
- initialise our repository with Git
- create an [initial, empty commit](https://www.garfieldtech.com/blog/git-empty-commit)
- create a `.gitignore` file and add in any files created by our IDE and anything else that we haven't explicitly
created
- stage the code and commit it

## Where'd my branch go?

One of the capabilities outlined in Accelerate is "Trunk-based development". This means either working with very
short-lived feature branches or no branches at all. Because I'll be the only person working on this product I intend to
go with no branching at all. Once I start deploying to production I'll need some safeguards, but for now I'll commit
straight to the trunk in as small commits as make sense.

## Simplicity itself

While the implementation of the server _technically_ works, I tend to follow the mis-quoted aphorism that
["code should be as simple as possible, and no simpler"](https://quoteinvestigator.com/2011/05/13/einstein-simple/). I
actually think this code is too simple. It doesn't clean up after itself nor is it developer and REPL friendly. With
that in mind I extract out `start-server` and `stop-server` functions and gracefully shut down the server when the
service is shut down.

```clojure
(ns restaurant
  (:require [ring.adapter.jetty :as jetty])
  (:import (org.eclipse.jetty.server Server))
  (:gen-class))

(defn start-server
  ([] (start-server {}))
  ([config] (jetty/run-jetty (fn [_] {:status 200 :body "Hello World!"}) config)))

(defn stop-server [server]
  (.stop ^Server server))

(defn -main [& _args]
  (let [server (start-server)]
    (.addShutdownHook
      (Runtime/getRuntime)
      (Thread. ^Runnable (fn [] (stop-server server))))))

(comment
  (def server (start-server {:port 3000 :join? false}))
  (stop-server server))
```

By passing in config I can now use the default config for production and modify it simply at development time. Here I
run the server on port 3000 and setting `:join?` to `false` means the server doesn't block our REPL thread.

## What's that noise?

Great. I can now start and stop our web server in the REPL, a much friendlier developer experience. The only problem is
that when I do start the application/REPL I get the following warning

```shell
SLF4J: No SLF4J providers were found.
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See https://www.slf4j.org/codes.html#noProviders for further details.
```

Looks like our server uses SLF4J for logging, so I should provide a logger implementation. I could use the no-op logger,
but I tend to like being informed when stuff goes wrong. With that in mind I use the SLF4J SimpleLogger by adding it to
our dependencies

```clojure
{:paths   ["src"]
 :deps    {org.clojure/clojure {:mvn/version "1.12.0-alpha5"}
           org.slf4j/slf4j-simple {:mvn/version "2.0.10"}
           ring/ring-jetty-adapter {:mvn/version "1.11.0"}}}
```

Now when I start the application I no longer get the warning. Once I start the server, however, we get the following log
messages

```shell
[nREPL-session-aca1af76-ccda-46d5-b745-63e29dc26d52] INFO org.eclipse.jetty.server.Server - jetty-11.0.18; built: 2023-10-27T02:14:36.036Z; git: 5a9a771a9fbcb9d36993630850f612581b78c13f; jvm 21.0.1+12-29
[nREPL-session-aca1af76-ccda-46d5-b745-63e29dc26d52] INFO org.eclipse.jetty.server.handler.ContextHandler - Started o.e.j.s.ServletContextHandler@cc0cd0{/,null,AVAILABLE}
[nREPL-session-aca1af76-ccda-46d5-b745-63e29dc26d52] INFO org.eclipse.jetty.server.AbstractConnector - Started ServerConnector@4ce608d3{HTTP/1.1, (http/1.1)}{0.0.0.0:3000}
[nREPL-session-aca1af76-ccda-46d5-b745-63e29dc26d52] INFO org.eclipse.jetty.server.Server - Started Server@36b520a7{STARTING}[11.0.18,sto=0] @23056ms
```

While this information can be useful, unless I'm being warned about something I don't want this noise distracting me,
and potentially hiding more important information, during development. I still want to know about warnings and errors,
so I need to change the minimum logging level to `Warn`. There's a couple of ways to do that, but my preferred is to do
it programmatically.

## Introducing "dev"

Clojure has a concept of aliases to add or remove functionality depending on the situation. In this case, when I'm
developing I want to set the minimum log level to `Warn`. I add an alias called `dev` to deps.edn

```clojure
{:paths ["src"]
 :deps {org.clojure/clojure {:mvn/version "1.12.0-alpha5"}
        org.slf4j/slf4j-simple {:mvn/version "2.0.10"}
        ring/ring-jetty-adapter {:mvn/version "1.11.0"}}
 :aliases {:dev {:extra-paths ["dev"]}}}
```

and a `dev/user.clj` file to our project

```shell
restaurant/
├── deps.edn
├── dev
│   └── user.clj
├── src
│   └── restaurant.clj
```

The `dev/user.clj` file will be loaded when the REPL starts, so I set the minimum log level in there

```clojure
(ns user
  (:import (org.slf4j.simple SimpleLogger)))

(System/setProperty SimpleLogger/DEFAULT_LOG_LEVEL_KEY "Warn")
```

Now when I start the server from the REPL I no longer get the information log lines, leading to a cleaner development
experience.

# Summary

In this first blog post I talked about why I'm creating this series, how it will (hopefully) demonstrate modern software
development techniques and processes and I built a simple, first cut of a web server. The project so far can be found
in a [GitHub repository](https://github.com/HughPowell/restaurant). In my next post I'll look at the second item on the
checklist, "Automate the build". If you've got comments or questions feel free to reach out via Twitter/X (linked above)
or on the Clojurians Slack channel.
