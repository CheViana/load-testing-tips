# Practical tips for load testing web application

Before web app goes live, it is a very good idea to perform load testing: apply artificial user load to web application. I'm going to lay out a couple of tips on how to prepare, run and analyse load testing session. Some points are quite obvious, yet I hope it will be useful to have all these tips (covered in sweat and tears) mentioned in one writing.


Roughly speaking, to prepare for long term high load session:

- determine scenario and success criteria for load testing session
- instrument monitoring of all things under load
- run preparatory sessions before unleashing high load

The load testing process usually goes something like this:

load it up - something brakes - stop the load - tune/fix/adjust - load it up again - ...

Repeat until complete long term high load session results meet success criteria.


## Metrics, targets and testing scenario

First of all, one needs to determine what is the amount of load application is expected to withstand, what kind of requests should be used to generate load and what would be acceptable web app error rate and response time.


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

Where one can get these numbers, and which URLs should be in scenario at all? In case new web app is replacing existing one (which is frankly speaking true for a majority of web apps), one can mine the logs of the legacy application: find out which pages are hit most frequently. It's ok to include in scenario only high-hit pages. E.g. from logs collected over last month (or even year) you can see that total sum of hits to home, product list and detail pages is 98.3% of all web app hits, and other page types account for max 0.1% hits per page type. For such case, load testing scenario can include only home, product list and detail page hits. Yet it is important that hits to other page types won't bring the whole web app down. In case "other page types" are all just static pages - probably no need to include them in scenario. In case there are page types which are not hit frequently but are heavy on resource usage - might be good idea to include those in main scenario, or run separate smaller-scale load testing session for those, possibly with adjusted success criteria.

Now there's the question of how much load should the tool apply to web app. That is also a question to answer which one can use logs/monitoring of legacy app. It's likely the case that user load for legacy app fluctuates a lot during the course of day or week. Determine the min, avg and max RPS legacy app handled during say last month (same datetime range used in previous analysis of how load should split between page types), and swing from there. I would suggest following formula for RPS in load testing session: `max(avgRPS * 4, 3 * maxRPS)`. Simplest formula good enough for most is `3 * maxRPS`. If user traffic is anticipated to grow in future, use anticipated future `maxRPS` instead of one derived from logs.

