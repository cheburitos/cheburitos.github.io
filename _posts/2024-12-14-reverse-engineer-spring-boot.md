## Reverse engineer Spring Web in 5 minutes using a recording debugger

Large Java frameworks can be reverse engineered with help of some tools. 
The article shows how to dig into Spring Web internals with help of [ulyp](https://github.com/0xaa4eb/ulyp) 
in less than 10 minutes. A developer can observe how framework works inside before diving into source code. 
This is a part 2 of the ongoing blog series about ulyp. 
The first part is available [here](https://0xaa4eb.github.io/2024/10/13/recording-java-code-execution-for-faster-debugging.html).

---

### Introduction

When working with gigantic frameworks, the main issue is to analyze their codebase. 
Sometimes, a developer needs to get understanding of internals real quick. This can happen when something doesn't work as expected,
yet basic debugging doesn't help much.

Some of the ways to get into the codebase:
* Use plain old debugger. Stop the app at some breakpoint and check the stack trace and surrounding types.
* Dive right to the source code with the obtained knowledge.
* Start from tests. Tests can be quite helpful if one wants to understand the codebase.
* Enable `TRACE` logs and try to gather some info from them.
* Use a sampling profiler. Building a flamegraph could be quite a good idea to start from. This can be quite useful if the app being analyzed is already under load.
* Use a recording debugger, which records the whole execution flow and then can show the call tree.

The article focuses on the last approach. In certain cases, using such a tool might be useful for a anybody who works with large codebase. 
Recorded execution flow can be used to get the idea how things work at runtime. A developer can then dive 
into source code with good understanding.

### Reverse engineering Spring Web

We will see how the tool works on Spring Web based app. The project which we are going to 
use is [spring-petclinic](https://github.com/spring-petclinic/spring-petclinic-rest). This is a basic web app written on Java with Spring Web. 
Suppose, we want to understand how it works. And we want to do it really quick.

We will skip the setup and jump right to recording. The first thing we want to know is where to start recording. 
This can be done by plenty of ways, but the easiest way is to gather some stack traces. If we stop inside the controller
we see the top of the stacktrace is

```
    <app methods>
    ...
    <other spring methods>
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:201)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116)
	...
    <tomcat methods>
	at java.base/java.lang.Thread.run(Thread.java:1583)
```

Since we want to reverse engineer Spring Web, we'd like to start from the bottom of the stacktrace. This is `org.springframework.web.filter.OncePerRequestFilter`
in our case. It's a first called method of Spring framework.

If we want to record execution, we should configure ulyp first. We specify the following system properties when start petclinic app, 
no code change is required.

```
-javaagent:/home/tools/ulyp/ulyp-agent-1.0.2.jar
-Dulyp.methods=**.OncePerRequestFilter.doFilter
-Dulyp.file=/tmp/test-petclinic.dat
-Dulyp.record-constructors
-Dulyp.record-collections=JDK
-Dulyp.record-arrays
```

Specifying `-Dulyp.methods=**.OncePerRequestFilter.doFilter` makes ulyp start recording whenever `doFilter` method of 
any class named `OncePerRequestFilter` is called. `-Dulyp.file=/tmp/test-petclinic.dat` specifies the output filepath. 
The other props enable additional recording features which is optional.

We open the UI at url `http://localhost:4200/petclinic/vets` which lists all veterinarians. We stop the app after, since ulyp 
writes everything to the output file immediately.

![petclinic app](https://0xaa4eb.github.io/assets/2024/12-14/petclinic1.png)

Now, we want to see what's recorded. ulyp comes with the desktop app which can open recording files. If we
open `/tmp/test-petclinic.dat` file, we can observe there is indeed some recorded data.

![recorded data](https://0xaa4eb.github.io/assets/2024/12-14/petclinic2.png)

What we see is a **recording call tree**. The top method is `OncePerRequestFilter.doFilter` as we expected. The next level are
methods which were called inside the method call (basically, children). Black background line hints how many calls are inside every subtree.
We can unfold every method call which has nested calls recorded. If we unfold further down, we can get the overall idea. 
Spring Web implements [chain of responsibility](https://en.wikipedia.org/wiki/Chain-of-responsibility_pattern) pattern. So, 
every filter does some work and then calls the next filter. Here we see that `OrderedFormContentFilter` calls 
`OrderedRequestContextFilter`.

![MVC filters](https://0xaa4eb.github.io/assets/2024/12-14/petclinic3.png)

You can read more about Servlet based application architecture [here](https://docs.spring.io/spring-security/reference/servlet/architecture.html).

If we dig down, we can observe how Spring Web creates [micrometer](https://github.com/micrometer-metrics/micrometer) events. 
Other stuff is updating context which can be accessed from other filters.

![micrometer events](https://0xaa4eb.github.io/assets/2024/12-14/petclinic4.png)

There are also many filters which are called on the path. The next interesting one is `AuthorizationFilter`. We can 
see how it calls `ObservationAuthorizationManager.check` method, which returns an instance of `AuthorizationDecision`. 
The filter later checks that `AuthorizationDecision.isGranted` returns `true`, which means the user has the access.

![AuthorizationFilter](https://0xaa4eb.github.io/assets/2024/12-14/petclinic5.png)

Next, we can see how `DispatcherServlet.doDispatch` is called, which does some logic, and finally we see our controller is called. 
The `VetController` returned a list of `Vet` objects.

![VetController working](https://0xaa4eb.github.io/assets/2024/12-14/petclinic6.png)

Basically, we can check how Spring MVC works, calls filters, validators, maps response into json, etc. 
Every method is recorded which later can be analyzed.

If we wanted it, we could dig down to Hibernate or even JDBC level, but that's not the point of the article. There is 
another [article](https://0xaa4eb.github.io/2024/10/13/recording-java-code-execution-for-faster-debugging.html) which shows how ulyp handles Hibernate.

### Conclusion

The article shown how a developer can record and analyze the internals of large framework. The obtained knowledge can 
help in development. Feel free to try [ulyp](https://github.com/0xaa4eb/ulyp) while working on a huge codebase.