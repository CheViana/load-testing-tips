# Practical tips for load testing web application

Before web app goes live, it is a very good idea to perform load testing: apply artificial user load to web application. I'm going to lay out a couple of tips on how to prepare, run and analyse load testing session.


## Metrics, targets and testing scenario

First of all, one needs to determine what is the amount of load application is expected to withstand, what kind of requests should be used to generate load and what would be acceptable web app error rate and response time.

In short, *determine scenario and success criteria for load testing session before starting one*.

### Scenario and load level

By scenario I mean which web app URLs load testing tool will hit.

Here's example load testing scenario:

```
    120 simulated users query web app:
    - 10% requests hit home page - GET https://musicstore.com/
    - 25% hit product list page - GET https://musicstore.com/shopnow/?p=1
    - 60% hit product detail page - GET https://musicstore.com/fancypiano123/
    - 5% submit order - POST https://musicstore.com/buy/
```

Where one can get these numbers, and which URLs should be in scenario at all? In case new web app is replacing existing one (which is frankly speaking true for a majority of web apps), one can mine the logs of the legacy application: find out which pages are hit most frequently. It's ok to include in scenario only high-hit pages. E.g. from logs collected over last month (or even year) you can see that total sum of hits to home, product list and detail pages is 98.3% of all web app hits, and other page types account for max 0.1% hits per page type. For such case, load testing scenario can include only home, product list and detail page hits. Yet it is important that hits to other page types won't bring the whole web app down. In case "other page types" are all just static pages - probably no need to include them in scenario. In case there are page types which are not hit frequently but are heavy on resource usage - might be good idea to include those in main scenario, or run separate smaller-scale load testing session for those, with another success criteria.

Now there's the question of how much load should the tool apply to web app. That is also a question to answer which one can use logs/monitoring of legacy app. It's likely the case that user load for legacy app fluctuates a lot during the course of day or week. Determine the min, avg and max RPS legacy app handled during say last month (same datetime range used in previous analysis of how load should split between page types), and swing from there. I would suggest following formula for RPS in load testing session: `max(avgRPS * 3, 2 * maxRPS)`. Simplest formula good enough for most is `2 * maxRPS`. If user traffic is anticipated to grow in future, use anticipated future `maxRPS` instead of one derived from logs.

There is another way to think about target load - in terms of simultaneous users. One can compute what is max amount of users quering the app at the same time, and use that for value of simulated users in load testing session. Most load testing tools only allow to specify amount of simulated users, not target RPS. Simulated user request should be close to real life. In case app users are live people (not other apps), it makes sense to introduce small delay between requests simulated user makes, and make sure tool reuses HTTP connection for requests from same artificial user. Some load testing tools allow to write involved scenario: user-thread queries thome page, then product list, then product list page 2, then product detail, then does checkout. It can be easier to write scenario this way, yet it is important to make sure created RPS level is representative of real life traffic. 

It is good idea to investigate both RPS level and simultaneous users level, and make sure both of those are covered with big gap by amount of artificial load you're gonna apply.

Also there is question of parametrizing artificial users requests. Users don't all view same product detail page, they view different products. Scenario should account for that - e.g. take product ID from some big pool of popular product IDs, so that requests aren't all https://musicstore.com/fancypiano123/, but https://musicstore.com/fancypiano1/ , https://musicstore.com/fancyguitar3/ , https://musicstore.com/fancydrums8/ ... Same is even true for product list pages - users view page 1, page 2, use different sort, page size, etc. Scenario should correspond to real traffic. Otherwise a handful of pages queried in load testing session will be mostly served from cache, and it won't be a representative test session. Test session should query parametrized URLs, with huge pool of param values, to keep cache usage close to real life. Good enough pool of param values could be a list of product IDs users viewed during last week.


### Server response time target

[Google recommends to keep server response time below 200msec](https://developers.google.com/speed/docs/insights/Server). Time-to-first-byte is server response time plus time it takes web response to travel over wire to the user (in case there's no edge caching involved). I don't think I have to underline importance of TTFB for web apps, which directly depends on server response time. What I want to mention though is that it is possible to measure response time at the different layers of web app "onion":
- avg response time reported by web app itself (e.g. [uWSGI avg worker request time](https://uwsgi-docs.readthedocs.io/en/latest/StatsServer.html))
- avg response time reported by load balancer in front of web app (if there's one)
- avg response time reported by gateway server in front of load balancer (by gateway I mean the endpoint which is first in chain of servers/infrastructure you have control over, first to accept user requests and route them downstream)

Particular app's "onion" can be different, but I hope you grab the point. I find it most beneficial to measure response time on the gateway layer (the outermost onion layer), the web app layer (the innermost onion layer), as well as load testing tool level. It is probably a good idea to have monitoring and logging on all other onion layers, as long as it's not significantly penalizes performance.

It makes sense to run load testing tool outside of web app environment, to emulate as close as possible how user requests are going to be made - load testing requests should flow through all the pipes real user requests will flow through. This way tool's reported response time can be thought of as TTFB approximation. 

Response time and error rate reported by load testing tool could be significantly affected by things like traffic loss en route, and such things tend to be tentative as hell, and skew results when you're trying to compare results of two sessions run on different days. This is one of reasons it's a good idea to run (big final) load testing session for significant amount of time, like a day or a week or even a month. And in case of trying to measure impact of some changes introduced in web app onion, it is a good idea to run load test at the same time on two endpoints (base and modified), or to run sessions as close in time as possible.

My advice is to look at both gateway server reported response time and load testing tool response time, making sure the later fits SLA, and using former as a main metric in comparative sessions, as gateway server reported response time deviates less, is less affected by factors which you don't have control over and factors which can change significantly over time.

I also want to mention that for response time it might make sense to use not avg but say 95 percentile value, or use both with different targets for avg and for 95 percentile. 
It might also make sense to have different target values for different web app pages - e.g. static home page to be super quick, detail view to be allowed to take a bit more time.


### Error rate target

Target error rate could be `Allowed errors <= 100% - SLA uptime target`, for example 1% (as reported by load testing tool). In case there's no [SLA](https://en.wikipedia.org/wiki/Service-level_agreement) set for product, it is probably a good idea to talk to project management to adopt one. Even as primitive SLA as "Uptime 99% (as reported by periodic external probe), avg response time (on the gateway level) <= 300msec, max time to resolve outage 4hr" will likely save a lot of headache when outage occurs.

Most things said for response time is true for other metrics you decide to use as success criteria - error rate, etc. Specific to error rate metric is the question of what to count as an error: 50x response, 40x response? Should there be different targets for different response statuses? Those are application-specific questions, and it is common that business side dictates those.

It is likely a good idea to count very long executing requests (10 seconds?) as failures, unless this is something expected for particular web app views. Yet that is antipatttern, strictly speaking, to have very long-executing requests served by regular web app workers, those should be [decoupled from web app](https://medium.com/geekculture/rest-api-best-practices-decouple-long-running-tasks-from-http-request-processing-9fab2921ace8).   

Scenario used by load testing tool usually contains definition of "success hit" which can be tweaked to suit your app needs.


### Tools

reuse connections


### Preparatory sessions

test backends first

### Environment

don't share infra used by load test with live prod apps
caching

### Results reporting
