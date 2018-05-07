# Updating an Entity

In the previous tutorial, we created our heart feature! You click on the heart, it
makes an Ajax request back to the server, and returns the *new* number of hearts.
It's all very cute. In *theory*... when we click the heart, it would update the
number of "hearts" for this article somewhere in the database.

But actually, instead of updating the database... well... it does *nothing*, and
returns a new, *random* number of hearts. Lame!

Look in the `public/js` directory: open `article_show.js`. In that tutorial, we
wrote some simple JavaScript that said: when the "like" link is clicked, toggle
the styling on the heart, and then send a POST request to the URL that's in the `href`
of the link. Then, when the AJAX call finishes, read the new number of `hearts`
from the JSON response and update the page.

The `href` that we're reading lives in `show.html.twig`. Here it is: it's a URL to
some route called `article_toggle_heart`. And we're sending the article *slug* to
that endpoint.

Open up `ArticleController`, and scroll down to find that route: it's `toggleArticleHeart`.
And, as you can see... this endpoint doesn't actually do anything! Other than return
JSON with a random number, which our JavaScript uses to update the page.

## Updating the heartCount

It's time to implement this feature correctly! Or, at least, *more* correctly.
*And*, for the first time, we will *update* an *existing* row in the database.

Back in `ArticleController`, we need to use the `slug` to query for the `Article`
object. But, remember, there's a shortcut for this: replace the `$slug` argument
with `Article $article`. Thanks to the type-hint, Symfony will automatically try
to find an `Article` with this slug.

Then, to update the `heartCount`, just `$article->setHeartCount()` and then
`$article->getHeartCount() + 1`. Side note, it's not important for this tutorial,
but in a high-traffic system, this could introduce a *race* condition. Between the
time this article is queried for, and when it saves, 10 other people might have also
liked the article. And that would mean that this would actually save the old, wrong
number, effectively removing the 10 hearts that occurred during those microseconds.

Anyways, at the bottom, instead of the random number, use `$article->getHeartCount()`.

So, now, to the *key* question: how do we run an `UPDATE` query in the database?
Actually, it's the *exact* same as inserting a *new* article. Fetch the entity
manager like normal: `EntityManagerInterface $em`.

Then, after updating the object, just call `$em->flush()`.

But wait! I did *not* call `$em->persist($article)`. We *could* call this...
it's just not needed for updates! When you query Doctrine for an object, it *already*
knows that you want that object to be saved to the database when you call `flush()`.
Doctrine is *also* smart enough to know that it should *update* the object, instead
of inserting a new one.

Ok, go back and refresh! Here is the real heart count for this article: 88. Click
the heart and... yea! 89! And if you refresh, it stays! We can do 90, 91, 92, 93,
and forever! And yea... this is not *quite* realistic yet. On a real site, I should
only be able to like this article *one* time. But, we'll need to talk about users
and security before we can do that.

## Smarter Entity Method

Now that this is working, we can improve it! In the controller, we wrote some code
to increment the heart count by one. But, whenever possible, it's better to move
code *out* of your controller. *Usually* we do this by creating a new service class
and putting the logic there. But, if the logic is simple, it can sometimes live inside
your *entity* class. Check this out: open `Article`, scroll to the bottom, and add
a new method: `public function incrementHeartCount()`. Give it no arguments and
return self, like our other methods. Then, `$this->heartCount = $this->heartCount + 1`.

Back in `ArticleController`, we can simplify to `$article->incrementHeartCount()`.

That's so nice. This moves the logic to a better place, and, it *reads* really
well: 

> Hello Article: I would like you to increment your heart count. Thanks!

## Smart Versus Anemic Entities

*And*... this touches on a somewhat controversial topic related to entities. Notice
that *every* property in the entity has a getter and setter method. This makes
our entity *super* flexible: you can get or set any field you need.

But, sometimes, you might *not* need, or even *want* a getter or setter method.
For example, do we really want a `setHeartCount()` method? I mean, should any part
of the app *ever* need to change this? Probably not: they should just call our
more descriptive `incrementHeartCount()` instead. I *am* going to keep it, because
we use it to generate our fake data, but I want you to *really* think about this
point.

By removing unnecessary getter or setter methods, and replacing them with more
descriptive methods that fit your business logic, you can, little-by-little, give
your entities more clarity. Some people take this to an extreme and have almost
zero getters and setters. Here at KnpU, we tend to be more pragmatic: we *usually*
have getters and setters, but we always look for ways to be more descriptive.

Next, our dummy article data is boring, and we're creating it in a hacky way.
Let's build an awesome fixtures system instead.