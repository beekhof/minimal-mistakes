---
layout: post
date: 2015-10-07 16:51
title: Minimum Viable Cluster
published: true
excerpt: The is a spectrum for availability solutions, where you fit depends on what assumptions you can make about your application stack.
category:
tags: 
- cluster
- concepts
- openstack
---

In the past there was a clear distinction between _high performance_
(HP) clustering and _high availability_ (HA) clustering, however the
lines have been bluring for some time.  People have scaled HA clusters
upwards and HP inspired clusters have been used to provide
availability through redundancy.

The trend in providing availablity of late has been towards the HP
model - pools of anonymous and stateless workers that can be replaced
at will.  A really attractive idea but in order to pull it off they
have to make assumptions that may or may not be compatible with some
peoples' workloads.

Assumptions are neither wrong nor bad, you just need to make sure they
are compatible with your environment.

So when looking for an availablity solution, keep Occam's razor in
mind but don't be a slave to it.  Look for the simplest architecture
*but* then work upwards until you find one that meets the needs of
your actual (not ideal) application or stack.

### Starting Simple

_Application HA_ is the simplest kind of cluster you can deploy because
the cluster and the application are the same thing.  It takes care of
talking to its peers, checking to see if they're still online, deciding
if it should remain operational (because too many peers were lost) and
synchronising any data between itself and peers.

This gives you basic fault tolerance, when a node fails there are
other copies with sufficient state to take up the workload.

Galera and RabbitMQ (with replicated queues) are two popular examples
in this category.

However when I said _Application HA_ was the simplest, thats only from
an admin's point of view, because the application is doing everything.

Some issues the creators of these kinds of applications think about
ahead of time:

- Can I assume a node that I can't see is offline?
- What to do when some of the nodes cannot see other ones? (quorum)
- What to do when half the nodes cannot see the other half? (split-brain)
- Does it matter if the application is still active on nodes we cannot see? (data integrity)
- Is there state that needs to synchronised? (replication)
- If so, how to do so reliably and in the presence of past and future failures?  (reconciliation)

So if you're looking to create a custom application with similar
properties, make sure you can fund the development team you will need
to make it happen.

And remember that the reality of those simplifying assumptions will
only be apparent after everything else has already hit the fan.

But lets assume the best-case here... if all you need is one of these
existing applications, great! Install, configure, done. Right?

### Maybe. It might depend on your hardware budget.

Unfortunately (or perhaps not) most companies aren't Google, Twitter
or Bookface.  Most companies do not have thousands of nodes in their
cluster, in fact getting many of them to have more than two can be a
struggle.

In such environments the overhead of having 1?, 2?, 10?!? spare nodes
(just in case of a failure - which will surely never happen) starts to
represent a significant portion of their balance sheet.

As the number of spare nodes goes down, so does the number of failures
that the system can absorb.  It is irrelevant if a failure leaves two
(or twenty) functional nodes if the load generated by the clients
exceeds the application's ability to keep up.

An overloaded system leads to operation timeouts which generates even
more load and more timeouts.  The surviving nodes aren't really
functional at that point either.

If the services lived in a widget of some kind (perhaps Docker
containers or KVM images), we could have a higher level entity that
would make new copies for us.  Problem solved right?

### Maybe.  Is your application truely stateless?

Some are, Memcache is one that comes to mind because its a cache,
neither creating nor consuming anything. However even web servers seem
to want session state these days, so chances are your application
isn't stateless either.

Stateless is hard.

Where do new instances recover their state from?  Who are its peers? A
static list isn't going to be possible if the widgets are anonymous
cattle. Do you need a discovery protocol in your application?

There may also be a penalty for bringing up a brand new instance.  For
example, the sync time for a new Galera instance is a function of the
dataset size and network bandwidth.  That can easily run into the
tens-of-minutes range.

So there is an incentive to either stop modelling everything as cattle
or to keep the state somewhere else.

Ok, so lets put all the state in a database.  Problem solved right?

### Maybe.  How smart is your widget manager?

You could create a single widget with both the application and the
database.  That would allow you to use `systemd` to achieve _Node HA_
- the automated _and deterministic_ management of a set of resources
within a single node.

In some ways, `systemd` looks a lot like a cluster manager.  It knows
about sets of services, it knows relationships between them (so that
the database is started before the application) and it knows how to
recover the stack if something fails.

Unfortunately you're out of luck if a failure on node A requires
recovery (of the same or a different service) on nodeB because the
caveat is that all these relationships must exist within a single
node.

