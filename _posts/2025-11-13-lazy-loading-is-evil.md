---
title: Lazy Loading is Evil
description: How a common ORM anti-pattern affects your backend
date: 2025-11-13
tags: [Coupling]
---

I had assumed everyone already agreed that lazy loading from the database was not a great idea, especially those of us dealing with ORMs and domain models, but a recent conversation reminded me that's far from true, so here we are again.

## What is Lazy Loading?

Lazy loading, in the backend context, is a pattern typically used when dealing with ORMs. Instead of loading all related data from the database upfront, you delay fetching certain parts until they're actually needed.

Here's an example:

```
class Container {
  private String name;
  
	@Lazy
	private List<Items> items;

	public List<Items> items() {
		return items;
	}
}
```

When you retrieve a `Container` entity from the database, the `name` is being fetched upfront, but the inner `items` are not. The query to fetch them will only execute once you call `container.items()`.
This means we don't bring them from the database until we need them, so the items are loaded on demand, not upfront.

Now, why would you want to load the items in lazy mode?
You might think, "well, sometimes I don't really need the items. I only need the container with its name, so why execute a heavy query for something I won't need?". You expect to gain performance by loading less data, which sounds great on paper, but that's where the trouble begins

## What's the Problem?

### Lazy Loading Couples Use Cases

The fact that you sometimes need a property and sometimes don't is a red flag. It means you're using **the same model for different use cases**.

This couples those use cases together, making your code harder to change, test, and evolve. You lose clarity about what each part of the system is supposed to do, and it's also riskier, because a change meant for one scenario can easily have unexpected effects on another.

When everything is built around a single, multipurpose model, you are more likely to end up in a mess.

### Lazy Loading Lies!

When you call `items()`, you might assume those items are already there. But under the hood, a potentially heavy database query is triggered to fetch those items.

This hidden behavior makes it dangerously easy to fall into the `N+1` query problem, causing a performance meltdown in production.
Even worse, you now have to know beforehand how calling `items()` works internally to avoid these issues.

This breaks encapsulation, because you're forced to look inside and know how something is loaded just to prevent failure, something that should be an implementation detail. That implementation detail leaks into the rest of your codebase, because suddenly calling a simple method requires internal knowledge to avoid disaster.

Your application becomes more fragile. It's easier to make a wrong choice and break the system in unexpected ways. For example, if someone were to naively remove lazy loading, might cause cascading failures. To make such changes, you'd have to hunt down every place where those items are accessed to assess the impact. That's risky, time-consuming, and unnecessary work.

## Where the Problem Comes From?

I think a big part of this is, as usual, frameworks. They make it easy to do questionable stuff, and even promote them as best practices.

Frameworks often provide "fixes" for problems that shouldn't exist in the first place. Developers struggle to analyze trade-offs, and as result they build a fragile system, where even a small change can bring production down (true story unfortunately).

## A Better Approach - Make Explicit the Implicit!

The need for lazy loading is usually a **symptom of a deeper design issue**.

If you sometimes need to load a property and other times don't, you'll be better off making those scenarios explicit. Separate your models, and use different models for different purposes. Move things that change for different reasons and at different times apart.

You can create different models that contain only the properties you need.
For example in a CQRS context, you typically separate your write models (which enforce invariants and consistency) from your read models (which are optimized for queries and reports). You do that because **they serve different purposes**.

Imagine a restaurant system:
- To book a table (write), you need a model that ensures domain rules and consistency.
- To generate a report of how many tables were booked last month (read), you need a model optimized for reading, and not necessarily enforcing rules.

When you make this distinction explicit, lazy loading becomes unnecessary. Models that need data will load it upfront, while models that don't need it simply won't even know that data exists.

## Final Thoughts

Next time you feel tempted to use lazy loading, stop and ask yourself, are you solving a real problem or just hiding a design flaw?

Lazy loading is a band-aid for coupling. Make the implicit explicit, and your system (and your future self) will thank you.
