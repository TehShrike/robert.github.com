---
title: "PFAB#4: Exception handling and coping with failure"
layout: post
tags: [Programming Projects for Advanced Beginners]
og_image: https://robertheaton.com/images/pfab-cover.png
redirect_from:
  - /pfab4
published: false
---
> Welcome to week 4 of Programming Feedback for Advanced Beginners. In this series I review a program [sent to me by one of my readers][feedback]. I analyze their code, highlight the things that I like, and discuss the things that I think could be better. Most of all, I suggest small and big changes that the author could make in order to take their code to the next level.
>
> (To receive all future PFABs as soon as they’re published, [subscribe by email][subscribe] or [follow me on Twitter][twitter]. For the chance to have your code analyzed and featured in future a PFAB, [go here][feedback])

Last week we began analyzing Michael Troyer's commute-time-measuring tool. If you missed it, [read that post first][pfab3] to catch up. This week we're going to look at how Michael could improve the way in which his program handles errors.

*(You may want to [open Michael's code in GitHub][commute-times] to reference while you read this post.)*

## You are already perfect

Before we start, I'd like to make it clear that Michael's code is already perfect for what it is. The logic is already entirely correct, and the program already achieves exactly what he wants it to. Sure, the world would be a fractionally tidier place if he tweaked a few things (as we're about to discuss), but no one's life would be materially improved. If in a year's time Michael is looking at another job in another city and wants to re-use his tool, he should just run his code again exactly as it is, and not worry about whether his error handling is as precise as it can possibly be.

On the other hand, suppose that Michael wants to consider expanding the program into something more complex; maybe with a web UI, or the ability to query different map engines, or to consider public transport or walking, or if he just wants to improve his programming chops for the sake of it. In this case I think that he should seriously consider the modifications that I'm about to suggest.

## Even better if: exception handling

Usually when a program throws an exception, it explodes, stops executing, and prints a stacktrace to the terminal. For example:

```
Traceback (most recent call last):
  File "process_data.py", line 6, in <module>
    get_data()
  File "process_data.py", line 4, in get_data
    response['id']
KeyError: 'id'
```

If you want your program to be able to gracefully deal with certain types of exceptions, you can use *exception handling* to prevent the explosion and tell your program how to recover. In most languages this looks something like:

```python
try:
    id = response['id']
except KeyError:
    print("Response does not contain an ID!")
    id = None
```

Michael makes extensive use of exception handling in his program, because he wants his data-collection program to run and collect data forever, or at least until he manually cancels it. If an exception is thrown while his program is running then he generally doesn't want it to explode. Instead he wants it to *catch* the exception, print out an error message to help with debugging, and continue running.

This is in principle a very sensible idea. It would be very annoying if Michael missed out on several days-worth of data because a request to Google Maps randomly failed for some transient reason that he didn't notice. By catching exceptions Michael allows his program to recover from problems like this and to continue its work.

Catching exceptions might sound like an always-amazing idea. Surely it's always best to stop your program from catching fire? However, errors are often invaluable feedback that something is wrong with your code that you need to pay attention to.

This week we're going to look at two common exception handling mistakes: *swallowing* errors whole, and catching too many types of error.

## Catching database errors

Michael's program contains a function called `Database#add_data`. It is particularly over-zealous in its handling of exceptions:

```python
class Database:

    def add_data(self, data):
        """
        Add data to database. Expects a dictionary.
        """
        try:
            # (Actual code snipped for brevity)
            write_to_database(data)
        except Exception as e:
            print('Error writing data to database:', e)
```

In English, this code means: "`try` to insert a record into the database. If any exceptions, of any sort, are raised, print an error message. *Don't* re-raise the exception (often known as *swallowing* the exception), because I want the program to keep on running."

There are 2 things I would want to change about this function before I would feel comfortable including it in an important, production system:

### 1. Don't swallow your own errors

I don't think that the `Database#add_data` function should even be responsible for catching and swallowing its own errors. To see why, imagine that you are another programmer working on this project, and you want to use `add_data` in your own code. You call the function and run your code, but you don't see any records created in the database. You tear your hair out for an hour. Eventually you scroll back through all of your output logs, and notice a line saying `Error writing data to database: SqliteException`. You realize that your writes to the database were failing for easily-fixable reasons, but the `add_data` function was trying to be helpful by preventing these errors from causing your program to halt. You wish that `add_data` had instead allowed the error to be raised properly so that that you could have known that there was a problem with your code.

In short, don't catch and swallow or ignore errors unless you're sure you know what you're doing.

### 2. Don't catch every type of Exception

The line `except Exception as e` means "catch *every* type of exception". Even if you do want your code to catch some types of exception, catching every type of exception is almost always a mistake.

When you catch every type of exception, you catch *every* type of exception, including some that are indicative of boneheaded bugs. For example, `except Exception as e` will even catch `NameError`, an error that indicates that a variable is not defined and suggests that you have typoed something somewhere. You almost always want to know about `NameError`s, and never want them to be caught.

Instead, we should work out what types of errors we expect to see and therefore aren't concerned by, and only catch those errors (for example: `except sqlite3.Error as e`). That said, in this database situation, I don't think that there *are* any errors that we expect to see. If `add_data` throws an exception then this suggests that there is likely a bug that we need to fix. I would therefore remove the `try`/`except` block from this function entirely.

Next let's look at a different part of the program, in which I think that error handling is entirely appropriate: the code that queries the Google Maps API.

## Catching exceptions from the Google Maps API

Michael's program talks to the Google Maps API by sending requests to it over the internet. Despite everyone's best efforts, sometimes requests over the internet fail for no particular reason. Maybe Google's API has a brief hiccup, or maybe Michael's wi-fi goes down briefly at 3am. Without exception handling, a single failed request to the Google Maps API would cause Michael's program to explode and stop collecting data.

To keep Michael's program chugging along in the face of intermittent internet issues, we should catch Google Maps errors that we believe are *transient*, meaning that they will go away if we try again. Our code might change from:

```python
data = get_commute_data(origin, destination, api_key)
database.add_data(data)
```

to:

```python
try:
    data = get_commute_data(origin, destination, api_key)
    database.add_data(data)
except googlemaps.ServiceUnavailable as e:
    print("Error querying GoogleMaps:", e)
```

This targeted error handling of only `googlemaps.ServiceUnavailable` exceptions means that we get the best of both worlds - our program doesn't explode if Google Maps is briefly down, but we are also notified of genuine, `NameError`-style bugs in our code.

Since our Google Maps request still failed, we will miss out on collecting any data for this iteration. This is a shame. Let's therefore take a look at how we can use a technique called *automatic retries* to repeatedly run our query until it succeeds. Let's also study a further technique called *exponential backoffs* that allows us to perform these retries safely and carefully, without spamming Google and incurring their automated wrath.

### Automatic retries and exponential backoffs

Michael's `get_commute_data` function for querying the Google Maps API starts like this:

```python
def get_commute_data(origin, destination, api_key):
    gmaps = googlemaps.Client(key=api_key)
    now = datetime.now()
    best_case = gmaps.directions(
            origin,
            destination,
            mode='driving',
            traffic_model='optimistic',
            departure_time=now)[0]
    # ... more logic goes here ...
```

As we know, if the Google Maps API has a hiccup then this version of `get_commute_data` will fail and throw an exception. To add basic, naive automatic retries, we could update function as follows:

```python
# EDITOR'S NOTE: this code has several problems with
# it! (see below)
def get_commute_data(origin, destination, api_key):
    gmaps = googlemaps.Client(key=api_key)
    now = datetime.now()
    while True:
        try:
            # EDITOR'S NOTE: arguments snipped for brevity
            best_case = gmaps.directions(...)
            break
        except googlemaps.Error as e:
            print("Error querying GoogleMaps:", e)
    # ... more logic goes here ...
```

This code uses a `while True` loop to repeatedly query the Google Maps API. If the call to `gmaps.directions` returns successfully and does not raise an exception, the `break` line is executed, the loop is exited, and the program continues with its work. However, if the query raises a `googlemaps.Error` exception, then the program catches it, prints an error message, and goes back to the top of the `while` to try the request again. This loop repeats until a request succeeds.

Stop and think - what are three problems with this approach? In particular, what happens if the Google Maps API is down for several hours because of a request volume overload? And what happens if your request is fundamentally invalid because you accidentally passed in `origin=None`?

Well:

1. Looping *forever* until a request succeeds is a bit much. After the 5th or so failed request, we should probably raise an actual exception and tell the rest of our program that something is going substantially wrong with our requests to Google
2. Catching all types of `googlemaps.Error`s is too broad, for very similar reasons to why catching all `Exception`s was too broad in our previous snippet. If our request is fundamentally invalid or malformed then it has no chance of ever succeeding, but we will still keep repeating it forever (or until we run out of retries). This is a waste of time and energy. Instead, we should think carefully about what specific types of error we actually want to retry. This will most likely be errors like `googlemaps.ServiceUnavailable`, and any others that we believe may be transient and not indicative of a logic mistake on our part.
3. If a request fails, we currently send our retry request almost instantly, without giving the Google Maps API any time to shake off its problems. This might make the API server's situation worse, or (more likely) Google will get grumpy at us and will *rate-limit* our requests, locking down our access to their system. We should therefore wait for a brief period before retrying our request. This is known as a *backoff*. The simplest *backoff strategy* is to wait for a fixed amount of time in between each request, which is known as a *constant backoff*. For example:

```python
def get_commute_data(origin, destination, api_key):
    gmaps = googlemaps.Client(key=api_key)
    now = datetime.now()

    # Retry a maximum of 5 times
    for i in range(0, 5):
        try:
            # EDITOR'S NOTE: arguments snipped for brevity
            best_case = gmaps.directions()
            break
        except googlemaps.ServiceUnavailable as e:
            print("Error querying GoogleMaps:", e)

            # If we have run out of retries, re-raise the
            # error so that the rest of the program
            # can deal with it.
            if i == 4:
                raise e

            # UPDATED CODE: sleep for 1 second before retrying
            sleep(1)
    # ... more logic goes here ...
```

Sometimes a constant backoff is too aggressive, and results in too many requests being made too quickly. In a *linear backoff* you increase the amount of time that you wait by a fixed amount with each retry. The first wait might be 1 second, then 2 seconds, then 3, 4, 5, and so on. In an *exponential backoff* you *multiply* the amount of time that you wait by a fixed amount with each retry. First you wait for 1 second, then 2 seconds, then 4, 8, 16, 32, and so on. For our purposes a constant backoff would be perfectly reasonable, but it's still useful to keep the other backoff strategies in mind for more complex or sensitive applications.

*Exercise for the reader: how would you update the above code to perform linear and exponential backoffs?*

## Conclusion

This week we've seen how to think about error handling. As a very rough rule of thumb, I think that your first instinct should be to not handle errors at all. If you think that your program can gracefully deal with specific types of errors, then by all means go ahead. But errors aren't something to be afraid of and squashed - they're a very useful mechanism for telling you that something has gone wrong.

Next week will be our final week looking at Michael's commute times program. We're going to spend it simplifiy and reducing the length of his database-querying code from 100 to 30 lines. Don't miss it.

To tide you over until then:

* Explore the archives: [How does online tracking actually work?][online-tracking]
* To receive all future PFABs as soon as they're published, [subscribe to the mailing list][subscribe]
* If you've written some code that you'd like feedback on, [send it to me!][feedback]
* Was any of this post unclear? [Email][about] or [Tweet][twitter] at me with suggestions, comments, and feedback on my feedback. I'd love to hear from you.

### Advanced appendix

Keen-eyed readers may have noticed that we began our discussion of `get_commute_data` by handling errors outside of the function:

```python
try:
    data = get_commute_data(origin, destination, api_key)
    database.add_data(data)
except googlemaps.ServiceUnavailable as e:
    print("Error querying GoogleMaps:", e)
```

and then switched to handling errors inside the function when we started to discuss automated retries:

```python
def get_commute_data(origin, destination, api_key):
    gmaps = googlemaps.Client(key=api_key)
    now = datetime.now()

    # Retry a maximum of 5 times
    for i in range(0, 5):
        try:
            best_case = gmaps.directions(
                origin, destination, mode='driving', traffic_model='optimistic', departure_time=now)[0]
            break
        except googlemaps.ServiceUnavailable as e:
            print("Error querying GoogleMaps:", e)
            if i == 4:
                raise e
            # Sleep for 1 second before retrying
            sleep(1)
    # ... more logic goes here ...
```

I chose to catch these errors at different points in our code because of the different purposes behind catching them. The first code snippet is concerned with how the overall program should handle complete Google Maps failures. Should the program exit? Try a different database engine? These are big, structural questions that the `get_commute_data` function should not have to know anything about. The job of `get_commute_data` is to query Google Maps for data and raise an exception if something goes wrong. It shouldn't have to know anything about what the rest of the program does with this data.

However, it's perfectly logical for `get_commute_data` to be responsible for handling automated retries. Suppose that you ask your personal assistant (for the purposes of this analogy assume you have a personal assistant) to call your friend and deliver a phone message. It's very reasonable for your assistant to be responsible for retrying the call without telling you if they can't immediately get through to your friend. You don't care how many attempts it takes, you just want your message to be delivered eventually. However, if your assistant still can't get through after a few days of trying, and concludes that they should give up, they should bubble up the problem to you so that you can decide how to deal with it.

As always, there's no single rule for the universal best place to catch and handle errors. It all depends on which part of your code should be responsible for knowing what to do when something goes wrong.

[pfab1]: https://robertheaton.com/pfab1
[pfab2]: https://robertheaton.com/pfab2
[pfab3]: https://robertheaton.com/pfab3
[commute-times]: https://github.com/robert/programming-feedback-for-advanced-beginners/blob/master/editions/3-4-5-commute-times/original
[online-tracking]: https://robertheaton.com/2017/11/20/how-does-online-tracking-actually-work/
[about]: https://robertheaton.com/about
[twitter]: https://twitter.com/robjheaton
[feedback]: https://robertheaton.com/feedback
[subscribe]: https://advancedbeginners.substack.com