This of course is not the container model - which likes to have each
service in its own widget.  More importantly, you always need to pay
the database synchronisation cost for every failure which is not
ideal.

Alternatively, if your application isn't active-active, you don't even
get the option of combining them into a single flavour of widget.

By splitting them up into two however, the synchronisation cost is
only payable when a database widget dies.  This improves your recovery
time and makes the widget purists happy, but now you make need to make
sure the application doesn't start until the database is both running,
synchronised and available to take requests.

> About now you might be tempted to think about putting retry loops in
> the application instead.
>
> Chances are however, there is another service that is a client of
> the application (and there is a client of the client of the
> ...).
>
> Every time you build in another level of retry loops, you increase
> your failure detection time and ultimately your downtime.

Hence the question: How smart is your widget manager?

* It needs to ensure there are at least N copies of a widget active.
* It might need to ensure there are less than M copies available.
* It might need to ensure the application starts after the database.
* It might need to be able to stop the application if not enough
  copies of the database are around and/or writable. Perhaps it got
  corrupted?  Perhaps someone needs to do maintenance?

Lets assume the widget manager can do these things.  Most can, that
means we're done right?

### Maybe. What happens if the widget manager cannot see one of its peers?

Just because the widget manager cannot see one of its peers with a
bunch of application widgets, does not mean they're not happily
swallowing client requests they can never process and/or writing to
the data store via some other subnet.

If this does not apply to your application, consider yourself blessed.

For the rest of us, in order to preserve data integrity, we need the
widget manager to take steps to ensure that the peer it can no longer
see does not have any active widgets.

This is one reason why `systemd` is rarely sufficient on its own.

_Hint: A great way to do this is to power off the host_

### Are you done yet?

One thing we skipped over is where the database itself is storing its
state.

If you were using bare metal, you could store it there - but thats
old-fashioned.  Storing it in the KVM image or docker container isn't
a good idea, you'd loose everything if the last container ever died.

Projects like glusterfs are options, just be sure you understand what
happens when partitions form.

If you're thinking of something like NFS or iSCSI, consider where
those would come from.  Almost certainly you don't want a single node
serving them up - that would introduce a single point of failure and
the whole point of this is to remove those.

You could add a SAN into the mix for some hardware redundancy, however
you need to ensure either:

- *exactly* one node accesses the SAN at any time (active/passive), or
- your filesystem can handle concurrent reads and write from multiple
  hosts (active/active)

Both options will require quorum and fencing in order to reliably hand
out locks.  This is the sweet-spot of a full blown cluster manager,
_System HA_, and why traditional, scary, cluster managers like
Pacemaker and Veritas exist.

Unless you'd like to manually resolve block-level conflicts after a
split-brain, some part of the system needs to rigorously enforce these
things.  Otherwise its widget managers all the way down.

### One of Us

Once you have a traditional cluster manager, you might be surprised
how useful it can be.

A lot of applications are resilient to failures once they're up, but
have non-trivial startup sequences.  Consider RabbitMQ:

* Pick one active node and start rabbitmq-server
* Everywhere else, run
  * Start rabbitmq-server
  * rabbitmqctl stop_app
  * rabbitmqctl join_cluster rabbit@${firstnode}
  * rabbitmqctl start_app

Now the Rabbit's built-in HA can take over but to get to that point:

- How do you pick which is the first node?
- How do you tell everyone else who it is?
- Can rabbitmq accept updates before all peers have joined?
- Can your app?

This is the sort of thing traditional cluster managers do before
breakfast.  They are afterall really just distributed finite state
machines.

Recovery can be a troublesome time too:

> http://previous.rabbitmq.com/v3_3_x/clustering.html
>
> the last node to go down must be the first node to be brought
> online. If this doesn't happen, the nodes will wait 30 seconds for
> the last discconected node to come back online, and fail afterwards.

Depending on how the nodes were started, you may see some nodes
running and some stopped.  What happens if the last node isn’t online
yet?

Some cluster managers support concepts like dual-phased services (or
"Master/slave" to use the politically incorrect term) that can allow
automated recovery even with constraints such as these.  We have
Galera agents that also take advantage of these capabilities - finding
the 'right' node to bootstrap before synchronising it to all the
peers.

### Final thoughts

HA is a spectrum, where you fit depends on what assumptions you can
make about your application stack.

Just don't make those assumptions before you really understand the
problem at hand, because retrofitting an application to remove
simplifying assumptions (such as only supporting pets) is even harder
than designing it in in the first place.

Whats your minimal viable cluster?