There is another way to think about target load - in terms of simultaneous users. One can compute what is max amount of users quering the app at the same time, and use that for value of simulated users in load testing session. Most load testing tools only allow to specify amount of simulated users, not target RPS. Simulated user request should be close to real life. In case app users are live people (not other apps), it makes sense to introduce small delay between requests simulated user makes (e.g. using [JMeter's Gaussian random timer](https://www.blazemeter.com/blog/comprehensive-guide-using-jmeter-timers)).

It is also worth to take a look at HTTP connection settings used by load testing tool to make sure it reuses HTTP connection for requests from same artificial user for some number of requests, set HTTP version, adjust other useful options like socket timeout (configure JMeter's `LoopController.loops` for thread group, set `HttpClient` to [HttpClient4 class](https://jmeter.apache.org/usermanual/properties_reference.html#httpclient4)).

Some load testing tools allow to write involved scenario: user-thread queries home page, then product list, then product list page 2, then product detail, then does checkout. It can be easier to write scenario this way, instead of defining how much RPS should go to which page, yet it is important to make sure created load level is representative of real traffic.

It is good idea to investigate both RPS level and simultaneous users level, and make sure both of those are covered with big gap by amount of artificial load that is going to be applied.

Also there is question of parametrizing artificial users requests. Users don't all view same product detail page, they view different products. Scenario should account for that - e.g. take product ID from some big pool of popular product IDs, so that requests aren't all https://musicstore.com/fancypiano123/, but https://musicstore.com/fancypiano1/ , https://musicstore.com/fancyguitar3/ , https://musicstore.com/fancydrums8/ ... Same is even true for product list pages - users view page 1, page 2, use different sort, page size, etc. Scenario should correspond to real traffic. Otherwise a handful of pages queried in load testing session will be mostly served from cache, and it won't be a representative test session. Test session should query parametrized URLs, with huge pool of param values, to keep cache usage close to real life. Good enough pool of param values could be a list of product IDs users viewed during last week.

What if app is completely new, there's no logs to mine? In such case one might brainstorm together with business folks scenario which best predicts app usage, and possibly use higher value in `N * maxRPS` formulae.

Scenario can also have a "ramp up" time configured - it won't unleash all simulated load at once, instead load will grow bit by bit for configured time. This way might better reflect real life, for example when canary rollout strategy is used. [JMeter](https://jmeter.apache.org/usermanual/test_plan.html#thread_group) and other tools allow to configure such option.


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


## Tools

There's plenty load testing tools available - see for example this [post](https://k6.io/blog/comparing-best-open-source-load-testing-tools/).
Popular long-existing tool used by lots of people is JMeter. It's good fit for those who need involved scenario and prefer GUI interactions over writing code. It's possible to run it with master+worker nodes setup in cloud (see this [post](https://github.com/kubernauts/jmeter-kubernetes)), or use pay-for online-based platforms which run JMeter for you.
Another tool which gains popularity quickly particularly for cloud-based setup is k6. It's good fit if you don't mind JavaScript/JSON and need something cloud-native. GitLab has [example](https://docs.gitlab.com/ee/user/project/merge_requests/load_performance_testing.html) on how to run k6 in job, which sounds like good idea for quick load test before rolling out new vesion of web app to prod, but no so great idea to use k6 in GitLab job for week-long high-load session before web app goes live for the first time. One can run k6 in separate from web app cloud project instead, or from some other hardware.
Simple tool which is also very powerful is [wrk2](https://github.com/giltene/wrk2). It allows to specify target RPS. Good fit if you have one static URL to test, or for use in simpler preparatory testing.
In case you're good with Python and don't need high RPS level, locust tool might be good fit, as it is rather easy to write complex scenario for.


## Preparatory, supplemental and resilience sessions

### Preparatory sessions

It makes sense to first load test backends and DBs web app uses - calculate what load is expected for DB, and apply that directly to backend. Load tool probably needs to be inside your infrastructure for such tests. Make sure backends and DBs and other things web app uses are handling load alright (satisfy SLA), then switch to load testing web app itself. It might be cheaper this way - load tests are expensive in terms of burned (cloud) resources.

It's also a good idea to run smaller load session for web app, before starting big load: same scenario as for high load with reduced load and shorter duration. For example, load test one web app pod, make sure it works OK for CPU load level up to scaling threshold value.

### Supplemental sessions

Among other kinds of sessions highly adviced to perform are "rectangular load" session and "spike" session.

"Rectangular load" is a session in which simulated user load has a form of rectangle: 20 RPS for 15 min, 10 RPS for 15 min, 20 RPS for 15 min, etc. Let's say initially web app stack has 2 pods in stack, when hit by 20 RPS it scales up to 4 pods, when load decreases to 10 RPS it scales down to 2 pods, etc. It could be abrupt switch from 10 to 20 RPS or it can be gradual change. For abrupt one can possibly expect a bit of errors during the uphill switch from 10 to 20 RPS, and those errors shouldn't exceed SLA level. Such session allows to look at how good web app scaling is tuned: how quickly web app stack can scale up, whether requests are dropped when app downscales. JMeter can be configured to produce "rectangular" load using [programmable timer](https://www.blazemeter.com/blog/comprehensive-guide-using-jmeter-timers):

```
    <JSR223Timer guiclass="TestBeanGUI" testclass="JSR223Timer" testname="JSR223 Timer" enabled="true">
          <stringProp name="cacheKey">false</stringProp>
          <stringProp name="filename"></stringProp>
          <stringProp name="parameters"></stringProp>
          <stringProp name="script">import java.util.Date;
Date date = new Date(); 
Long minutes = date.getMinutes();
Long minLastDigit = minutes.mod(20);

if (minLastDigit &lt; 10 ) {
	log.info(&quot;Even minute: adding timer&quot;);
	return 1000;
} else {
	return 100;
}</stringProp>
          <stringProp name="scriptLanguage">groovy</stringProp>
    </JSR223Timer>
    <hashTree/>
```

"Spike" session is a simulated short-time spike in load level. It is expected to encounter elevated error level during the spike. Of most interest is how web app stack recovers: it should be able to work normally (satisfy SLA) after the spike is gone, all on it's own, without developer's help.

### Resilience sessions

Resilience testing is a big part of preparation to go live. Resilience session is for example simulated backend outage during high-level load test session. In case backend is optional for main web app functionality, one would expect no impact (low impact) on load test session error rate when such backend goes down. It helps to tune backend timeouts and retries better, not only for being able to perform well when all is good, but also for worst cases. In case backend is required for web app to produce it's main content, it is of course expected that error rate will increase and response time decrease. Of most interest is to see how well app recovers after backend outage ends, and how well other web app pages that don't use the backend are working when that backend is down. It could be OK from the business perspective to be completely down when particular dependency is down, yet it is still adviced to emulate as much outages as you can imagine, to check how expectations are working out, and that alerting bells are ringing when they should.

Outages are best tested over main high load scenario, doesn't have to be long-running session though.

In addition to backend outages, don't forget about other possible kinds of outages, e.g. cache. It's a good idea to [tune caching policies](https://redis.io/topics/lru-cache), to set how cache should behave when it's memory usage reaches maximum configured memory, and to test how app behaves when cache is close to saturation. And test out monitoring for cache saturation if there's one.


## Load testing environment

Rule of thumb is to not share any infrastructure, backends, caches, anything used by application under load test with live production-serving apps. Simple as it sounds, it might be not easy (and not cheap) to achieve. It could be an option to run load tests during off-peak hours, if you're feeling lucky. In case you're not testing high-load apps, say it's 1 RPS load testing session, it might be ok to share infrastructure, yet it still posseses risks as new things are being loaded, and unexpected things can happen. In case load session is run in production environment, highly adviced is to keep it under close observation and terminate load quickly if something wrong.

Instrumented monitoring for web app itself, webserver, backends, caches, infrastructure is extremely helpful for diagnosing issues that surface during load testing. Highly advice to monitor all things under load. 


## Results reporting

One real nice thing about modern load testing tools is that most of them can be configured to report load testing results in real time, e.g. it is possible to make [JMeter emit current RPS, error rate, response time to a db which will power real-time dashboard](https://www.influxdata.com/influxdb-templates/apache-jmeter/). Online-based platforms which run load testing tool for you usually provide such monitoring too. It is especially helpful for long-duration test sessions.

Last piece of advice - don't get load testing tool's logs to fill up all free space, and hault itself. Provision enough log space, or turn it off in favor of real time monitoring.
