---
title: 'Distributed Transactions with TorqueBox'
author: Jim Crossley
layout: news
tags: [ xa, transactions ]
---

[MarkL]: http://twitter.com/nmcl
[Bob]: http://twitter.com/bobmcwhirter
[Nick]: http://twitter.com/nicksieger
[simple]: http://diveintomark.org/archives/2010/02/23/simplicity-is-hard-lets-go-shopping
[fun]: http://people.apache.org/~acmurthy/WhyIsProgrammingFun.html
[MessageProcessors]: http://torquebox.org/documentation/LATEST/messaging.html#messaging-consumers
[Backgroundable]: http://torquebox.org/documentation/LATEST/messaging.html#backgroundable
[art]: http://api.rubyonrails.org/classes/ActiveRecord/Transactions/ClassMethods.html
[ar-jdbc]: https://github.com/jruby/activerecord-jdbc-adapter
[rails]: http://torquebox.org/documentation/LATEST/web.html#rails
[jbossds]: https://docs.jboss.org/author/display/AS7/DataSource+configuration
[IronJacamar]: http://www.jboss.org/ironjacamar
[HornetQ]: http://hornetq.org
[JBossTS]: http://www.jboss.org/jbosstm
[JRuby]: http://jruby.org
[bugs]: https://jira.jboss.org/jira/secure/CreateIssue.jspa?issuetype=1&pid=12310812
[ir]: http://eaipatterns.com/IdempotentReceiver.html
[2pc]: http://en.wikipedia.org/wiki/Two-phase_commit_protocol
[xa]: http://en.wikipedia.org/wiki/X/Open_XA
[Infinispan]: http://infinispan.org

# TorqueBox is [Atomic, Dog](http://www.youtube.com/watch?v=LuyS9M8T03A)

