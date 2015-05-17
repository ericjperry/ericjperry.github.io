---
title: Change port in sbt for Scalatra app
author: eric
layout: post
permalink: /change-port-in-sbt-for-scalatra-app/
categories:
  - Uncategorized
---
I&#8217;ve been diving into the Scala(tra) world as part of a new project at work, and I wanted to share a useful piece of information here that I had trouble finding on the Interwebs. My task involves running two different Scalatra apps that are talking to each other, so of course they need to run on different ports when bringing them up locally. I am using sbt as the build manager for this app, and my build settings are all contained in project/build.scala, but I couldn&#8217;t figure out where to put the port information. Alas, after I created a **build.sbt** file in the root folder for the app and added the port information everything worked out perfectly. The following line in build.sbt will set the container&#8217;s port to 3002 when running `./sbt container:start`

<pre class="brush: scala; title: ; notranslate" title="">port in container.Configuration := 3004
</pre>
