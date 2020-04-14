---
title: Adding a New Thanos Component that Embeds Cortex Query Frontend
type: proposal
menu: proposals
status: in-review
owner: bwplotka
---

### Related Tickets

* Response caching: https://github.com/thanos-io/thanos/issues/1651
* Moving query frontend to separate repo: https://github.com/cortexproject/cortex/issues/1672

## Summary

This proposal describes addition of a new Thanos command (component) into `cmd/thanos` called `frontend`.
This component will literally import a certain version of Cortex [frontend package](https://github.com/cortexproject/cortex/tree/4410bed704e7d8f63418b02b328ddb93d99fad0b/pkg/querier/frontend).

We will go through rationales, and potential alternatives.

## Motivation

[Cortex Frontend](https://www.youtube.com/watch?v=eyBbImSDOrI&t=2s) was introduced by Tom in August 2019. It was designed
to be deployed in front of Prometheus Query API in order to ensure:

* Query split by time.
* Query step alignment.
* Query retry logic
* Query limit logic
* Query response cache in memory, Memcached or Redis.

Since the nature of Cortex backend is really similar to Thanos, with exactly the same PromQL API, and long term capabilities, the caching
work done for Cortex fits to Thanos. Given also our good collaboration in the past, it feels natural to reuse Cortex's code.
We even started discussion to move it to separate repo, but there was no motivation towards this, for a good reason.
At the end we were advertising to use cortex query frontend on production on top of Thanos and this works considerably well, with some
problems on edge cases and for downsampled data as mentioned [here]()

However, we realized recently that asking users to install suddenly Cortex component on top of Thanos system is extremely confusing:

* Cortex has totally different way of configuring services. It requires to decide what module you have in single YAML file. Thanos in opposite
have flags and different subcommand for each component.
* Cortex has bit different way of configuring memcached, which is inconsistent with what we have in Thanos Stote Gateway.
* There are many Cortex specific configuration items which can confuse Thanos user and increase complexity overall (e.g. )
* We have many ideas how to improve Cortex Query Frontend on top of Thanos, but adding Thanos specific configuration options will increase
complexity on Cortex side as well.
* Cortex has no good example or tutorial on how to use frontend either. We have only [Observatorium example](https://github.com/observatorium/configuration/blob/master/environments/openshift/manifests/observatorium-template.yaml#L515).

All of this were causing confusion and questions like [this](https://cloud-native.slack.com/archives/CK5RSSC10/p1586504362400300?thread_ts=1586492170.387900&cid=CK5RSSC10).

At the end we decided with Thanos and Cortex maintainers that the ultimately it would be awesome to create a new Thanos subcommand called `frontend`.

## Use Cases

* User can cache responses for query range
* User can use the same configuration patterns as rest of Thanos components.
* Thanos Developer does not need to write and maintain custom response cache logic.

## Goals of this design

* Enable response caching that will easy to use for Thanos users.
* Keep it extensible and scalable for future improvements like advanced query planning, queuing, rate limiting etc.
* Reuse as much as possible between projects, contribute.

## No Goals

* Create Thanos specific response caching from scratch.

## Proposal

The idea is to create `thanos frontend` component that allows specifying following options:

* `--query-range.split-interval`, `time.Duration`
* `--query-range.max-retries-per-request`, `int`, default = `5`
* `--query-range.disable-step-align`, `bool`
* `--query-range.response-cache-ttl` `time.Duration` default = `1m`
* `--query-range.response-cache-config(-file)` `pathorcontent` + [CacheConfig](https://github.com/thanos-io/thanos/blob/55cb8ca38b3539381dc6a781e637df15c694e50a/pkg/store/cache/factory.go#L32)

We plan to have in-mem, fifo and memcached support for now. Cache config will be exactly the same as the one used for Store Gateway.

### Open Questions

* Is `thanos frontend` a valid name? I feel it's short and verbose enough.

### Alternatives

#### Don't add anything, document Cortex query frontend and add examples of usage

Unfortunately we tried this path already without success. Reasons were mentioned in [Motivation](202004_embedd_cortex_frontend.md#Motivation)

#### Add response caching to Querier itself, in the same binary.

This will definitely simplify deployment if Querier would allow caching directly. However, this way is not really scalable.

Furthermore, eventually frontend will be responsible for more than just caching. It is meant to do query planning like splitting or even
advanced query paralization (query sharding). This might mean future improvements in terms of query scheduling, queuing and retrying.
This means that at some point we would need an ability to scale query part and caching/query planner totally separately.

Last but not least splitting queries allows to perform request in parallel. Only if used in single binary we can achieve load balancing of those requests.

NOTE: We can still consider just simple response caching inside Querier if user will request so.

#### Write response caching from scratch.

I think this does not need to be explained. Response caching has proven to be not trivial. It's really amazing that we
have opportunity to work towards something that works with experts in the field like @tomwilkie and others from Loki and Cortex Team.

Overall, [Reusing is caring](https://www.bwplotka.dev/2020/how-to-became-oss-maintainer/#5-want-more-help-give-back-help-others).

## Work Plan

1. Refactor [IndexCacheConfig](https://github.com/thanos-io/thanos/blob/55cb8ca38b3539381dc6a781e637df15c694e50a/pkg/store/cache/factory.go#L32) to generic cache config so we can reuse.
1. Add necessary changes to Cortex frontend
    * Metric generalization (they are globals now).
1. Add `thanos frontend` subcommand.
1. Add proper e2e test using cache.
1. Document new subcommand
1. Add to [kube-thanos](https://github.com/thanos-io/kube-thanos0)

## Future Work

Improvements to Cortex query frontend, so Thanos `frontend` as described [here](https://github.com/thanos-io/thanos/issues/1651)