Ever since I came to work at Red Hat, my [boss'][Bob] [boss][MarkL],
the JBoss CTO and reputed transactions expert, has been politely
nagging us to bring transactions to TorqueBox. Well, I'm proud to
announce we finally got around to it: TorqueBox 2.x features
distributed XA transaction support in what we believe is a darn
elegant Ruby API... because it's mostly transparent. ;)

Few things get an enterprise architect's blood pumping (boiling?) more
than distributed transactions, though many anti-enterprisyists dismiss
them as heavyweight, preferring comparatively complex alternatives,
e.g. [idempotent receiver][ir], that are arguably just as
resource-intensive and often more error-prone.

To those naysayers, we proudly say **"Pfffftt!"**

The goal of the TorqueBox project is to make robust enterprise
services [simple] and [fun] to use, combining the power of JBoss with
the expressiveness of Ruby. Distributed XA transactions are our latest
attempt at achieving that goal.

# Some Terminology

It's important to understand the difference between a conventional
database transaction and a *distributed* transaction: *multiple
resources* may participate in a distributed transaction. The most
common example of a transactional resource is a relational database,
of course. But other examples include [message brokers][HornetQ] and
[NoSQL data grids][Infinispan]. Distributed transactions allow your
application to say, tie the success of a database update to the
delivery of a message, i.e. the message is only sent if the database
update succeeds, and vice versa. If either fails, both rollback.

[X/Open XA][xa] is a standard specification for implementing
distributed transactions. It uses a [two-phase commit (2PC)][2pc]
protocol. Often these terms are used interchangeably to refer to the
same thing.

# Messaging

Here's how we've made messaging transactional in 2.x:

- By default, all [MessageProcessors] are transactional, so each
  `on_message(msg)` invocation demarcates a transaction. If no
  exceptions are raised, the transaction commits. Otherwise, it rolls
  back.
- Any messages published to any JMS destinations automatically become
  part of the current transaction, by default. So they won't be
  delivered until that transaction commits.
- All [Backgroundable] tasks are transactional, so if invoked within a
  transaction, it will only start when the transaction commits.
- Any manipulations of your Rails ActiveRecord models (persisted to
  your XA-compliant database) within `on_message(msg)` will become
  part of its transaction.

This bears repeating: all the above you get for free when your app is
deployed on TorqueBox 2.x. No extra config is required. 

# TorqueBox.transaction() {...}

In addition, we've introduced a new method, `TorqueBox.transaction`,
that can be used to enlist multiple XA-compliant resources into a
single distributed transaction from anywhere in your application. This
includes message destinations, background tasks, and the
[Infinispan]-backed TorqueBox cache, which are all automatically
transactional, by default.

But wait, there's more!

You can also include Rails ActiveRecord models, which are enhanced
when run in TorqueBox, so contrary to the
[ActiveRecord Transactions docs][art]:

- Transactions **can** be distributed across database connections when
  you have multiple class-specific databases.
- The behavior of nested transaction rollbacks won't surprise you: if
  the child rolls back, the parent will, too, excepting when the
  `:requires_new=>true` option is passed to the child.
- Nested transactions should work correctly on more than just MySQL
  and PostgreSQL; in theory, they should work on any database
  providing an XA driver known to work with JBoss, including H2,
  Derby, Oracle, SQL-Server, and DB2. (Sqlite3 doesn't support XA)
- Callbacks for `after_commit` and `after_rollback` work as you would
  expect for models involved in a `TorqueBox.transaction`

# Configuration

All ActiveRecord-dependent apps running on TorqueBox must use the most
excellent JRuby [activerecord-jdbc-adapter][ar-jdbc] as described in the
[docs][rails].

Typically, [XA datasources are configured][jbossds] in non-standard,
often complicated ways involving XML and occasional jar file
manipulation. These datasources are then bound to a logical name in a
JNDI naming service, to which your application refers at runtime. The
[ar-jdbc] adapter indeed supports putting that JNDI name in your Rails
`database.yml` config file.

But that is so totally gross.

Referring to a JNDI name makes your app dependent on a JNDI service,
which is fine when your app is running inside TorqueBox, but it breaks
your ability to run database migrations, the `rails console`, or
anything else you might do *outside* of TorqueBox.

So TorqueBox creates those XA datasources for you automatically when
your app deploys, using the conventional [ar-jdbc] settings in your
`database.yml`. When running outside of TorqueBox,
`TorqueBox.transaction` should gracefully resolve to a no-op.

Bottom line: **no extra configuration is required**.

Line below bottom line: hopefully not a deal breaker, but we don't
fully support using ERB scriptlets in `database.yml` yet.

# Let's see some code, kk? kk!

Here's a typical TorqueBox message handler:

<pre class="syntax ruby">class Processor < TorqueBox::Messaging::MessageProcessor
  always_background :send_thanks_email
  def on_message(msg)
    thing = Thing.create(:name => msg)
    inject('/queues/post-process').publish(thing.id)
    send_thanks_email(thing)
    # raise "rollback everything"
  end
end</pre>

Of course, the commented raise statement is contrived, but it
illustrates the point that if any exception occurs during the
execution of `:on_message`, no `Thing` instance is created, no
`post-process` queue receives a message, and no gratitude is emailed.

The above code is certainly valid in TorqueBox 1.x, but uncommenting
the raise statement would only cause the message to be redelivered, by
default another 9 times, effectively resulting in the creation of 10
like-named `Thing` objects, 10 messages sent to the `post-process`
queue, and 10 emails of gratitude sent out. Without transactions,
you'll need to introduce compensation logic -- more design, code,
tests, bugs, and meetings -- to ensure the integrity of your data.

The `on_message` method is invoked after an implicit transaction is
started, but you can do this explicitly yourself using
`TorqueBox.transaction`. It accepts the following arguments:

- An arbitrary number of resources to enlist in the current
  transaction *(you probably won't ever use this)*
- An optional hash of options; currently only `:requires_new` is
  supported, defaulting to `false` *(this might come in handy)*
- A block defining your transaction *(kind of the whole point)*

If the block runs to completion without raising an exception, the
transaction commits. Otherwise, it rolls back. And that's pretty much
all there is to it.

Here's the "surprising" example from the [Rails docs][art] which
results in the creation of both 'Kotori' and 'Nemu':

<pre class="syntax ruby">User.transaction do
  User.create(:username => 'Kotori')
  User.transaction do
    User.create(:username => 'Nemu')
    raise ActiveRecord::Rollback
  end
end</pre>

And here's the TorqueBox version which creates neither 'Kotori' nor
'Nemu', as you would expect:

<pre class="syntax ruby">TorqueBox.transaction do
  User.create(:username => 'Kotori')
  TorqueBox.transaction do
    User.create(:username => 'Nemu')
    raise ActiveRecord::Rollback
  end
end</pre>

Alternatively, you can fix the original, surprising code by simply
passing it to `TorqueBox.transaction`:

<pre class="syntax ruby">TorqueBox.transaction do
  User.transaction do
    User.create(:username => 'Kotori')
    User.transaction do
      User.create(:username => 'Nemu')
      raise ActiveRecord::Rollback
    end
  end
end</pre>

To prevent Nemu's failures from discouraging Kotori, use
`:requires_new` just like with ActiveRecord, and only Kotori will be
created:

<pre class="syntax ruby">TorqueBox.transaction do
  User.create(:username => 'Kotori')
  TorqueBox.transaction(:requires_new => true) do
    User.create(:username => 'Nemu')
    raise ActiveRecord::Rollback
  end
end</pre>

Message destinations are automatically enlisted into the active
transaction, so transactionally sending a message and persisting a
`Thing` instance, for example, is simple:

<pre class="syntax ruby">TorqueBox.transaction do 
  inject('/queues/foo').publish("a message")
  thing.save!
end</pre>

Occasionally, you may not want a published message to assume the
active transaction. In that case, pass `:tx => false`, and the message
will be delivered whether `thing.save!` succeeds or not.

<pre class="syntax ruby">TorqueBox.transaction do 
  inject('/queues/foo').publish("a message", :tx => false)
  thing.save!
end</pre>

Be careful publishing messages outside the transaction of a
`MessageProcessor`, though. Exceptions raised from `on_message`
trigger redelivery attempts, 9 by default, so...

<pre class="syntax ruby">class Processor < TorqueBox::Messaging::MessageProcessor
  def on_message(msg)
    inject('/queues/post-process').publish("foo", :tx => false)
    raise "you're gonna post-process 10 'foo' messages"
  end
end</pre>

# The Fine Print

It's still early days for this, so be gentle when reporting [bugs]!
It's only been tested with Rails 3.0 and 3.1 so far, on H2,
PostgreSQL, and MySQL backends, but not with a "real application", if
you know what I mean.

There seem to be issues when running a Rails 3.1 app under 1.9 mode in
JRuby 1.6.4, but these should be addressed in JRuby 1.6.5.

Up until we officially release TorqueBox 2.0, the API is subject to
further coagulation. Feedback is always appreciated, of course,
especially if you're able to test some of the database types we don't
have access to, so
[keep those cards and letters coming in!](http://www.youtube.com/watch?v=1mzmO8eovt8)

# Acknowledgments

Transactions are hard.

Much thanks and credit goes to the JBoss teams and communities behind
[IronJacamar], [HornetQ], [JBossTS], [Infinispan] and everyone else I
forgot. You guys rock!

Of course, none of this would be possible without the excellent work
of -- and continued support from -- the [JRuby]
community. [Nick Sieger][Nick] in particular deserves a special
shout-out for his care and feeding of [ar-jdbc].

And last but not least, my [team](http://projectodd.org), and
especially [Bob], who pretty much single-handedly wrote all the
"automatic XA datasouce creation from database.yml" magic himself.

Thanks! :)
