# Random notes on things

### Healthchecks 

1. Aggregate other services into your application healthcheck
The first time I implemented a healthcheck for a user-facing application, I decided to include tests of external components like data stores and message queues. Intuitively that seemed like the correct thing to do because if the application can't read data, it's not healthy, right?

The problem with that approach is how it causes things like intermittent connectivity issues to fan out and become full blown outages. Let's say your application can't connect to its primary data store. That's a problem of course, but it can probably still serve a partly functional UI and some data might still be available from caches or alternative stores.

If an application healthcheck fails and there are no healthy nodes for your load balancer to send traffic to, it will return a blanket 502 response instead. That's a worse experience for your users than a part-working application, which is missing some data but otherwise appears okay. They'll probably leave with a worse opinion of your product if all they see is a generic fail screen.

2. Set a short timeout on healthcheck requests
If your healthcheck is simple and doesn't aggregate any external services, you expect it to be fast. So there's no need for a long timeout, right? 2 seconds should be plenty, or so I thought.

Funny things can happen to applications under load and there will be times when your assumptions about how long things take are broken. It's better to err on the side of caution, so that your instances aren't prematurely marked unhealthy. An application that works slowly is preferable to one that doesn't work at all.

3. Set a long timeout on healthcheck requests
After you've experienced problems with a short timeout, the temptation is to move to the other end of the spectrum and make it very long. That can cause problems too though, if your chosen timeout is a bounding factor on how frequently healthcheck requests can be made.

At least in GCP, which is the environment I'm most familiar with, it's not possible to set healthcheck frequency to a shorter period than this timeout. If your service is down and the healthcheck has, say, a 30-second timeout configured, that means there will be at least a 30-second wait between healthcheck attempts. That can be a problem when you want your service to come back as quickly as possible after an outage.

So there's a goldilocks zone for healthcheck timeouts, and you should try to find the sweet spot for your own system. Mine is currently 8 seconds.

4. Leave a long delay before starting healthchecks on new instances
In GCP, the default delay before healthchecks start running against new instances is 5 minutes. That means a 5 minute wait before they handle any traffic, which might be fine in a deployment scenario but is probably not fine if they're being scaled in to handle excess load or replace unhealthy instances.

Measure how long it takes your application to start and then adjust that setting accordingly. The sooner your new instances start receiving traffic, the better.

5. Set a low threshold on consecutive failures before turning unhealthy
It can be tempting to mark nodes as unhealthy after just one failure from the healthcheck endpoint, but again you should err on the side of caution here and wait until there have been multiple consecutive failures. There's no need to swap out instances for one-off problems that are resolved quickly.

6. Set a high threshold on consecutive successes before turning healthy
Where you don't want to err on the side of caution, is when unhealthy nodes become healthy again. There's no benefit to waiting for consecutive successes, that can prolong an outage. As soon as their healthchecks succeed, you want to get those instances back into the frontline.

Conclusion
Essentially, all these gotchas boil down to two principles: don't mark instances unhealthy prematurely and mark new instances healthy as soon as possible. Those seem like obvious truisms when stated in isolation, but it's easy to miss their significance when they're buried under concrete infrastructure settings.
