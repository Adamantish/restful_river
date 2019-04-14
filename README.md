
# RESTful River

### Got Microservices? 
Let's PUT stressful streams in the ditch.

## The Situation

- You have some ruby microservices.
- Guaranteeing consistent state everywhere is a complete PITA.
- Your app is fairly CRUD oriented.

Back when you had a monolith at least you just had one database. That idea won't fly now. What to do?
- **HTTP REST (Post/Puts/Delete)?** Easy but you know it's a BAAD idea for so many reasons - mostly that you now have synchronous calls spoiling your nice, autonomous services.
- **Event driven?** What are we, google? The army? This is *not* why I chose Rails. I'm supposed to write millions of callbacky, [Rube Goldberg](https://en.wikipedia.org/wiki/Rube_Goldberg_machine) event handlers and STILL I'm screwed for consistency if anything ever goes wrong? Do I re-trigger all events then? That takes forever and who knows if it even works because what about the deletes? (To be fair, if there is so much state that needs sharing, perhaps some services should be merged but, hey, we are where we are.)
- **Event Sourced?** Ok, I will be google. I *can* havez the guaranteed eventual consistency if I only throw away my SQL databases, replace with one event store, rewrite my app with a [DDD](https://airbrake.io/blog/software-design/domain-driven-design) philosophy, carefully preserve every outdated piece of business logic I ever wrote and don't write any event handlers which touch the outside world. 

Why so pain?!!!
 
## A Solution

**Do the bad REST thing but make it good!**

Take some ideas from [CQRS](https://martinfowler.com/bliki/CQRS.html), make it asynchronous, performant, batch optimized with eventual consistency easily guaranteed.

## I wants tha codez

***Service A***
```ruby
class MyProfilesRiver < RestfulRiver::Source
  dam_burst_every 3.seconds # (or 0.seconds or 1.day)
end

MyProfilesRiver.puts(profile_payload)
```

***Service B (and C, D, etc..)***

```ruby
class MyProfilesRiver < RestfulRiver::Mouth
  mouth_breathers [MyProfileReceiver, SomeOtherNoseyClass] # Or you can say `listeners` if you have no joy.
end

class MyProfileReceiver
  def puts(profile_payloads)
	# Do your clever, efficient stuff with this batch of payloads or just n+1 if it's no biggy
	# but remember to store the payload.river_timestamp if you want the really good stuff I'll tell you next.
  end
end
```
Nice.

Ok, it's weeks later, there was some unhandled edge case, errors were thrown, people cried, things are **not consistent**. Let me fix this.

Set it up... Go back into `MyProfileReceiver` and add
```ruby
 def entity_timestamps
   SomeProfilesDatastore.select(:id, :river_timestamp).to_hashes
 end
```
Press the makeeverythinggoodagain button...
```ruby
MyProfilesRiver.reconcile!(MyProfileReceiver)
```
All is good! The latest missed updates got identified and re-sent to my `puts` method.

What if the problem was on the other end? Messages didn't get "rivered" in the first place? There's a reconciliation mechanism for that side too which you can read about in the docs.

Also "woah, that ran so fast I could even just put it in a scheduled job and become a very relaxed person!" 

## This looks much simpler than how you're *meant* to do event streaming. Could this really work for me?

Yes. It makes a lot of nice, broad assumptions about your data model which are right for *most* cases.  Funnily enough these are the same simplifying assumptions that helped HTTP REST to kill SOAP: Namely...

-  Your app is fairly CRUD based.
-  You don't sweat the extra I/O of sending the whole payload instead of complicated partial updates. (that's why it's a calm, fat river, not a choppy stream)

In terms of sheer efficiency, the heavy lifting is largely in the PostgreSQL engine. Whilst the I/O of a river is usually broader than a stream it's not in such a dangerous place. In fact, for services that don't need updating so often it becomes trivial to rate limit high I/O updates of the same entities.

## What is this sorcery?

You mean "how does it work"? You know, you could have just asked that in a normal way. 

Pretty simple really. It breaks a "rule" and asks to be pointed to a PostgreSQL server (but, hey, I bet you have one spun up already) and quietly makes a shared database. 

In another sense it doesn't break the rule. If this database goes down it's the same as a message broker going down. i.e. services can keep functioning even if they go a bit out of date.

In contrast to a full on message broker,  the latest version of every entity (indexed by id) is retained so it's easy to batch up requests when needing to scale and full
reconciliation becomes no problem.

## If so simple, why do I need a gem?

So much good stuff under the hood going on to avoid race conditions with single-threadedness whilst parallelising other workloads.


## Is this just a Rails thing?

Nope. Any ruby program with access to a shared PostgreSQL server works fine.

## Wait, there are River objects sharing the same name in different services but one is a `Source`, the other is a `Mouth`.

Yup. This a simple way of making clear that there's only room for a writer(source) or reader(mouth) in the same service, not both. I'm touched you want to use it everwhere but it's probably better to have clear one-way data flows. Anyway, you can namespace under different modules just so long as the class names match.

## Why are you dissing Event Sourcing?

Actually I think it's a beautiful concept, DDD is a great way to do collaboration with non-technical people on many projects and I'd love an excuse to use it for more compellingly complex domains... But for most projects and and skillsets it's just overkill and turns easy things hard and risky without enough reward.

## Do you have a surprisingly apt quote for me?

> "What makes a river so restful...is that it doesn't have any
> doubt - it is sure to get where it is going, and it doesn't want to go
> anywhere else."

 *Hal Boyle*

