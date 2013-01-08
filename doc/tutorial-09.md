# Tutorial 09 - DOM manipulation

In the [last tutorial][1] we introduced `domina.events` namespace to
make our events management a little bit more clojure-ish than just
using CLJS/JS interop. In this tutorial we'are going to face the need
to programmatically manipulate DOM elements as a result of the
occurrence of most DOM events (e.g. `mouseover`, `mouseout`, etc.).

# Introduction

As we already saw, [domina library][2] has a lot to offer for managing
the selection of DOM elements and for handling almost any DOM
event. Let's go on by using it to verify how it could help us in
managing the manipulation of DOM elements which is one of the most
important features of any good JS library and/or framework.

To follow that goal we're going to use again our old shopping
calculator example by adding to its `Calculate` button both a
`mouseover` and a `mouseout` event handlers.  The `mouseover` handler
reacts by adding a simple text, saying "Click to calculate", to the
form itself. The `mouseout` handler, instead, reacts by deleting that
paragraph.  I know, the requirement is very stupid but, as you will
see, pretty representative of a kind of problem you're going to face
again and again in yuor CLJS programming.

# Mouseover event

Let's start by adding a `mouseover` handler to the `Calculate`
button. The first step is to write a function, named `add-help`, which
appends a `div` and the *Click to calculate* inner text to the end of
the `shoppingForm` DOM node. To do that, we're are going to experience
the domina library by using its `append!` function from `domina`
namespace. The documentation attached to `append!` function says that:

> Given a parent and child contents, appends each of the children to all
> of the parents. If there is more than one node in the parent content,
> clones the children for the additional parents. Returns the parent
> content.

And here is a simple example of `append!` usage from [domina readme][3].

```clojure
;;; from domina readme.md
(append! (xpath "//body") "<div>Hello world!</div>")
```

which appends a `div` node to the end of the `body` node. It uses
`xpath` to select a single parent (i.e. `body`) and a `string` to
represent a single `div` fragment to be added to the parent.

I don't know about you, but I don't feel to be at home with `xpath`
and I limit myself to use it just where CSS selector will fail
(e.g. ancestor selection) or become to complex to be managed and/or
maintened.

Anyway, domina offers you three options for nodes selection:

* `xpath` from `domina.xpath` namespace
* `sel` from `domina.css` namespace
* `by-id` and `by-class` from `domina` namespace

Thankfully `append!` accepts, as a first argument, any domina
expression which returns one or more `content` (i.e. one or more DOM
node). This means that, for such a stupid case, we can safely use
`by-id` selector to select the parent to be passed to `append!`.

Here is the `add-help` definition to be added to `shopping.cljs` file.

```clojure
(defn add-help []
  (dom/append! (dom/by-id "shoppingForm")
               "<div class='help'>Click to calculate</div>"))
```

> NOTE 1: we added `class='help'` attribute to the appended `div` in
> such a way that we can later delete it.

We can now add `add-help` handler to the `mouseover` event of the
`calc` button like so:

```clojure
(defn ^:export init []
  (when (and js/document
             (.-getElementById js/document))
    (ev/listen! (dom/by-id "calc") :click calculate)
    (ev/listen! (dom/by-id "calc") :mouseover add-help)))
```

If you now compile, run and visit [`shopping-dbg.html`][4] page 

```bash
$ lein ring server # from modern-cljs home dir
$ lein cljsbuild auto dev # from modern-cljs home in a new terminal
```

you will see that every time you move your mouse over the `Calculate`
button, a new text saying "Click to calculate" is going be added to
the end of the shopping calculator form.

![Shopping calculator][5]

## Mouseout event

What we now need is a way to delete that text anytime the mouse moves
out of the `Calculate` button. Thankfully, `domina.events` namespace
support `mouseout` event as well.

We need to define a new function, named `remove-help`, which deletes
the DOM node previously added by `add-help` to the form, and then we
need to attach that function to the `mouseout` event of the
`Calculate` button. Here is the complete `shopping.cljs` source file.

