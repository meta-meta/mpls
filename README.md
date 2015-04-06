# mpls

Live-code Max and Max4Live with Clojure.


## Prereqs

- [Max][https://cycling74.com/products/max/]
- [Leiningen][http://leiningen.org/]

## Setting Up

Download `mpls.jar` and copy it into the java lib folder for Max.

Mac: `/Applications/Max 6.1/Cycling '74/java/lib/`
Windows: TODO
Linux: TODO

### Doing Stuff

Launch Max/Max4Live and create an `mxj mpls` object. (If you get the error message "Could not load class 'mpls', double-check if you have the `mpls.jar` in the right place.")

Create a `nrepl start` message box and connect it to your `mxj mpls` box.

![](resources/1.png)

Exit edit mode and click your `nrepl start` object.

You should see a message that says `nrepl server running on port [port]`

Open a term and enter

```
lein repl :connect localhost:[port]
```

using `port` from above.

You should now have a Clojure repl. (You can also use your repl-supporting Clojure editor to connect. Cider is supported with cider-nrepl v0.9.0-SNAPSHOT)

Test it out

```clj
(+ 1 1)
```

Still 2? Good.

Import some stuff

```clj
(require '[mpls :refer :all])
```

Do you first thing

```clj
(post "Here I am!")
```

If you Max logging window is open (Max > Window > Max Window), you will see `Here I am!` printed there.

Now create a `bang` object and connect it to `mpls`.

![](resources/2.png)

Now bang! it.

You should see something like `/bang unimplemented` in the Max window. Let's fix that.

Announce yourself, like a good rock star.

```clj
(hello-mpls!)
```

(`mpls` will dispatch messages sent to the `mpls` box to certain functions defined in the namespace that calls the `hello-mpls!` function, if you care.)

Now define a function called `bang`.

```clj
(defn bang [this inlet] (post "pew! pew! pew!"))
```

Bang it again! Congratulations, you're live-coding Max.

## Outlets

Send a message to an outlet like this:

```clj
(out "message")
```

To see your message, create a message box, wire the first outlet of `mxj mpls` to the second inlet of the message box, then do

![](resources/3.png)

```clj
(out "hello, there")
```

Connect the second outlet to an integer box. Then try

![](resources/4.png)

```clj
(out 1 1024)
```

And this

```clj
(dotimes [i 1000] (out 1 i))
```

See the magic.

## Tying it together

```clj
(def n (atom 0))
(defn bang [this inlet] (out 1 (swap! n inc)))
```

Bang away!

## Other messages

You've seen `bang`, which responds to bangs on the inlet. There is also `int-msg` for ints, `float-msg` for floats, `list-msg` for lists, `dblclick` for mousey things, and `msg` for everything else.

`bang` and `dblclick` take two arguments (`this` which is `mpls` : MaxObject, and the 0-indexed inlet number). `list-msg` and `msg` take three arguments, `this`, inlet, and a vector of args. `int-msg` and `float-msg` take three args: this, inlet, and a number.

Messages sent to `mpls` for message boxes trigger `msg` calls with a vector of their space-delimited contents. An example to illustrate

Create a message box with the text "reset", then

```clj
(defn msg [this inlet [command]]
  (when (= command "reset")
    (do
      (reset! n 0)
      (out 1 @n))))
```

Lets specify what to reset to. Connect another message box "reset 234"

```clj
(defn msg [this inlet [command i]]
  (when (= command "reset")
    (do
      (reset! n (or i 0))
      (out 1 @n))))
```

## Programmatic layout

You're probably pretty tired of creating all these bangs and message boxes by hand, huh.

Try this

```clj
(def my-button
  (mnew "button" 200 200))
```

So much easier, right? Well how about this

```clj
(for [i (range 100)] (mnew "button" (* i 10) 200))
```

Whoa, right. Now go delete them all one-by-one.

Once that's done, it's probably better to assign your creations so you can refer to them later.

```clj
(def buttons (for [i (range 100)] (mnew "button" (* i 10) 200)))
```

What, whuh? Didn't work. Or did it?

```clj
(doall buttons)
```

There they are. Clojure `for` is lazy and not evaluated until needed. `doall` forces evaluation (as does printing to the repl, which is why it worked when we didn't assign the buttons to a var).

You could also wrap the `for` statement in `doall` before assigning to `buttons`.

Okay, now let's delete them. We use `mremove`, which isn't called `remove` because Clojure already has a `remove` function.

```clj
(map mremove buttons)
```

Pretty neat, huh.

`map` is also lazy, FYI. It works only because the repl realizes the results to print them out. Also, it's not really supposed to be used for side-effecting operations like `mremove`. You should be using `doseq`.

```clj
(doseq [b buttons] (mremove b))
```

But that's just not as terse and cool as `(map mremove buttons)`, so do what you like.

## Connecting things

Still connecting objects with a mouse. For shame.

```clj
(def button (mnew "button" 200 200))

(connect button 0 box 0)
```

And disconnect

```clj
(disconnect button 0 box 0)
```

## Cast of characters

Who's this `box` character? It's the the box that encloses `mxj mpls`. In max we connect boxes, which are all subclasses of `MaxBox` in java extensions. (Clojure just wraps the java api. You knew that, right?)

Want to know more about `MaxBox`es? You'll want to read the java api.

Mac: /Applications/Max [version]/java-doc/api/index.html
Windows: TODO
Linux: TODO

You also have access to a few others

- mpls : MaxObject - Our custom java class. Also the first argument to any msg-type function (`msg`, `bang`, `int-msg`)
- box : MaxBox - The box enclosing `mxj mpls`
- patcher : MaxPatcher - the patcher 
- window : MaxWindow - the window

## Warning! Warning!

Now that you've been looking at the API, you're in a great position to crash Max hard. See, Max operates in a couple of different threads, and some threads don't allow some operations.

When creating a traditional java extension, you wouldn't worry too much about this, because the methods you define are generally called for you in the proper context.

However, when we're live-coding, we're crashing our cart right through those guardrails. While I've taken some care to prevent Max-crashing operations in the mpls API, there are no such accommodations when calling the java API directly.

For example, creating graphical objects using the java API will crash Max:

```clj
;; DON'T DO THIS (except for educational purposes)
(.newDefault patcher "button" 200 200 nil)
;; CRASH!
```

How do we circumvent this catastrophe? The `defer` function takes a body of code and runs it safely on the low-priority queue.

```clj
;; This is okay
(defer (.newDefault patcher 200 200 "button" nil))
```

You'll notice that defer returns `nil`. That's because the work is done asynchronously. If you want to do something with your new button, well, you'll need to use callbacks.

Just kidding!!!

Use `defer-sync`, which will wait for your action to complete and return the result.
```clj
;; Notice the return value
(defer-sync (.newDefault patcher 200 200 "button" nil))
```

This is exactly how `mnew` works.

So how do you know if your operation needs to be deferred? ¯\_(ツ)_/¯. Trial and error?

## Misc

### mpls args

In case you care, you can send args when creating `mpls`. One arg defines the nrepl port (default 51580), two args define the number of inlets and outlets you want (you get an extra info outlet, no extra charge!). Three args set `port`, `n-inlets`, `n-outlets`. Weird scheme, isn't it.

```clj
mxj mpls 1235 5 5
```

### matom, matoms, matom->, matoms->, parse

If you're using the java API, you'll need to worry about `Atom`s (`com.cycling74.max.Atom` not Clojure's `atom`).

The Max java API often takes and returns values wrapped in the `Atom` class or a java array of `Atom`s. You can easily create these from regular old Clojure values using `matom` and `matoms`

```clj
(matom 1)
(matom "String")
(matoms [1 "String"])
```

Similarly, you can convert back from `Atom`s using `matom->` and `matoms->`

```clj
(matoms-> (matoms [1 "String"]))
```

You can also create `Atom`s from a space-delimited string using `parse`, which takes a string and returns an array.

```clj
(parse "1 String")
```

### Read the code!

Most, but not all of `mpls`'s functionality is documented in this README. `mpls` is small, small. Don't be afraid to read through the code.

## Wanna help?

Those TODOs in this README need TODOing. Pull requests or issues or emails could solve that if you're working on Windows or Linux.

Also, I haven't tested this on anything on but my machine. Pull requests for bug or missing features are appreciated. Github issues are okay, too. Twitter shaming, not so much (though I'm not on Facebook, so feel free to post hurtful messages about bugs there).

And most of all, if you do something cool with this, post about it, and let me know.

## License

Copyright © 2013-2015 Selah Ben-Haim
Distributed under the Eclipse Public License, the same as Clojure.
