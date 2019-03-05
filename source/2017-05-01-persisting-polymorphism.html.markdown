---
title: Persisting Polymorphism
date: 2017-05-01
tags: php
---

Recently I worked on a project where a user's choice needed to permanently modify the behavior of an object. The gist of the project is that a user is creating a job posting and at some point they will choose how they wish to publish the job: using a job credit, as a daily rate, on a recruiting plan, etc. The choice of publishable type alters the behavior of the job every time the user decides to change its status (from draft to open, draft to scheduled, open to closed). For example: when a job credit job is changed from draft status to open status the system has to verify that a credit is available and then use the credit on the job, but a daily rate job would instead charge the users credit card.

You can imagine how this could lead to some pretty nasty code:

```php
public function open()
{
    if ($this->publishable_type === 'credit') {
        // verify credit is available and anything else that
        // needs to be done before a credit job is opened
    } elseif ($this->publishable_type === 'daily') {
        // charge credit card or anything that needs to be
        // done before a daily job is opened
    } elseif ($this->publishable_type === 'plan' {
        // an elseif for any other types
    }

    $this->status = 'open';
    $this->save();
    // and any other code that has to be executed when a job is
    // opened regardless of which publishable type is chosen

    if ($this->publishable_type === 'credit') {
        // assign credit to job
    } elseif ($this->publishable_type === 'daily') {
        // whatever needs to be done for daily job
    } elseif ($this->publishable_type === 'plan' {
        // an elseif for any other types
    }
}
```

This implementation would result in the `open()` method being modified anytime a new publishable type is added. Similar methods would have to exist for the other job status changes as well. Not a maintainable approach. If we could abstract the behavior for each of the types into their own classes this method could be significantly improved. Consider if the open method could look like this instead:

```php
public function open()
{
    $this->publishableType()->beforeOpen();

    $this->status = 'open';
    $this->save();

    $this->publishableType()->afterOpen();
}
```

Much nicer.

To make this work the `publishableType()` method needs to return a new instance of the type class. We can take a cue from Laravel’s polymorphic relationships and save the namespace of the type class to a `publishable_type` column on the job table. Meaning that the column would contain something along the lines of: `App\Daily`, `App\Credit` or `App\Whatever`. The `publishableType()` implementation looks like:

```php
public function publishableType()
{
    return new $this->publishable_type($this);
}
```

And that is it. As long as each publishable type implements the same interface then changing the behavior of a job is a simple as persisting a different publishable type to the database.