```clojure
(ns modern-cljs.shopping
  (:require [domina :as dom]
            [domina.events :as ev]))

(defn calculate []
  (let [quantity (dom/value (dom/by-id "quantity"))
        price (dom/value (dom/by-id "price"))
        tax (dom/value (dom/by-id "tax"))
        discount (dom/value (dom/by-id "discount"))]
    (dom/set-value! (dom/by-id "total") (-> (* quantity price)
                                            (* (+ 1 (/ tax 100)))
                                            (- discount)
                                            (.toFixed 2)))))

(defn add-help []
  (dom/append! (dom/by-id "shoppingForm")
               "<div class='help'>Click to calculate</div>"))

(defn remove-help []
  (dom/destroy! (dom/by-class "help")))

(defn ^:export init []
  (when (and js/document
             (.-getElementById js/document))
    (ev/listen! (dom/by-id "calc") :click calculate)
    (ev/listen! (dom/by-id "calc") :mouseover add-help)
    (ev/listen! (dom/by-id "calc") :mouseout remove-help)))
```

If you have not killed the previous automatic recompilation
command (i.e. `$ lein cljsbuild auto dev`), you just need to reload
the `shopping-dbg.html` page to see the effect of `mouseover/mouseout`
pair of events by moving the mouse cursor in and out of the
`Calculate` button.

# If you are like me

I have to admit to be very bad both in HTML and in CSS coding and I
always prefer to have a professional designer available to do that
job.

If you're like me, you would not want to code any HTML/CSS fragment as
a string like we did when we manipulated the DOM by adding a `div` to
the `shoppingForm` form.

If there is a thing that I don't like about domina library is that it
requires the child/children argument to be passed to `append!` and
other DOM manipulation functions as a string containing a true HTML
fragment, not a CLJ data structure. That's way I searched around to
see if someone else, having my same pain, solved it.

## hiccups

The first CLJS library I found to relieve my pain was
[hiccups][6]. It's just a CLJS port of [hiccup][7] that uses vectors
to represent tags and maps to represent a tag's attrbutes.

Here are some basic examples of hiccups usage:

```clojure
(html [:span {:class "foo"} "bar"])
;; emits "<span class=\"foo\">bar</span>"

(html [:script])
;; emits "<script></script>"

(html [:p])
;; emits "<p/>
```

hiccups also provides a CSS-like shortcut for denoting `id` and
`class` attribute

```clojure
(html [:div#foo.bar.baz "bang"])
;; emits "<div id=\"foo\" class=\"bar baz\">bang</div>"
```

which brings us to solve our problem of converting to CLJ data
structure the `"<div class=/"help/">Click to calculate</div>"` string
we previously passed to `append!` function

```clojure
(html [:div#.help "Click to calculate"])
```

We are now ready to use hiccups by:

* add hiccups to `project.clj` dependencies
* require `hiccups.core` macro namespace
* use hiccups syntax to generate the HTML string to be passed to
  domina `append!` function
  
Here is the complete `project.cljs` which now includes the hiccups
dependency

```clojure
(defproject modern-cljs "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :min-lein-version "2.0.0"

  ;; clojure source code path
  :source-paths ["src/clj"]

  :dependencies [[org.clojure/clojure "1.4.0"]
                 [compojure "1.1.3"]
                 [domina "1.0.0"]
                 [hiccups "0.1.1"]]

  :plugins [[lein-cljsbuild "0.2.10"]
            [lein-ring "0.7.5"]]

  ;; ring tasks configuration
  :ring {:handler modern-cljs.core/handler}

  ;; cljsbuild tasks configuration
  :cljsbuild {:builds
              {:dev
               {;; clojurescript source code path
                :source-path "src/cljs"

                ;; Google Closure Compiler options
                :compiler {;; the name of emitted JS script file
                           :output-to "resources/public/js/modern_dbg.js"

                           ;; minimum optimization
                           :optimizations :whitespace
                           ;; prettyfying emitted JS
                           :pretty-print true}}
               :pre-prod
               {;; same path as above
                :source-path "src/cljs"

                :compiler {;; different JS output name
                           :output-to "resources/public/js/modern_pre.js"

                           ;; simple optimization
                           :optimizations :whitespace}}
               :prod
               {;; same path as above
                :source-path "src/cljs"

                :compiler {;; different JS output name
                           :output-to "resources/public/js/modern.js"

                           ;; advanced optimization
                           :optimizations :advanced}}
               }})
```

