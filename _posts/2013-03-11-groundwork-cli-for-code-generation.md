---
layout: post
title: Groundwork CLI for code generation
date: 2013-03-11 02:30:00
---

Recently I've been playing around with [yeoman](http://yeoman.io/) using the angular generators, and it's quite handy to have new services generated with a jasmine spec file ready to go. It really shines when I've got some momentum and I don't want to lose it during the mundane task of creating new files.

I thought it would be nice to have one of these tools in PHP, so to avoid doing the more important things I had planned today I cobbled together version 1.0.0 using Symfony Console. It can be installed using composer from the [github project here](https://github.com/dancras/groundwork).

Simply install, add a .groundwork folder to your project, copy [my class template from here](https://github.com/dancras/groundwork/tree/master/.groundwork), tweak it to suit your needs, and run the create command:

    vendor/bin/groundwork create class "Vendor\Project\MyAwesomeClass"

The template linked above will create a class file and a phpunit test case to PSR-0 conventions, with a dummy test passing so you can get started right away.

Going forward I'll probably investigate yeoman further as it is a more mature tool solving the same problem, however not all php projects have (or want) nodejs, npm, grunt, yeoman and it's plethora of dependencies.
