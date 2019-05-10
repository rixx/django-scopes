django-scopes
=============

Motivation
----------

Many of us use Django to build multi-tenant applications where every user only ever
gets access to a small, separated fraction of the data in our application, while
at the same time having *some* global functionality that makes separate databases per
client infeasible. While Django does a great job protecting us from building SQL
injection vulnerabilities and similar errors, Django can't protect us from logic
errors and one of the most dangerous types of security issues for multi-tenant
applications is that we leak data across tenants.

It's so easy to forget that one ``.filter`` call and it's hard to catch these errors
in both manual and automated testing, since you usually do not have a lot of clients
in your development setup. Leaving [radical, database-sependent ideas](https://github.com/bernardopires/django-tenant-schemas)
aside, there aren't many approaches available in the ecosystem to prevent these mistakes
from happening aside from rigorous code review.

We'd like to propose this module as a flexible line of defense. It is meant to have
little impact on your day-to-day work, but act as a safeguard in case you build a
faulty query.

Usage
-----

Let's assume we have a multi-tenant blog application consisting of the three models ``Site``,
``Post``, and ``Comment``:

	from django.db import models

	class Site(models.Model):
		name = models.CharField(…)

	class Post(models.Model):
		site = models.ForeignKey(Site, …)
		title = models.CharField(…)

	class Comment(models.Model):
		post = models.ForeignKey(Post, …)
		text = models.CharField(…)

In this case, our application will probably be full of statements like
``Post.objects.filter(site=current_site)``, ``Comment.objects.filter(post__site=current_site)``,
or more complex when more flexible permission handling is involved. With django-scopes, we
engourage you to still write these queries with your custom permission-based filters, but
we add a custom model manager that has knowledge about posts and comments being part of a
tenant scope:

	from django_scopes import ScopedManager

	class Post(models.Model):
		site = models.ForeignKey(Site, …)
		title = models.CharField(…)

		objects = ScopedManager(site='site')

	class Comment(models.Model):
		post = models.ForeignKey(Post, …)
		text = models.CharField(…)

		objects = ScopedManager(site='post__site')

The keyword argument ``site`` defines the name of our **scope dimension**, while the string
``'site'`` or ``'post__site'`` tells us how we can look up the value for this scope dimension
in ORM queries.

You could have multi-dimensional scopes by passing multiple keyword arguments to
``ScopedManager``, e.g. ``ScopedManager(site='post__site', user='author')`` if that is
relevant to your usecase.

Now, with this custom manager, all queries are banned at first:

	>>> Comment.objects.all()
	ScopeError: A scope on dimension "site" needs to be active for this query.

The only thing that will work is ``Comment.objects.none()``, which is useful e.g. for Django
generic view definitions.

You can now use our context manager to specifically allow queries to a specific blogging site,
e.g.:

	from django_scopes import scope

	with scope(site=current_site):
		Comment.objects.all()

This will *automatically* add a ``.filter(post__site=current_site)`` to all of your queries.
Again, we recommend that you *still* write them explicitly, but it is nice to know to have a
safeguard.

Of course, you can still explicitly enter a non-scoped context to access all the objects in your
system:

	with scope(site=None):
		Comment.objects.all()

This also works correclt nested within a previously defined scope.

Sounds cumbersome to put those ``with`` statements everywhere? Maybe not at all: You probably
already have a middleware that determines the site (or tenant, in general) for every request
based on URL or logged in user, and you can easily use it there to just automatically wrap
it around all your tenant-specific views.