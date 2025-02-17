---
img: /img/profefe.png
layout: post
title: "Continuous profiling in Go with Profefe"
date: 2020-01-03 09:08:27
categories: [post]
tags: [golang, pprof, profefe]
summary: "Taking a snapshot at the right time is nearly impossible. A very easy
way to fix this issue is to have a continuous profiling infrastructure that
gives you enough confidence of having a profile at the time you need it."
changefreq: daily
---

There are a lot of articles about profiling in Go. Julia Evans for examples
wrote ["Profiling Go programs with
pprof"](https://jvns.ca/blog/2017/09/24/profiling-go-with-pprof/) and I rely on
it when I do not remember how to properly use pprof.

Rakyll wrote ["Custom pprof profiles"](https://rakyll.org/custom-profiles/).

`pprof` is a powerful tool provided by Go that helps any developer to figure out
what is going in the Go runtime. When you see a spike in memory in your running
container the next question is who is using all that memory. Profiles tell you
the answer.

But they need to be grabbed at the right time. The unique way to have a profile when
you need it is by taking them continuously. Based on your application you should
be able to specify how often you have to gather a profile.

This requires a proper infrastructure that we can call "Continuous profiles
infrastructure". It is made of collectors, repositories and you need an API to
store, retrieve and query those profiles.

When we had to set it up at InfluxData we started to craft our own one until I
saw [`profefe`](https://github.com/profefe/profefe) on GitHub. What I love about
the project is its clear scope. It is a repository for profiles. You can push
them in Profefe and it provides an API to get them out, it servers the profiles in a
way that make them easy to visualize directly with `go tool pprof`, you can even
merge them together and so on. It also have a clear interface that helps you to
implement your own storage.

The project
[README.md](https://github.com/profefe/profefe/blob/master/README.md) well
explains how it works but I am going to summarize the most important actions in
this article.

## Getting Started

There is a docker image that you can run with the command:

```
docker run -d -p 10100:10100 profefe/profefe
```

You can push a profile in profefe:

```
$ curl -X POST \
    "http://localhost:10100/api/0/profiles?service=apid&type=cpu" \
    --data-binary @pprof.profefe.samples.cpu.001.pb.gz

{"code":200,"body":{"id":"bo51acqs8snb9srq3p10","type":"cpu","service":"apid","created_at":"2019-12-30T15:18:11.361815452Z"}}
```

You can retrieve it directly via its ID:

```
$ go tool pprof http://localhost:10100/api/0/profiles/bo51acqs8snb9srq3p10

Fetching profile over HTTP from http://localhost:10100/api/0/profiles/bo51acqs8snb9srq3p10
Saved profile in /home/gianarb/pprof/pprof.profefe.samples.cpu.002.pb.gz
File: profefe
Type: cpu
Time: Dec 23, 2019 at 4:06pm (CET)
Duration: 30s, Total samples = 0
```

There is a lot more you can do, when pushing a profile you can set key value
pairs called `labels` and they can be used to query a portion of the profiles.

You can use `env=prod|test|dev` or `region=us|eu` and so on.

Retrieving a profile only via ID it's not the unique way to visualize it.
Profefe merges together profiles from the same type in a specific time range:

```
GET /api/0/profiles/merge?service=<service>&type=<type>&from=<created_from>&to=<created_to>&labels=<key=value,key=value>
```

It returns the raw compressed binary, it is compatible with `go tool pprof` as
well as the single profile by id.

## Conclusion

I didn't develop profefe, [Vladimir (@narqo)](https://github.com/narqo) is the
maintainer, I like it and how it is coded. I think it solves a very common
issue. He wrote a detailed post about his project
["Continuous Profiling and Go"](https://medium.com/@tvii/continuous-profiling-and-go-6c0ab4d2504b)

> Wouldn’t it be great if we could go back in time to the point when the issue
> happened in production and collect all runtime profiles. Unfortunately, to my
> knowledge, we can’t do that.

One of my colleague Chris Goller wrote a self contained AWS S3 implementation
that is now submitted as PR. We are running it since a couple of weeks now. It
is hard to onboard developers in a new tool, even more during Christmas but the
API layers makes it very comfortable and friendly to use. Next article will be
about what we did to get it running in Kubernetes continuously profiling our
containers.
