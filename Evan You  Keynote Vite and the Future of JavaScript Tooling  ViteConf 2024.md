# Evan You | Keynote: Vite and the Future of JavaScript Tooling | ViteConf 2024

- Hello everyone and welcome back to another ViteConf.  
I'm Evan You, creator of Vite,  
and I'm really excited to have you here with us online again  
and I've got some big news to share with you,  
and it's about Vite and the future of JavaScript tooling.  
So let's get right into it.  
First of all, a customary look at the MPM downloads.  
We've passed 15 million weekly downloads.  
That's more than double compared to last year's ViteConf.  
And just a few days ago, we had a single day  
with more than 3 million downloads.  
Pretty amazing.  
On the other hand, Vitest is quickly replacing Jest  
as the go-to JavaScript test runner,  
crossing 6 million weekly downloads of its own.  
In the latest State of JS survey,  
Vite and Vitest topped the charts  
in almost every applicable category,  
most adopted, highest retention, highest interest,  
and most loved library overall.  
I was really pleasantly surprised when the results came out,  
so thank you to all Vite users for the love.  
And we have a vast ecosystem that keeps growing.  
Vite is now the default tooling  
for most frameworks when building an SPA,  
and powers almost every JavaScript meta-framework  
except XJS.  
Vite-based frameworks are now powering some  
of the most high-profile applications  
and websites in the world, ranging from OpenAI's ChatGPT  
to Shopify, Reddit, Porsche, Linear, and many more.  
It also has first-class integration  
with tools like Storybook  
and backend frameworks like Laravel.  
The significance of this convergence, in my opinion,  
is that frameworks building together  
on top of Vite can share a strong interop story.  
Not only can they avoid reinventing the wheels,  
assembling a custom build stack,  
they can also share plugins, test runners,  
and even deployment adapters, allowing each framework  
to focus on innovation in areas that really matter.  
Saying this last year might have been a bit premature,  
but today, I think it's fair to claim  
that Vite has become the shared infrastructure layer  
for the next generation of web applications.  
With all that said, I would be the first one to acknowledge  
that Vite isn't perfect.  
The trust that the JavaScript community has placed  
on Vite has pushed me to think deeply about its flaws,  
its future, and what we should do about it.  
I came to a pretty ambitious conclusion,  
and I want to go through my thought process with you today.  
Let's start with why people love Vite.  
People love Vite  
because it makes web development feel simple again.  
Now that, this implies at one point,  
everyone felt web development was becoming too complicated,  
and it's true.  
Vite is hiding a lot of complexity  
from its end users under the hood.  
Internally, there are many things we wish we could simplify,  
change, or improve.  
We're only able to handle all that complexity thanks  
to the underlying tools that Vite is built on top of.  
Mostly esbuild, Rollup, and SWC.  
esbuild is used for dependency bundling,  
transforming TypeScript, JSX.  
It's also used for target lowering,  
syntax lowering transforms,  
and minification in the production builds by default.  
Now, Rollup is used for production bundling,  
and we use a Rollup-compatible plugin system.  
SWC is not really needed by default,  
but if you're building a React app  
and want faster React refresh transforms than Babel's,  
you should be using the SWC-based React plugin  
that we provide.  
So when you're using that, you're effectively going  
to be using all three of these dependencies through Vite  
in one single application.  
Now, this architecture works, right?  
It's been working pretty well for a lot of you,  
but, and we are grateful that these tools allow Vite  
to delegate all these hard problems to them,  
but it has very glaring problems,  
and let's go through them one by one.  
First, I need to explain why we need two bundlers  
in the first place.  
We use esbuild, which is blazing fast,  
but its tree shaking and code splitting are not as complete  
and configurable as Rollup's.  
And its plugin system is a bit too rigid compared  
to Rollup's.  
So we only use it to bundle dependencies during development.  
This is called the dep pre-bundling  
or the dep optimizer, as we know it.  
By default, we also use esbuild as a transformer  
to transform TypeScript, perform syntax lowering  
as I just mentioned, and minify chunks during production.  
We use Rollup for the production build  
because it is better suited  
for bundling applications compared to esbuild,  
given that it has better chunk control  
and a bit more flexibility  
in configuring the tree shaking behavior.  
However, Rollup is written in JavaScript.  
While the bundling performance is better than webpack's,  
it is much slower compared to bundlers written  
in a native language, for example, esbuild.  
And for SWC, one of the reasons we don't really include SWC  
in Vite by default is because it's heavy binary size.  
On MacOS, the binary is 37 megabytes  
and that's more than twice the size of Vite  
and all these dependencies combined.  
And while SWC has comprehensive transforms  
and a high-quality minifier,  
it doesn't really offer a usable bundler.  
So even if we include SWC,  
we still need to use another bundler  
to cover the other parts.  
So we are unfortunately stuck  
with this multi-dependency scenario.  
The first problem  
that this leads to is behavior inconsistency.  
There can be difference in the ways the esbuild  
and Rollup handle mixed ESM and CJS module wraps.  
And sometimes, this can lead to subtle bugs  
that can only be discovered  
after production build goes live.  
Not good.  
Second is the massive inefficiency in the build pipeline.  
The same piece of source code  
in a Vite production build gets repeatedly parsed into AST,  
transformed, and serialized back into strings  
by these different tools.  
And worse, they're often passed to back  
and forth between native side, for example,  
the Go process, and JavaScript main thread  
and then pass to Rust and then pass back to JavaScript.  
So all these passing these large chunks  
of data across processes also has a lot of overhead,  
and it gets even worse when source maps are enabled  
because every step,  
individual step between these tools requires remapping  
of the source maps and merging them.  
Finally, the unbundled native ESM dev server works great,  
but only up to a certain threshold.  
The network overhead when you need to load tens of thousands  
of unbundled modules over HTTP really adds up,  
and it can take seconds for a page  
to load during development.  
While probably 90% of apps we're building are not this big,  
but the development experience  
for that other 10% becomes problematic compared  
to a fully bundled approach.  
Unfortunately, in the current model,  
we don't have a good solution for this.  
We wish we could use a bundler  
to do full bundling during development too,  
but esbuild is incompatible with our plugin system  
and will result  
in even more development production inconsistencies,  
while using Rollup will just simply make it too slow.  
So sounds like we need to unify the bundler used in Vite  
by building one ourselves.  
And that is exactly why we started Rolldown.  
But before we dig into Rolldown, I thought a bit more,  
as we worked on Rolldown, I dig deeper into the idea.  
I found that even the abstractions  
below the bundle layer have similar problems.  
So why do we stop at the bundler?  
I realized  
that the challenge Vite is facing is really a reflection  
of the JavaScript ecosystem at large.  
JavaScript grew from a simple scripting language  
to literally the most widely used language  
in the world today.  
During this process, the JavaScript community created many,  
many, many different tools  
to bridge the gap between its lack of official tooling  
and the rapid growth in the scale  
and complexity of its use cases.  
We're building more and more ambitious applications  
and we had to create all these tools just to scramble  
to keep up with the demand  
of all these engineering best practices  
and newfound complexity and the requirement to have types,  
the requirement to bundle, minify, and everything.  
We discovered these problems along the way  
and created tools to solve them along the way.  
This is a blessing and a curse,  
because the JavaScript ecosystem  
is probably the most vibrant community out there.  
There are so many innovations,  
so many great tools that came out.  
It did give us all the tools we needed.  
But it also gave us a new set of problems,  
that is fragmentation, incompatibilities, inefficiencies,  
and in many cases, frustration.  
My conclusion is that to eliminate the inconsistency  
and inefficiency issues, a unified toolchain  
for JavaScript is needed.  
It starts at the parser and includes semantic analysis,  
transformer, linter, formatter, minifier, bundler,  
all the way up to the common layer  
of abstraction that supports the final frameworks.  
And that common layer is Vite.  
And this is also what I believe this toolchain should be.  
Unified, high performance, composable,  
and runtime agnostic.  
First unified.  
All the different tasks involved in the process  
of turning source code  
into final build artifacts should be performed  
by the same set of tools using the same AST,  
with the same consistent rules  
for configuration, module format interop,  
and path resolution.  
Second, high performance.  
For well-defined tasks like transforming TypeScript  
or bundling code, the tool should be written  
in a compiled-to-native language  
with an obsession for performance at every possible chance.  
Given JavaScript's scale today,  
performance improvements in our tooling is a huge leverage  
that speeds up not only dev cycles, CI builds,  
product delivery, and ultimately saves money for everyone.  
Third, composable.  
The toolchain should be composable.  
Each component  
of the toolchain should be individually consumable  
as a dependency.  
For example, the parser, the resolver,  
and the transformer should all be usable as Rust crates  
or MPM packages.  
There also should be efficient Rust, JS interop  
so that users, while leveraging these native tooling,  
can still write JavaScript plugins to fit their needs.  
And finally, the core  
of the toolchain should be runtime agnostic.  
Developers trying to leverage these tools or just a part  
of the toolchain should not be forced  
to lock themselves into a specific JavaScript runtime.  
We want the same development experience,  
no matter what runtime  
or target environment you are targeting,  
as long as you're writing JavaScript.  
Now, you might be thinking,  
isn't this just way too ambitious?  
Well, I used to think so too.  
Some other folks have even tried and failed.  
But think about it.  
Four years ago, who would have believed  
that Vite would become the convergence point  
for JavaScript frameworks as it is today?  
Well, I believe if we have the right people,  
the right execution, the right leverage,  
and enough resources, this goal,  
despite it being very ambitious, is indeed possible.  
So this is why I started a company called Void Zero,  
a company building the next generation  
of JavaScript tooling.  
We have raised 4.6 million seed funding, led by Accel  
with participation from Amplify Partners,  
and many experienced founders in the dev tools space.  
We have built a full-time team that is working hard  
to realize this vision.  
Now, here is a big picture overview  
of what we are building at Void Zero.  
Oxc will be the foundational language toolchain  
that supports everything.  
It includes the parser, semantic analysis,  
transformer, minifier, resolver, linter, and formatter.  
Rolldown is the bundler built on top of Oxc  
and will be the unified bundler powering a future version  
of Vite.  
Through Vite and Vitest, we will be moving all frameworks  
and tooling depending on them to a unified foundation.  
So let's talk about the progress so far.  
This is the the progress chart for Oxc.  
For Oxc, we have already completed the parser, linter,  
and resolver.  
Boshen, Oxc's project lead,  
will actually talk more about this in more details,  
so I'll be a bit brief here.  
For the transformer, we've already completed TypeScript,  
JSX, we had refresh transforms  
and isolated declarations for TypeScript DTS emit.  
Our current focus is on finishing the transformer  
with syntax lowering transforms.  
The minifier and formatter are in working prototype state,  
and will be worked on after the transformer is done.  
And now, how fast is Oxc?  
So this is the meaty part.  
To the best of our knowledge, we're proud to say  
that it's probably the fastest  
in every comparable category out there so far.  
It has the fastest parser, the fastest linter,  
the fastest resolver, and the fastest transformer,  
all up to three times faster  
than other Rust-written solutions.  
I have linked the benchmarks  
and you can check them out yourself  
after the talk via my shared slides.  
There are some other often overlooked metrics I'd also like  
to mention about Oxc.  
First of all, it uses much less memory than,  
for example, Babel.  
It also uses less memory than SWC.  
And it also shifts significantly smaller binaries compared  
to other solutions.  
Remember I mentioned the reason we didn't want  
to include SWC by default in Vite was because  
of the binary size.  
So Oxc Transform, when used as a dependent MPM dependency,  
downloads the respective binary for the target platform,  
and on MacOS, the size is only 1.95 megabytes,  
less than two megabytes,  
and that is, well,  
almost 20 times smaller than SWC size, right?  
And if you use Oxc as a Rust dependency, as a Rust crate,  
it also compiles much faster  
because it doesn't rely heavily on macros.  
So that also improves your development experience  
if you're using Oxc as a Rust developer.  
Now, onto Rolldown.  
For Rolldown, we have already completed most  
of the features you would expect from a bundler.  
Notably, CJS and ESM interop, source maps, output formats,  
basic code splittings, CLI,  
config file support and all that.  
Typical things you'd expect from a bundler.  
But on top of that, we also have Oxc integration,  
which means Rolldown actually ships  
with built-in TypeScript and JSX transforms,  
built-in Node-compatible resolving, built-in alias handling,  
and everything that you expect  
from a fully featured resolver,  
and it also has finished some advanced features  
such as production-quality tree shaking,  
advanced chunk splitting options,  
which is actually more powerful than Rollups already,  
and we have reached 90% compatibility with Rollup plugins.  
Only some niche options and properties are not supported,  
and it can already run a good chunk  
of official Rollup plugins,  
unless it relies on some specific properties  
that we do not support.  
So right now, we are focusing on two things for Rolldown.  
The first is making Rolldown a general purpose bundler  
that has first-class support for CSS, HTML, and assets.  
And the other is testing its integrations into Vite,  
replacing esbuild and Rollup,  
and porting some of the Vite internal plugins over to Rust  
to reduce overhead.  
This work in progress Rolldown-powered Vite,  
which we currently just called it Rolldown Vite,  
currently passes 98% of dev tests and 82% of build tests.  
So we're on a very good trajectory  
to complete the test coverage and then start releasing it  
in alpha state.  
Once we finish these two,  
there will be some more behavior alignment,  
performance tuning,  
and code quality polish before we release Rolldown  
as a 1.0 beta.  
To get an idea of the performance of Rolldown,  
we used a benchmark forked from another bundler,  
and we increased the number  
of components the benchmark contains.  
So in our version of the benchmark,  
we're bundling 19,000 modules, where 10,000  
of them are React JSX components  
and 9,000 are iconify JS files.  
Now, Rolldown completes the bundle in 0.63 seconds.  
That is almost two times faster than esbuild,  
and three times faster than farm  
and five times faster than rsbuild.  
Note that we don't even include JavaScript bundlers  
in this benchmark anymore  
because the bar would literally go out of the screen.  
And if we try to bundle this application  
with the current Rollup-powered Vite,  
it actually just runs out of memory and never finishes.  
Another case study  
that is a bit more real-world is using Rolldown and Oxc  
to bundle Vue.  
Now, Vue's codebase is a pretty non-trivial codebase.  
It's a TypeScript monorepo with 11 packages,  
and 62 dist bundles.  
There are cross-dependencies between these packages.  
We use TypeScript everywhere,  
and they are pretty complicated  
cross-package dependency graph  
and some pretty custom prebuilt steps that we have.  
So we actually, being able to migrate over  
from our previous Rollup-based build setup over  
to Rolldown actually simplified the config by quite a bit  
because there are multiple plugins we were able to remove  
that are supported just as built-in features by Rolldown.  
Now, notably, the bundle packages  
and TypeScript declaration, now using Rolldown  
and Oxc's isolated declarations DTS emit transform  
are passing not only Vue's own tests,  
but also Vue's Ecosystem CI tests,  
where the built artifacts are tested in the real build  
and test in the real build and test flows  
of a dozen downstream Vue ecosystem projects.  
So this is a good testament  
for the production readiness of Rolldown and Oxc in general.  
Although we don't really consider them production ready yet,  
we consider being able to pass Vue ecosystem tests a very,  
very good milestone that we can use  
to verify how ready it is.  
Now, because successfully passing all  
of these tests required correctness in every aspect  
of the pipeline, including transforms, platform,  
and format output, plugin compatibility,  
also, the pre-minify bundle size are matching that  
of the output of Rollup,  
which means there is comparable tree shaking  
and dead code elimination quality.  
And of course, the performance.  
Here, we are comparing three versions of Vue,  
actually three commits at different times.  
3.2 were about 19 months ago when all the tooling  
that we used were written in JavaScript.  
And it was also inefficient  
because we were essentially generating DTS  
for each package individually.  
Now, in 3.5, the current main branch,  
actually, probably in 3.4, we did a major build refactor.  
We migrated over to esbuild using Rollup plugin esbuild  
to the TypeScript transforms,  
we are using SWC for minification,  
and we moved over to Rollup plugin DTS for DTS bundling.  
So that was actually a big run optimization,  
but the main bundling was still using Rollup  
because it was the only one that was able  
to retain the same bundle size in the production builds,  
due to the tree shaking requirements.  
So in Vue 3.5 main branch,  
the current full build time including types is 5.8, 5.,  
8.5 seconds, sorry, compared to the 3.2, close two minutes,  
that's already a pretty big improvement.  
But now, after migrating to Rolldown and Oxc transform  
for the DTS emit,  
the whole build takes a little bit more than one second.  
And this is not even the most optimal form yet  
because right now, we are doing the DTS emit  
and bundling in a separate process in the second step.  
In the future, Rolldown will include the DTS emit  
and bundling as built-in feature  
that's going to be performed in parallel  
to the source code bundling.  
So when that happens, the whole build process  
for Vue will be well below one second.  
So that is the progress so far, and what's next?  
So first of all, let's take a look at Vite today.  
This is Vite today.  
This is showing the architecture of the upcoming v6  
with the Environment API merged.  
It still relies on esbuild, it still relies on Rollup,  
and if you're using React, you want fast refresh, transform,  
and you don't want to use Babel,  
you will still be using SWC, right?  
So this is Vite today,  
and in the worst case scenario,  
you'll be using three different tools  
that's wrapped inside Vite.  
This is the next evolution of Vite,  
and its current work in progress,  
as will be known as Rolldown Vite.  
It is powered by Rolldown and Oxc internally.  
It will greatly improve the dev product consistency  
because now there's just one toolchain  
that handles both development and production bundling.  
It will reduce the internal overhead  
because there's just less code passing  
between different tools,  
and it'll greatly improve the production build performance  
because Rolldown is written in Rust and splitting fast.  
So this is going to be the version  
that we'll be focusing on right now  
and we hope to get into your hands next year,  
early next year.  
And in the longer future, we will likely ship a version  
of Vite that dedicates even more to Rolldown.  
And we use the same bundling capabilities of Rolldown  
for development, Environment ModuleRunner, and production.  
So this allows us to get rid  
of the unbundled network bottleneck  
for extremely large applications.  
It'll also ensure maximum consistency  
across browser environment,  
simulated Node.js server rendering or worker environments,  
and also for production builds.  
It'll provide the best possible performance  
in all of these scenarios  
and also gives you a consistent toolchain  
that handles all the tasks together.  
And this is our long-term goal.  
Currently, it's in prototype stage  
because the first thing we wanna do is to make sure  
that the full bundle mode can actually work  
and we can actually also have HMR  
on top of a full bundle emitted by Rolldown.  
And the full bundle mode performance prototype is, again,  
very promising.  
From the benchmark,  
using the same React JSX-based mock application, we can see  
that the page load is more than five times faster compared  
to today's Vite, compared to the unbundled version of Vite.  
And the HMR is also close to instant  
with some optimizations on the watcher side.  
So that said, this is still early.  
Our current priority  
is to ensure the existing Vite ecosystem  
can smoothly start benefiting from our work on Rolldown  
and Oxc, and this process is going to take some time.  
We'll be working on this future version of Vite in parallel,  
and we hope to also be able  
to show you a more stable version of it later next year.  
Well, that's it.  
I hope this gets you excited about the future of Vite  
and JavaScript.  
We wouldn't have had the courage  
to take on this challenge without the trust  
and support from the Vite community,  
and we hope this work will make us all more productive  
and result in more great apps being built faster.  
Thank you.  
(upbeat funky pop music)  
