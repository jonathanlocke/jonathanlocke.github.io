
2021.09.07

### A poor man's GitHub multiple-repository, continuous integration build system &nbsp; <img src="https://state-of-the-art.org/graphics/gears/gears-32.png" srcset="https://state-of-the-art.org/graphics/gears/gears-32-2x.png 2x" style="vertical-align:baseline"/>

#### Refactoring feature branches across multiple repositories

A common use case when working with multiple, dependent repositories is to use [git flow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow) to create multiple feature branches:

    kivakit            [feature/simplify-log-api]
    kivakit-extensions [feature/simplify-log-api]

If project(s) in *kivakit-extensions* here depend on projects in *kivakit*, refactoring code in *kivakit* can propagate code changes to *kivakit-extensions*. Then both feature branches are modified.

When we commit and push these feature branches (ideally all at the same time for convenience), it would be nice to know that our continuous integration (CI) build system will build them correctly. There are existing solutions to the problem of repository build order, including the [KIE build chain tool](https://blog.kie.org/2021/07/cross-repo-pull-requests-build-chain-tool-to-the-rescue.html). However, for [KivaKit](https://www.kivakit.org), we decided it was desirable to have a bit more flexibility than [GitHub Actions](https://docs.github.com/en/actions) provide with their *.yaml* configuration files, so we created a few simple scripts to manage our multi-repository builds. Remember Perl?

#### The KivaKit build system

In our poor man's solution to this problem, each repository has its own *.github/scripts/build.pl* file that is invoked from a set of *.yaml* workflows. The relevant portion of a workflow *.yaml* file is very simple. It passes all the build responsibility off to the Perl script *build.pl*:

    - name: Build
       run: |
         perl ./.github/scripts/build.pl package

The *build.pl* script for a given repository takes a *build-type* argument, which can be one of two values:

1. *package* - build the repository's projects
2. *publish* - build the repository's projects and publish them to OSSRH

The build script clones the repository *Telenav/cactus-build* into the GitHub Action workspace and includes Perl functions from a shared *build-include.pl* file in *.github/scripts*:

    system("git clone --branch develop --quiet https://github.com/Telenav/cactus-build.git");
    
    require "./cactus-build/.github/scripts/build-include.pl";

The script then uses the provided functions to clone and build any dependent repositories in the proper order, followed by the repository itself:

    my ($build_type) = @ARGV;
    my $github = "https://github.com/Telenav";
    
    clone("$github/kivakit", "dependency");
    clone("$github/kivakit-extensions", "build");
    
    build_kivakit("package");
    build_kivakit_extensions($build_type);

[This script](https://github.com/Telenav/kivakit-extensions/blob/develop/.github/scripts/build.pl) in *kivakit-extensions* clones *kivakit* and *kivakit-extensions* into the GitHub Actions workspace (the second parameter determines which branch gets checked out). It then builds the project *kivakit* before building the dependent project, *kivakit-extensions*.
 
#### Conclusion

This simple build system was created over a fun (and nostalgic!) weekend with Perl. It is true that this is not the most efficient way to build dependent repositories. And for private repositories, that inefficiency will make GitHub a little richer every day. However, this approach to clone and build all required projects, in order, on every repository build action, is conceptually simple, robust, flexible and easy to debug offline.

#### Caveat - Out of order pushes

It is useful to note here that if the *kivakit-extensions* build action clones *kivakit* before it has been pushed it will build against the wrong branch. Out-of-order pushes are a problem with any build method. Imagine that someone pushes a *kivakit-extensions* feature branch and then goes to lunch without pushing the corresponding *kivakit* feature branch. Failure is always possible with GitHub CI, because GitHub doesn't know enough about repositories and branches, and how they relate to each other. 

The *KivaKit* *clone()* function in *build-include.pl* makes this issue less likely (assuming you have a high bandwidth Internet connection) by simply delaying for 15 seconds before cloning dependencies. It is no substitute for a proper locking mechanism (which could ensure that cross-repository CI builds would never fail), but in practice it works nearly all the time when all interdependent feature branches are pushed at the same time.

<br/><img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Code

The KivaKit build system is somewhat specific to KivaKit in a few places, but it could easily be modified to work in other situations. Feel free to download it and tailor it to your needs. The Perl code for the *kivakit* project's build is here:

* [cactus-build/.github/scripts/build-include.pl](https://github.com/Telenav/cactus-build/blob/develop/.github/scripts/build-include.pl)
* [kivakit/.github/scripts/build.pl](https://github.com/Telenav/kivakit/tree/develop/.github/scripts)
* [kivakit/.github/workflows/](https://github.com/Telenav/kivakit/tree/develop/.github/workflows)

The build and workflow files for other KivaKit projects are available in the same location in those projects. 

[Lexakai](https://www.lexakai.org) also uses the KivaKit build system.

<br/>

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />

Questions? Comments? Tweet yours to @OpenKivaKit or post here:

<script
  async
  src="https://utteranc.es/client.js"
  repo="jonathanlocke/jonathanlocke.github.io"
  issue-term="kivakit-build"
  theme="github-dark"
  crossorigin="anonymous"
></script>

