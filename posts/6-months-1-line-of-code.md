Title: 6 months, 1 line of code
Date: 2024-07-27
Tags: clojure, infrastructure, restaurant

# That was a ... journey
So it's been somewhat longer than I'd hoped. After
[the first blog post in this series](./modern-software-engineering-for-a-small-team.html) I started on the second item in
the checklist, "Automate the build". And that's where I got suited up and blasted off as a fully fledged [architecture
astronaut](https://en.wikipedia.org/wiki/Architecture_astronaut).

# Constructing the monolith

Everything started off reasonably sensibly. Since I'm using Clojure's
[deps system](https://clojure.org/guides/deps_and_cli) to configure my application it made sense to use the
[tools.build](https://clojure.org/guides/tools_build) library to build an UberJAR for the service. Because of the
simplicity of the application, so far, it was basically a copy-&-paste job of the
[tools.build example](https://clojure.org/guides/tools_build).

The first step was to create a `:build` alias in my `deps.edn` file to manage the build.
```clojure
{...
 :aliases {...
           :build {:paths ["infra"]
                   :deps {io.github.clojure/tools.build {:mvn/version "0.9.6"}}
                   :ns-default build}}}
```

I like to keep all my infrastructure/configuration away from the rest of my code, so I created an `infra` directory and
I created a `build.clj` file to house the build code.

```shell
restaurant
...
├── infra
│   └── build.clj
...
```

I then copied the example build namespace from the guide. Including the build number in the file name of the JAR makes
life a little more complicated later, so I removed it and tided up the rest of the file.

```clojure
(ns build
    (:require [clojure.tools.build.api :as b]))

(def target-dir "target")
(def class-dir (format "%s/classes" target-dir))

;; delay to defer side effects (artifact downloads)
(def basis (delay (b/create-basis {:project "deps.edn"})))

(defn clean [_]
  (b/delete {:path target-dir}))

(defn uber [_]
  (clean nil)
  (b/copy-dir {:src-dirs ["src"]
               :target-dir class-dir})
  (b/compile-clj {:basis @basis
                  :class-dir class-dir})
  (b/uber {:class-dir class-dir
           :uber-file (format "%s/restaurant.jar" target-dir)
           :basis @basis
           :main 'restaurant}))
```

This allowed me to create the UberJAR from either the REPL (as long as it's started with the `build` alias)

```clojure
(build/uber nil)
```

or the command line.

```shell
clojure -X:build uber
```

Both of those commands creates the UberJAR at `target/restaurant.jar`.

I was now able to run the application UberJAR from the command line

```shell
sudo java -jar target/restaurant.jar
```

and then check that it worked using [curl](https://curl.se/) (or your web browser of choice)

```shell
$ curl localhost
Hello World!
```

# Shooting for the stars

I had recently come across
[Large Scale Software Development (and a big trap)](https://www.youtube.com/watch?v=slV0zdUEYJw) by the YouTube channel
[Code to the Moon](https://www.youtube.com/@codetothemoon) that nicely encapsulates the architecture I wanted to use.
Essentially, I wanted to architect the service as a stateless monolith that is run behind a load balancer. This allows
for fairly straight forward scaling (both horizontally and vertically) and downtimeless deployments, two things that
modern software engineering embraces. In my experience this is a suitable architecture for the vast majority of projects
to begin with, and for a lot will be all the architecture that will ever be needed.

## Wishlist

"All" I wanted was somewhere I could deploy my service that
* was a Platform-as-a-Service (PaaS) that aligned with the architecture defined above
* was API and/or CLI first
* could run a javaagent alongside my service
* would not let me deploy a trashed version of the service
* was cheap

### Platform-as-a-service

Because I'm a developer most of the time my ops skills are a gaping chasm of desirability. Could I learn to cobble
together something in AWS, GCP or Azure, probably? Could I do something similar in kubernetes (or k8s or whatever it's
called), it's probably not beyond me? Did I _want_ to spend a bunch of time doing that? Nope. Would it have been secure,
performant or reliable? Highly unlikely. All I wanted was someone to do the hard work of putting all the pieces together
and present them on a platter to me. Not too much to ask, I think you'll agree.

### API/CLI first

My intent is to deploy this service in small increments and very often using some sort of CI/CD tooling, so at the very
least the deployments must be automatable. If everything else is as well, that will allow me to build any tooling I
might need. I don't want your easy "one button deployment" I want a simple "one call deployment".

### Run a javaagent

Logging has been a wonderful asset over the last few decades, but I really believe that tracing (via
[OpenTelemetry](https://opentelemetry.io/)) and the observability it provides is the future of understanding live
systems. I suspect it will supersede most, if not all, use cases for logging (not with the certainty that certain
corners of the internet believed Blockchain was the future finance or similar corners that A.I. is the future of
everything, but pretty close). OpenTelemetry provides a Java agent that can automatically instrument a JVM application
using libraries that it knows about. E.g. it will instrument the Jetty HTTP server because that's widely known enough
that the good folk over at the OpenTelemetry project have created automatic instrumentation for it as opposed to the
HTTPKit server, which isn't as well known. For me to benefit from this automated tracing I need to be able to start my
application with this Java agent. 

### Won't let me deploy trashed service versions

I'm a firm believer in [trunk based development](https://trunkbaseddevelopment.com/), or at the very least, short lived
feature branches. Ideally I was looking for a service that would provide
[canary releases](https://martinfowler.com/bliki/CanaryRelease.html) which would allow me to progressively roll out new
versions of my service to increasingly larger percentages of my users. At the very least I wanted somewhere that would
not allow me to deploy obviously broken versions of my service.

### Cheap

I'm only a lowly software engineer, this isn't an attempt to make money and I already have plenty of expensive hobbies.
I don't need to add cloud computing to that list.

## Google-fu failures

I'm not proud to say it, but in this moment, my Google-fu failed me. [Heroku](https://www.heroku.com/) looked promising
as it could host the UberJAR, but there was no way that I could find to run the OpenTelemetry javaagent alongside it. 
[Railway](https://railway.app/) probably should have been enough, but the documentation didn't immediately click and
somehow I missed [fly.io](https://fly.io) (we'll get back to that one).

## Blasting off

So I did what any self-respecting (some might say foolish or naive) software engineer suffering from "not invented here"
syndrome would do and spent the next 6 months, off and on, building my deployment process through SSH to a
[Digital Ocean](https://www.digitalocean.com/) droplet. Had I not been suffering from said syndrome I could have spent
that time learning something useful, like k8s or AWS. Don't get me wrong, I learnt a lot about Docker's API, how to
use [lispyclouds contajners library](https://github.com/lispyclouds/contajners) and
[Metosin's sieppari](https://github.com/metosin/sieppari), and programmatically using SSH, but it appears building a
PaaS isn't as easy as it first looks. Who knew?

## Crashing back to earth

And then fly.io turned up in one of my social media feeds (I suspect it was
[The Primeagen](https://www.youtube.com/@ThePrimeTimeagen), but I can't find the video now). It's a PaaS primarily
driven through the CLI, it runs Docker containers so I can run a javaagent alongside my service, it's got multiple
deployment options and it's cheap. Plus it handles SSL certs, secret management and has GitHub Actions integrations.
Tick, tick, tick, tick. Okay, it's got a release option called "Canary" that isn't what I would describe as a canary
release, but at least it should be enough to stop me shooting myself in the foot. If I'm reasonably careful.

# Containing the UberJAR

Since I'm not a Docker expert I decided to steal the expertise of other humans who actually know what they're doing.
[Practically](https://practical.li) has some
[excellent documentation](https://practical.li/engineering-playbook/continuous-integration/Docker/) on building Docker
images for Clojure applications. [Andrey Fadeev](https://www.youtube.com/@andrey.fadeev) has a really useful
[YouTube video](https://www.youtube.com/watch?v=2dzHJx_hUDk&t=9s) about Docker and Clojure that helped it click for me.
In fact, all his videos are great, if you're looking to step up your Clojure game you could do a lot worse than work
your way through his back catalogue.

I created a following `Dockerfile` in the `infra` directory.

```shell
restaurant
...
├── infra
│   └── ...
│   └── Dockerfile
...
```

The strategy was to use a fully featured base to build the UberJAR and then a very thin base to run service itself.

```dockerfile
FROM clojure:temurin-21-alpine AS builder

RUN mkdir -p /build
WORKDIR /build

COPY deps.edn /build/
RUN clojure -P -X:build

COPY ./src /build/src
RUN mkdir -p /build/infra
COPY ./infra/build.clj /build/infra/
RUN clojure -T:build uber

FROM eclipse-temurin:21-jre-alpine AS final

LABEL org.opencontainers.image.source=https://github.com/HughPowell/restaurant
LABEL org.opencontainers.image.description="Restaurant reservation application"
LABEL org.opencontainers.image.licenses="MPL-2.0"

RUN apk add --no-cache \
    dumb-init~=1.2.5

ARG UID=10001
RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "${UID}" \
    clojure

RUN mkdir -p /service && chown -R clojure. /service

USER clojure

WORKDIR /service
COPY --from=builder --chown=clojure:clojure /build/target/restaurant.jar /service/restaurant.jar

ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["java", "-jar", "/service/restaurant.jar"]
```

Since I was defining a license here I also included the text of the license in a LICENSE file at the root of the
project. My go to is the [Mozilla Public License V2 (MPL-2.0)](https://www.mozilla.org/en-US/MPL/2.0/).

# Flying back to the moon

Now this is what I call a [quick start guide](https://fly.io/docs/getting-started/launch/). Install, signup, launch
(which generated a local config file in `infra/fly.toml`), deploy, all from the CLI. Just one small problem. It wouldn't
let me (probably quite reasonably) expose port 80 on the docker container, so I decided to update the application to
listen on port 3000 and then exposed that port from the Dockerfile. I didn't need to change the port the service ran on,
but it meant the added benefit that I could run the UberJAR locally without needing `sudo`. Once that was fixed up I
successfully deployed the service and for good measure I pointed a subdomain of my `hughpowell.net` domain name to it.

Now all I needed was a way to prevent me shooting myself in the foot as often as possible. Fly.io provides a number of
[deployment strategies](https://fly.io/docs/launch/deploy/#deployment-strategy) including what they call 'canary'. This
boots a single additional machine, waits for it to become healthy and then replaces all the running machines one-by-one.
Not how I would describe a canary deployment strategy, but enough for the time being.

To enable this I set up a `[deploy]` section in my `infra/fly.toml` file like so

```toml
[deploy]
  strategy = 'canary'
  max_unavailable = 1
  wait_timeout = "1m"
```

With my deployments now configured all I needed was to automate them every time a change was committed to `main`. I
chose GitHub Actions as my CI/CD pipeline orchestrator of choice. No great reason, it's integrated into GitHub, I've
used it a couple of times before and it seems as competent as anything else at this scale. Having added my fly.io API
token to the `FLY_API_TOKEN` secret I check out the project, install the `flyctl` application and deploy the service. I
limit the job to 15 minutes because I never want build times to exceed that.

```yaml
name: Build the Restaurant service
on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup deployment controller
        uses: superfly/flyctl-actions/setup-flyctl@master

      - name: Deploy service
        run: flyctl deploy --config infra/fly.toml
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

So, yeah. The last six months has basically boiled down to `flyctl deploy --config infra/fly.toml` and a lot of
learning.

# What's going on?

Now that my service is running in production I need a way to check that it's doing everything it should do and nothing
it shouldn't. As noted above I wanted to use OpenTelemetry for that. Instrumenting the service locally required
downloading the [OpenTelemetry Java agent](https://opentelemetry.io/docs/zero-code/java/agent/getting-started/) and then
starting the UberJAR with the `-javaagent:<path-to-OpenTelemetry-agent>` flag. You can also pass a similar option to
your REPL and you'll get traces while developing. There's a couple of options for interacting with OpenTelemetry
locally. You can export the traces [to the console](https://opentelemetry.io/docs/languages/java/exporters/#console) or
to an observability tool like [SigNoz](https://signoz.io/), my current tool of choice, or [Digma](https://digma.ai/).

## What about the old school logs?

Unfortunately OpenTelemetry hasn't penetrated the entire Java ecosystem (yet!) so I still needed a way to output logs
that are being written by the libraries I have and will import. Luckily the OpenTelemetry defines a standard which the
Java agent has implemented to facilitate recording logs as spans.

First off I had to deal with the chaos that is
[Clojure logging infrastructure](https://lambdaisland.com/blog/2020-06-12-logging-in-clojure-making-sense-of-the-mess).
To be fair it's mostly Java's fault. Either way, the magic incantations were to add a stack of dependencies

```clojure
{...
 :deps    {...
           ch.qos.logback/logback-classic {:mvn/version "1.2.3"}
           org.slf4j/jcl-over-slf4j       {:mvn/version "1.7.30"}
           org.slf4j/jul-to-slf4j         {:mvn/version "1.7.30"}
           org.slf4j/log4j-over-slf4j     {:mvn/version "1.7.30"}
           org.slf4j/osgi-over-slf4j      {:mvn/version "1.7.30"}
           org.slf4j/slf4j-api            {:mvn/version "1.7.30"}
           ...                                                                                        
           }
 ...
}
```

the OpenTelemetry logback appender

```clojure
{...
 :deps    {...
           io.opentelemetry.instrumentation/opentelemetry-logback-appender-1.0 {:mvn/version "2.6.0-alpha"}
           ...
           }
 ...
 }
```
and then deregister the console log appender and create an OpenTelemetry one

```clojure
(ns restaurant
    ...
    (:import (ch.qos.logback.classic Level Logger)
             (io.opentelemetry.instrumentation.logback.appender.v1_0 OpenTelemetryAppender)
             (org.eclipse.jetty.server Server)
             (org.slf4j LoggerFactory))
    ...)

(defn configure-open-telemetry-logging []
      (let [context (LoggerFactory/getILoggerFactory)
            ^Logger logger (.getLogger context Logger/ROOT_LOGGER_NAME)]
           (.detachAppender logger "console")
           (let [open-telemetry-appender (doto (OpenTelemetryAppender.)
                                               (.setContext context)
                                               (.setCaptureCodeAttributes true)
                                               (.start))]
                (doto logger
                      (.setLevel Level/INFO)
                      (.addAppender open-telemetry-appender)))))

...

(defn -main [& _args]
      (configure-open-telemetry-logging)
      ...)

(comment
  (configure-open-telemetry-logging)
  ...
  )
```

It's marginally disconcerting that logging now takes up almost exactly half of the service's code, but to be fair, the
rest is just managing the HTTP server.

## The important information

I added the OpenTelemetry Java agent to my Dockerfile and added the required arguments to attach to the JVM as the
service started.

```dockerfile
FROM clojure:temurin-21-alpine AS builder

RUN mkdir -p /artifacts
RUN wget -O /artifacts/opentelemetry-javaagent.jar https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/v2.6.0/opentelemetry-javaagent.jar

...

FROM eclipse-temurin:21-jre-alpine AS final

...

COPY --from=builder --chown=clojure:clojure /artifacts/opentelemetry-javaagent.jar /service/opentelemetry-javaagent.jar

...

CMD ["java", "-javaagent:/service/opentelemetry-javaagent.jar", "-jar", "/service/restaurant.jar"]
```

For production traces I like [Honeycomb](https://honeycomb.io) (I'm also a big fan of Honeycomb's CTO
[Charity Majors](https://charity.wtf), especially her perspectives on observability and socio-technical systems. I
highly recommend perusing her blog when you have a chance.) They've got a free plan of 20 million events per month,
which is more than enough to get started with. Having signed up I used `flyctl` to store the sensitive env vars that
required my API key

```shell
fly secrets set OTEL_EXPORTER_OTLP_HEADERS="x-honeycomb-team=<API key>"
fly secrets set OTEL_EXPORTER_OTLP_METRICS_HEADERS="x-honeycombb-team=<API key>,x-honeycomb-dataset=restaurant"
```

and configured the rest of the required environment variables in my `fly.toml` configuration

```toml
[env]
  OTEL_TRACES_EXPORTER = 'otlp'
  OTEL_METRICS_EXPORTER = 'otlp'
  OTEL_EXPORTER_OTLP_ENDPOINT = 'https://api.honeycomb.io'
  OTEL_EXPORTER_OTLP_TRACES_ENDPOINT = 'https://api.honeycomb.io/v1/traces'
  OTEL_EXPORTER_OTLP_METRICS_ENDPOINT = 'https://api.honeycomb.io/v1/metrics'
  OTEL_SERVICE_NAME = 'restaurant'
```

The service was redeployed and logs and traces were being sent to Honeycomb.

# Summary

The last six months (on and often off) has basically resulted in one line of code,
`flyctl deploy --config infra/fly.toml`. It's pretty humbling, knowing I'm still perfectly capable of barreling headlong
into a rabbit hole far enough that the diggers need to be called in.

The restaurant service is now running in production, re-deployed every time a new change is pushed to `main` and is
pushing traces and logs through OpenTelemetry to Honeycomb.

Next time we'll finish off the check-list, "Turn on all the error messages". See you then. Hopefully somewhat sooner
than this time.

**Next:** [Guardrails](./guardrails.html)

**Previous:** [Modern software engineering for a small team](./modern-software-engineering-for-a-small-team.html)
