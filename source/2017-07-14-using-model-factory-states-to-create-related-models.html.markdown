---
date: 2017-07-14
title: Using Model Factory States To Create Related Models
tags: php
---

_**Update 2019:** my opinions on this topic have changed quite a bit since it was originally published. See the update section at the bottom of this article_

Laravel’s model factories can be used to automatically create related models when the factory is called. This is a great way to DRY up the arrange portion of your tests. Consider the following factory definition:

```php
$factory->define(Comment::class, function ($faker) {
    return [
        'post_id' => factory(Post::class)->lazy(),
        'content' => $faker->sentence()
    ];
});
```

This factory will create a comment along with a post whenever it is called unless the post is explicitly provided.

```php
// creates a comment and a post
factory(Comment::class)->create();

// uses the provided post instead of creating one
$post = factory(Post::class)->create();
$comment = factory(Comment::class)->create([
    'post_id' => $post->id
]);
```

While the first example can save you from having to repeat the post creation call across your test suite, I think it suffers from two problems. First, when a comment is created with `$comment = factory(Comment::class)->create();` I cannot tell just by looking at the test that a post has also been created. This implicit behavior could confuse my future self or anyone else working with the code. Second, I will no longer be able to use the factory as an argument to the save() method:

```php
$post->comments()->save(factory(Comment::class)->make());
```

This code will correctly assign the comment to the post, as expected. However, since the post_id is not explicitly provided to the comment factory an additional unassigned post will be created.

Despite its convenience I think these two problems limit the usefulness of the lazy() method. Fortunately Laravel gives us a solution - model factory states:

```php
$factory->define(Comment::class, function ($faker) {
    return [
        'content' => $faker->sentence()
    ];
});

$factory->state(Comment::class, 'withPost', function ($faker) {
    return [
        'post_id' => factory(Post::class)->lazy(),
    ];
});
```

By defining a state we can use the base comment factory as an argument to the save() method without an additional post being created. When we need a comment with a post we can be explicit and use the `withPost` state.

```php
// No unassigned post since the base comment factory does not define it
$post->comments()->save(factory(Comment::class)->make());

// Creates the post but is explicit about it
$comment = factory(Comment::class)->states(['withPost'])->create();
```

#### Update 2019

I now disagree with much of what is written above. A model factory should include everything that is required to create a valid model (but nothing more). If a post is required to create a comment then that definitely belongs in the comment factory. Otherwise every time we use that factory in our tests we are going to have to deal with assigning the post. The original article listed two problems:

> When a comment is created with `$comment = factory(Comment::class)->create();` I cannot tell just by looking at the test that a post has also been created.

This is a good thing. If the post is not needed by the test then I don't want to see it. Having details about the post in a test that is only testing the comment is noise that obscures the meaning of the test.

> I will no longer be able to use the factory as an argument to the save() method ... `$post->comments()->save(factory(Comment::class)->make());` ... since the post_id is not explicitly provided to the comment factory an additional unassigned post will be created.

I do like the simplicity of passing the factory to the save method but I no longer believe it's worth the tradeoffs. The alternative is not significantly worse:

```php
$post = factory(Post::class)->create();
$comment = factory(Comment::class)->create([
    'post_id' => $post->id
]);
```

And it's fine because we are only going to do this when the test actually needs both the post and a comment. Our test setup code is inline with what is to come in the test.

I still use model factory states frequently, but when it comes to creating relationships with them I recommend them in situations where the related model is not a required field. A potential use case could be the other side of this comment -> post relationship:

```php
$post = factory(Post::class)->states(['withComments'])->create();
```

This communicates that our test needs a post with some comments but the details of these comments are not going to be important.
