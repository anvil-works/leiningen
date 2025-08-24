# Managed Dependencies With Leiningen

Managed dependencies allow you to specify overrides for dependencies that you
may encounter into your dependency tree, in order to ensure consistency.

There are two main reasons you may want to do this. The main one is that your
project's dependency tree might have some dependencies in it that are specified
as ranges. Ranges are antithetical to repeatability. Today you might get one
result from a range, and tomorrow after a new release happens, your result will
change. This is absolutely unacceptable for a tool that strives to provide
consistency, but the dependency resolving library we use unfortunately has this
behavior anyway. Managed dependencies allow us to work around this design flaw
by locking in a specific version in a way that will override whatever ranges
you might encounter.

The other reason managed dependencies are useful is that they allow you to lock
in consistent dependency sets across several projects.

## `:managed-dependencies`

The `:managed-dependencies` section of your `project.clj` file is just like the
regular `:dependencies` section, with two exceptions:

1. It does not actually introduce any dependencies to your project.  It only says,
  "hey leiningen, if you encounter one of these dependencies later, here are the
  versions that you should use instead of whatever is specified."
2. It allows the version number to be omitted from the `:dependencies` section,
  for any artifact that you've listed in your `:managed-dependencies` section.
  Directly declared versions in `:dependencies` will take precedence over
  managed version numbers, but versions that are pulled in transitively will not.

Here's an example:

```clj
(defproject superfun/happyslide "1.0.0-SNAPSHOT"
  :description "A Clojure project with managed dependencies"
  :min-lein-version  "2.7.0"
  :managed-dependencies [[clj-time "0.12.0"]
                         [me.raynes/fs "1.4.6"]
                         [ring/ring-codec "1.0.1"]]
  :dependencies [[clj-time]
                 [me.raynes/fs]])
```

In the example above, the final, resolved project will end up using the specified
 versions of `clj-time` and `me.raynes/fs`.  It will not have an actual dependency
 on `ring/ring-codec` at all, since that is not mentioned in the "real" `:dependencies`
 section.

In the absence of ranges, this feature is not all that useful on its own,
because in the example above, we're specifying the `:managed-dependencies` and
`:dependencies` sections right alongside one another, and you could just as
easily include the version numbers directly in the `:dependencies` section.
The feature becomes more powerful when your build workflow includes some other
way of sharing the `:managed-dependencies` section across multiple projects.

## A note on modifiers (`:exclusions`, `:classifier`, etc.)

The managed dependencies support in leiningen *does* work with modifiers such as
`:exclusions` and `:classifier`.  There are two legal syntaxes; you can explicitly
specify a `nil` for the version string, or you can simply omit the version string:

```clj
(defproject superfun/happyslide "1.0.0-SNAPSHOT"
  :description "A Clojure project with managed dependencies"
  :min-lein-version  "2.7.0"
  :managed-dependencies [[clj-time "0.12.0"]]
  :dependencies [[clj-time :exclusions [foo]]])
```

or

```clj
(defproject superfun/happyslide "1.0.0-SNAPSHOT"
  :description "A Clojure project with managed dependencies"
  :min-lein-version  "2.7.0"
  :managed-dependencies [[clj-time "0.12.0"]]
  :dependencies [[clj-time nil :exclusions [foo]]])
```

Note that `:classifier` is actually a part of the maven coordinates for an
artifact, so for `:classifier` artifacts you will need to specify the `:classifier`
value in both the `:managed-dependencies` and the normal `:dependencies` section:


```clj
(defproject superfun/happyslide "1.0.0-SNAPSHOT"
  :description "A Clojure project with managed dependencies"
  :min-lein-version  "2.7.0"
  :managed-dependencies [[commons-math "1.2" :classifier "sources"]]
  :dependencies [[commons-math :classifier "sources"]])
```

## Lein "parent" projects

One way of leveraging `:managed-dependencies` across multiple projects is to use
the [`lein-parent` plugin](https://github.com/achin/lein-parent).  This plugin
will allow you to define a single "parent" project that is inherited by multiple
"child" projects; e.g.:

```clj
(defproject superfun/myparent "1.0.0"
   :managed-dependencies [[clj-time "0.12.0"]
                            [me.raynes/fs "1.4.6"]
                            [ring/ring-codec "1.0.1"]])

(defproject superfun/kid-a "1.0.0-SNAPSHOT"
   :parent-project [:coords [superfun/myparent "1.0.0"]
                    :inherit [:managed-dependencies]]
   :dependencies [[clj-time]
                  [me.raynes/fs]])

(defproject superfun/kid-b "1.0.0-SNAPSHOT"
 :parent-project [:coords [superfun/myparent "1.0.0"]
                  :inherit [:managed-dependencies]]
 :dependencies [[clj-time]
                [ring/ring-codec]])
```

In this example, we've consolidated the task of managing common version dependencies
in the parent project, and defined two child projects that will inherit those
dependency versions from the parent without needing to specify them explicitly.

This makes it easier to ensure that all of your projects are using the same versions
of your common dependencies, which can help make sure that your uberjar builds are
more predictable and repeatable.

## Other ways to share 'managed-dependencies'

Since the `defproject` form is a macro, it would be possible to write other plugins
that generated the value for a `:managed-dependencies` section dynamically.  That
could provide other useful ways to take advantage of the `:managed-dependencies`
functionality without needing to explicitly populate that section in all of your
`project.clj` files.
