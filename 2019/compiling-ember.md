# Compiling Ember

# Abstract

Compilers have a reputation for being esoteric and intimidating. But they don't need to be! A compiler is just a program that writes programs. This talk will be a practical tour through the Embroider build system that also teaches compiler concepts along the way.

With the power of multi-pass compilation, Embroider can take a long-lived, conventional Ember app and give it lazy loading, code splitting, and tree shaking. All without modifying the app's own code.

# Essay

Compilers have a reputation for being esoteric and intimidating. If you studied computing at a university, the compiler course was probably considered "one of the hard ones". And for those who entered web development directly through more direct training like bootcamps or self-study, compilers maybe didn't come up at all.

But compilers don't need to be scary! It's true that writing a great compiler from scratch is a complicated job, but we can go quite far just by understanding what compilers are, what they're capable of, what they're not capable of, and how even simpler compilers can do important jobs.

The word "compiler" was coined by Grace Hopper around 1950, and by 1952 she had written the first one.

[quote]

She had to go into excruciating detail to convince people it was even possible or desirable to author programs in "high level" ways, and had to invent a lot of things we take for granted.

[show fig5 from hopper's "The education of a computer"]

Here she is trying to explain the division of labor between the programmer and the computer, and how compilers would radically alter it.

Before we go further, we should define "compiler".

> A compiler is a program that...

Wait, let's just be really sure first. What is a program?

(the part is cuttable) Most of us rarely sit down and "write a program", after all. The daily work of web development is mostly making small changes one at a time within a larger program. The boundaries of "what is a program" depend on your frame of reference. From the perspective of your operating system, you probably spend all day making tiny adjustments inside the program "Chrome". From the perspecitve of your web browser, your Ember app constitute a program that begins executing each time you load the page.

A program is just a set of instructions for taking some input and producing some output.

[diagram: program with I/O]

What's the input and output of an Ember app? As it starts up, the input is a URL. The output is HTML (really DOM, for concerned performance nerds, but "HTML" is a true enough understanding). Then as it continues to run, the input is events from the user (like clicks, scrolls, or changing the URL) and events from the network (like some data finally becoming available, or a message arriving on a websocket).

So, back to "what is a compiler?"

A compiler is a program that takes a program as input and outputs a program.

[diagram: analogous to program with i/o, but showing program in all the boxes]

Of course, a compiler that only knows how to handle a single *particular* program as input wouldn't be very interesting, so when we say the input is a program, we mean the input is *any program* in a *particular language*. And a compiler that doesn't emit a consistent language as its output also wouldn't be very useful, so we can assume the output language is constant. So we can get more precise and say

A compiler is a program (written in language A) whose input is a program (written in language B) and whose output is also a program (written in language C).

[updated diagram with language labels]

This is an expansive definition. It's possible to quibble with my definition, and some people distinguish between several kinds of compilers:

- "true" compiler: language B is easier for humans to read, whereas language C is easier for computers to run
- decompiler: language B is easier for computers to run, while language C is easier for humans to read
- transpiler: language B and language C are nearly the same language
- "bootstrapping" or "self-hosting compiler": language A and language B are nearly the same language.

I mention this so you know how the word "transpiler" fits in. Is ECMAScript 2018 the same language as ECMAScript 5? If you say yes, then babel is "only" a transpiler. If you say no, babel is a "true compiler". But this distinction is not very interesting, so for the rest of this talk we're going to just say "compiler". Babel, by the way is also a self-hosting compiler. It's own source code runs through Babel to build itself.

Now that we have a solid foundation in what a compiler is, let's talk about what it can and cannot do.

Compiling is not evaluating. Remember that every program has input and output:

[diagram repeats from earlier]

And remember that a compiler's input and output are programs:

[diagram repeats from earlier]

But that means the input and output programs also have input and output:

[expanded diagram]

The first thing to notice here is that the compiler can't know what the input will be. The output program needs to work for all possible inputs. This is why a compiler is not an interpreter, and it can't evaluate (aka run) the program. This is what we mean when we say the compiler has only *static* knowledge of the program, not *dynamic* knowledge. These words, static & dynamic, get overloaded to mean many different things in programming, because they're pretty useful general purpose words. Just remember for this talk, they mean:

  static: things you can know about a program before you pick an input and run it
  dynamic: things you cannot know about a program before you pick an input and run it

Second, if our compiler works correctly, it didn't change the *meaning* of the input program. The input program and the output program have the same meaning, so for the same input they produce the same output.

[diagram coalesces the two inputs two one box, and same for outputs]

We say they have the same *semantics*. This makes the compiler's job more clear. It can't change the semantics. It can make the program faster, or smaller, or able to run on new hardware, or able to run in new environments. But it can't change the meaning of the program.

Hiding in this diagram is one of the key messages I want you to take away from this talk. You see, for a lot of languages, this arrow (from the input program to its output) is not actually implemented. The *only* way to run the input program and get an output is to compile the input program, get the output program and run the output program.

[highlight the various parts of the diagram to go along with the words in the previous paragraph. Maybe with remote control, or maybe in the actual slideshow.]

If you have ever written Java, or C, or Rust, you're familiar with the idea of a program that can't be run until after it's compiled.

[substitute specifics into digram, like "C code" and "Mach-O 64-bit executable x86_64"]

But it's also true for all your Ember apps. The raw files that you author and the addons you install from NPM aren't something you can run -- they need to go through our compiler -- ember-cli -- first.

[substitute specifics into diagram, like "app/components/hello-world.js" & "app/templates/components/hello-world.hbs" and "app.js" & "vendor.js"]

In situations where there's no implementation that can run the input program directly, you may be tempted to make a mistake. Because the "output of the input program" is imaginary, you ignore it. You don't carefully define it, and you start to think of the semantics of your language as "whatever the compiler does with it".

[contrast the simple
    input->(input program)->output
  subset of the diagram with the complex
    input->(input program)->compiler->(output program)->output
  subset of the diagram. Label each subset as "semantics".]

This is a mistake for two reasons. First, having a clear language with clear meanings, *independent of their implementation* is important for human readers. We want users to be *thinking in Ember*, not thinking in the compiled output language.

Second, your language is now tied to the particular implementation of the compiler. It much harder to improve your compiler without breaking programs, or to evaluate a competing compiler with different opinions.

For this discussion, I'm going to name this mistake the "implementor's trap", because people who know the most about the implementation are the most likely to fall into the trap. They know "how it really works", so why spend time thinking about the abstract semantics?

When we think about the compilation that Ember apps pass through, we *mostly* succeed in avoiding the implementor's trap. Most of the code you write is modern, spec-compliant Javascript. "Spec-compliant" is another way of saying "it follows clearly defined semantics that are independent of any particular implementation". And Ember's own APIs are carefully designed to separate interface from implementation.

But there is one area where we fall short, and that is ES modules.

Ember was a very early adopter of ES modules and Ember people even helped shape the spec. We have been authoring using ES modules *syntax* for a long time (and this will turn out later to be our saving grace). But we have two semantic problems.

[show code example that confuses default export with module. Maybe from a real addon, if I can find one of my prs.]

(This example can be cut if we're over time, it's powerful evidence of brokenness, but not central to my story.)

There are popular addons that did this, so I'm confident there are plenty of apps that do it too. But it's a clear bugfix: people accidentally wrote code that isn't valid Javascript because ember-cli failed to stop them, and we can do the bugfix and make these into errors, and people will need to fix them.

However, there is a larger problem, and it's an instance of the implementors trap. Specifically, what does it really *mean* when you type these import statements?

[show example imports typical in an ember app]

The ECMA spec calls this part the ModuleSpecifier

[annotate the syntactical parts of the import statements]

The spec doesn't tell you what it's supposed to mean. It's left up to the implementor to provide the abstract operation HostResolveImportedModule. It's not part of Javascript-the-language (governed by the ECMA spec, written by TC39), it's part of the browser environment (governed by WHATWG) or the Node environment or whatever kind of Javascript system we're talking about.

So which standard did ember-cli follow for HostResolveImportedModule? None of the above. This is where we have the implementor's trap. We don't have a spec. Each of these means whatever the implementation does.

[take the common examples of import statements and show what each actually means
 - something that calls define from vendor
 - something from treeForAddon
 - a file in the app
 - a totally babel-provided example like Ember.Component
]

Users can't easily know what they're getting. Editors, linters, and typecheckers can't understand enough to offer advice. And the compiler can't perform optimizations like knowing which modules are even being used or not.

I'm criticizing ember-cli here, so in it's defence I want to point out that when we introduced ember-cli, even the idea of having a required build step was controversial. ember-cli was bleeding edge. And we made the right bet. Having shared build tools isn't controversial anymore. But at the time, nobody had ever implemented a complete ES-module-driven compiler before, this was a learn-as-you-go process.

In addition to have an implementation and not a spec, the implementation happens partially at runtime, not just compile time. You can't know what any particular import does without evaluating *all* of the code across the app and all its addons. This doesn't violate the letter of the ECMA spec (because HostResolveImportedModule is a runtime operation), but I would argue it violates the spirit of the design. Look closely at how import is defined. None of these are legal syntax:

[examples of doing non-string-literal things in module specifier]

Furthermore, do you know what the spec says about evaluation order?

[example with imports interleaved with other statements]

Or why this is not legal syntax?

[trying to put an import inside a conditional]

It's because everything about imports is designed to be *static*. Remember: static knowledge is things that a compiler can know without running your program.

[show each with a call to arbitraryCode() gumming up the works
 - non-string-literal modulespecifier
 - evaluation order ambiguity
 - conditional import]

As an interesting aside: all these things *are* legal with *dynamic* import(). That's what it's for.

[show dynamic import examples]

Now that I have described the problem, this is the part of the talk where we switch to describing the solution, which is implemented in the Embroider project.

I said we were lacking a spec, so the first step is making a spec. That spec is in proposed Ember RFC 507. It defines a new "v2" ember package format. In a v2 package, ModuleSpecifiers always have static semantics and they follow Node's conventions, because that is the most widely supported de-facto standard.

There's a lot more in the RFC. The v2 format has a bunch of features designed to ease the transition by giving packages static, declarative ways to provide the same public API as before.

Spec in hand, how can we use it? None of our existing code follows the spec?

We break the problem into three pieces, corresponding to three compiler stages. A thing I didn't say earlier about compilers is that most of them, once they get powerful enough, tend to be multi-staged:

[expand our abstract compiler diagram to multiple stages]

One reason for this is that your first stage can take a rich human-oriented language and simplify it into a simpler intermediate language that is easier for the second stage to understand and optimize. The layers can specialize. New language features compose better when they can live in one layer at a time, leaving the interfaces between the layers relatively stable.

Another reason this is powerful is that you don't need to get your input language all the way to your desired output language. You just need to get it to a well-supported intermediate language, and then let a shared tool go the rest of the way.

[rust example]

For example, the Rust compiler goes from Rust to HIR (high-level intermediate representation) to MIR (middle intermediate representation) to LLVM IR (low-level virtual machine intermediate representation). Which gets passed to LLVM, which is a shared compiler for generating machine code. LLVM is used by lots of other languages too, including C, C++, Fortran, Haskell, Objective-C, and Swift.

This will not be on the test. What I want you to remember is: multi-stage compilation is a powerful technique, and delegating the last stage to a widely-shared general-purpose tool is usually a good idea.

Embroider is a three-stage compiler.

Stage1 is implemented in the `@embroider/compat` package. Its input is all the code you already have: apps and addons. Its job is to take the wild and wooly universe of addons and rewrite them to the new v2 package format.

[first stage diagram]

The output of the first stage is an Ember app that *only depends on v2 packages•. Which means all subsequent stages get to "live in the future", seeing only v2 packages.

Here's an example of the kind of things that happen in stage 1.

[show an example transformation, with files moving, package metadata being added, some imports being rewritten]

 - custom broccoli hooks get run and written out to disk
 - files get moved around so their import paths match their true paths on disk
 - modules that try to escape their package get captured and moved back inside
 - addon's custom babel transforms get applied
 - addon's custom AST Transforms get applied
 - special packages like `@embroider/synthesized-vendor` get built to represent a grab bag of older behaviors in a v2-compatible way (like providing a shared dumping ground in which packages can overwrite each other's files!)

As more of your dependencies ship natively in v2 format (after the RFC is merged!), there's less and less work to do in stage1.

[add stage2 to diagram]

Stage2 is implemented in `@embroider/core`. Its job is to take a collection of v2 packages representing an Ember app and compile away all Ember conventions. Stage2 knows about routes, controllers, components, templates, helpers, engines, fastboot, etc. It outputs Javascript/HTML/CSS that can run without knowing about any of those things.

Here's an example of what happens in stage2

[stage2 example showing the synthesized entrypoint with all optimizations on, and maybe even route splitting (mention that engines look the same). Show the synthesized imports in a template. Show the html too.]

An interesting thing about the output of stage2 is that, in theory, if you had a sufficiently capable web-browser, it could already run your app at this stage. It would not be optimized, and would need to download potentially thousands of separate modules, but it could run. In practice, a little bit of glue code would be needed to actually try this. The stages don't necessarily happen serially. Some of the stage2 work, like compiling templates, is really handled like a callback from stage3, because that lets it happen lazily on demand.

That brings us to stage3, which is generic Javascript packaging and optimization. All the Ember-specific concepts are already compiled away, so with very little setup we can call Webpack or Rollup or a yet-to-be-written future packager. The stage3 packager can see exactly what modules are needed, and when. It can easily leave out unused ones, and emit code for lazily loading parts that are only conditionally needed.

As a result, Embroider gives you:
 - complete pull-based builds (some people call this tree-shaking)
 - faster builds (it turns out static things are easier to compile than highly dynamic things)
 - dynamic import() for arbitrary code splitting and lazy loading
 - automated route-aware code-splitting
 - just import anyting from NPM, statically or dynamically (ember-auto-import already covers this for non-ember packages, but embroider extends that to cover all the code within ember apps and addons too. Use auto-import for all your third-party deps! It's very future compatible with embroider.)

By themselves, each of these features isn't super remarkable in 2019. But you know what *is* remarkable? Adding these features to a big, long-lived app that was started before these features were on anybody's radar, with no big rewrite. And we're going to do this for *every* Ember app. I'm proud of our community ethos of building lasting, living shared solutions.