And here is the updated `shopping.cljs` source file.

```clojure
(ns modern-cljs.shopping
  (:require-macros [hiccups.core :as h])
  (:require [domina :as dom]
            [domina.events :as ev]))

(defn calculate []
  (let [quantity (dom/value (dom/by-id "quantity"))
        price (dom/value (dom/by-id "price"))
        tax (dom/value (dom/by-id "tax"))
        discount (dom/value (dom/by-id "discount"))]
    (dom/set-value! (dom/by-id "total") (-> (* quantity price)
                                            (* (+ 1 (/ tax 100)))
                                            (- discount)
                                            (.toFixed 2)))))

(defn add-help []
  (dom/append! (dom/by-id "shoppingForm")
               (h/html [:div.help "Click to calculate"])))

(defn remove-help []
  (dom/destroy! (dom/by-class "help")))

(defn ^:export init []
  (when (and js/document
             (.-getElementById js/document))
    (ev/listen! (dom/by-id "calc") :click calculate)
    (ev/listen! (dom/by-id "calc") :mouseover add-help)
    (ev/listen! (dom/by-id "calc") :mouseout remove-help)))
```

> NOTE 2: this is the first time we met macros in CLJS. CLJS macros are
> written in CLojure and are referenced via the `:require-macros`
> keyword in the namespace declaration where the `:as` prefix is
> required. It has to be noted that the code generated by macros must
> target the capability of CLJS (see [Difference from Clojure][8])

We're now happy with what we reached by using domina and hiccups to
make our shopping calculator sample as clojure-ish as possibile. The
only thing that still hurts me is the `.-getElementById` interop call
in the `init` function which can be very easly removed by just using
`aget` like so:

```clojure
(defn ^:export init []
  (when (and js/document
             (aget js/document "getElementById"))
    (ev/listen! (dom/by-id "calc") :click calculate)
    (ev/listen! (dom/by-id "calc") :mouseover add-help)
    (ev/listen! (dom/by-id "calc") :mouseout remove-help)))
```

Update the `shopping.cljs` source file as above, save it, clean and
recompile everytng as usual and finnaly run the project as usual

```bash
$ lein cljsbuild clean # from modern-cljs home dir
$ lein clean
$ lein cljsbuild once
$ lein ring server 
$ lein trampoline cljsbuils repl-listen # from modern home dir in a new terminal
```

And visit [`shopping-dbg.html`][4]. Then visit [`shopping-pre.html`][9] and
[`shopping.html`][10] to verify that all the builds still work.

As homework I suggest you to code modify `login.cljs` accordingly to
what we did for `shopping.cljs` in this and in the
[previous tutorial][1].

# Next step - TO BE DONE

TO BE DONE

# License

Copyright © Mimmo Cosenza, 2012-13. Released under the Eclipse Public
License, the same as Clojure.

[1]: https://github.com/magomimmo/modern-cljs/blob/master/doc/tutorial-08.md
[2]: https://github.com/levand/domina#examples
[3]: https://github.com/magomimmo/domina/blob/master/readme.md#examples
[4]: http://localhost:3000/shopping-dbg.html
[5]: https://raw.github.com/magomimmo/modern-cljs/master/doc/images/mouseover.png
[6]: https://github.com/teropa/hiccups
[7]: https://github.com/weavejester/hiccup
[8]: https://github.com/clojure/clojurescript/wiki/Differences-from-Clojure
[9]: http://localhost:3000/shopping-pre.html
[10]: http://localhost:3000/shopping.html