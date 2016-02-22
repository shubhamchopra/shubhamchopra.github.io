---
title: Hello World! Setting up a blog with Hakyll + Travis CI
description: Walking through the process of starting a static blog using Hakyll and Travis CI
---

## Starting a blog
I have recently started learning Haskell and thought it would be a good idea to blog my progress. Nothing too fancy, just a few basic things, mostly a static website. Actually, completely static website. The folks at GitHub have graciously allowed everyone to start their own blogs, but they mostly want static content. I did not like the idea of writing html files and then maintaining them individually. The thought of changing the look or say, the header or footer, or maybe to edit the main page after every post, did not seem very appealing. I wanted a tool, that could create a blogging site for me. I looked at WordPress and it had the capability of generating static webpages that I could then upload. Promising, but didn't look too encouraging.

Enter [Hakyll](http://jaspervdj.be/hakyll). It lets me write posts in whatever format I please and could generate static website just like I wanted. The fact that it was all written in Haskell positively warmed my heart. So, the problem at hand is to find the easiest way we can have our blog deployed as "source code", written in a format like, but not limited to, markdown, latex, lhs even. We then need a system that can read this source, build the Hakyll binaries, and generate the site and push it to the github repo for hosting. Bonus points for being able to host on your own domain name.

First, lets get the environment prepped. I will be using [Stack](http://haskellstack.org/) and [GHC](https://www.haskell.org/ghc/) 7.10. The installation process is fairly well documented. Once you have stack working, you can follow the instructions on installing Hakyll. It might take a little time for stack to download and build all the libs it depends on.  

## Getting started with [Hakyll](http://jaspervdj.be/hakyll)
For a simple blog, you really don't have to tinker around with the Haskell code base. The Haskell code base defines your routing rules, and your compilation rules that will be used to read your posts, convert them to html and then put them in pre-defined templates. This is then compiled into a binary called _site_. You can use this binary to build your static site, and even see the site using its _watch_ command. Bottomline, if you aren't modifying or creating new directories or changing the structure of the website, you wouldn't need to touch the Haskell bits. Once you are familiar with this, you are ready to start writing posts.

## Pushing stuff to GitHub
Once you have yours posts ready, you can create a repo in your account on GitHub that is called _username_.github.io. Init git in the directory you have your website in. You can add _site, _cache, .stack-work to the .gitignore file. You should push your code base to a branch in your repo, say __source__. This is because GitHub expects your static website to be on the __master__ branch. 

You can build your site using _stack exec site build_. Your static website would be generated in the directory _site. At this point, you have your whole website ready. You can try pushing the contents of _site folder to the master branch of your repo and see if you can access it using the address _username_.github.io. The downside of this, is that you would have to repeat this process every time you did any modification to your site. Or every time you wrote a new post. Not fun. That's where Travis CI comes into the picture.

## [Travis CI](https://travis-ci.org)
Travis is a continuous integration tool. It lets you specify the details of your project in a simple specification and uses it to build and test your project. You don't have to setup your own environment like you would have to do with say Jenkins. And its free for open-source projects! So what you now do is specify the details of your Hakyll project in a .travis.yml file and let Travis talk to your GitHub repo. Things can get a little nasty here, so I will walk through the bits.

First up, as you probably saw when you installed Hakyll, it takes a decent amount of time to download and build all the libs it depends on. So we would ideally want to cache this collection of libs. Travis lets you do that with its new container infrastructure. We enable it by setting _sudo: false_.

    # Use new container infrastructure to enable caching
    sudo: false

Travis doesn't support stack yet. So we will download our own tools and use them. We don't want Travis to spend any time boot-strapping a heavy build environment.

    # Choose a lightweight base image; we provide our own build tools.    
    language: c

We set some environment variables here. We will get to the _secure_ section in a bit.

    env:
      global:
      - GHC_VERSION=7.10.3
      - secure: [encrypted secret key]

Here, we tell Travis that we would only be interested in checking out the branch source.

    branches:
      only:
      - source
    
    #Caching so the next build will be fast too.    
    cache:
      directories:
      - "$HOME/.stack"

This is where we download our own stack binary from Stack's website. We decompress it and make stack available in the path.

    before_install:
    - mkdir -p ~/.local/bin
    - export PATH=$HOME/.local/bin:/opt/ghc/$GHC_VERSION/bin:$PATH
    - travis_retry curl -L https://www.stackage.org/stack/linux-x86_64 | tar xz --wildcards --strip-components=1 -C ~/.local/bin '*/stack'

Note that we still would not have GHC available in the environment yet. So we install it here.

    addons:
      apt:
        sources:
        - hvr-ghc
        packages:
        - cabal-install-head
        - ghc-7.10.3

The _install_ and _script_ tags show the usual commands we would run to build the website.

    install:
    - stack --no-terminal build
    
    script:
    - stack --no-terminal exec site build

A lot of the magic happens in next section. A few things to note before we go further. Travis checkouts the branch in a way that puts it in a _detached head_ state. This means any changes to the branch cannot be committed. Also, we had _site in .gitignore.

So what we do here, it to enter the _site folder, init a git repo, set the remote to our own repo and just commit back only the content of this directory to the _master_ branch of our repo. That's all fine, but lets say this commit happens once. The next time we push any changes, the master branch already exists and git won't let us push changes until we pull the master branch. Since the website is anyways getting generated every time, and it is not the source we are tracking, deleting the master and then pushing the website to master afresh avoids all the trouble around merge conflicts.

    after_script:
    - cd _site
    # Adding a CNAME file, so our custom domain works with github 
    - echo "blog.${GH_USER}.com" > CNAME  
    - git config --global user.email "$GH_EMAIL"
    - git config --global user.name "$GH_NAME"
    - export REMOTE=$(git config remote.origin.url | sed 's/.*:\/\///')
    - git init  
    - git remote add github https://${GH_USER}:${GH_TOKEN}@${REMOTE}
    - git add --all
    - git status
    - git commit -m "Built by Travis ( build $TRAVIS_BUILD_NUMBER )"
    # We don't care about commits to the master as a completely new site would be generated everytime  
    - git push github :master 2>&1 | grep -v http  
    - git push github master:master 2>&1 | grep -v http

If you aren't hosting your blog on your own website, you don't need the line where __CNAME__ file is created. 

### The _secure_ section
As a part of this process, you would be committing code back to your repo in GitHub. You would have to provide authentication tokens to be able to do that. But these authentication tokens are equivalent to the password to your GitHub account. _Do NOT put your token into a file like this and make it public_. Travis solves this problem by providing an encryption tool. More information about is available [here](https://docs.travis-ci.com/user/encryption-keys/). 

It lets you encrypt any secure keys with a simple command like _travis encrypt key=value_. This command would generate an encrypted string that you can put in your .travis.yml file. Travis uses asymmetric encryption for this. You encrypt the data using a public key, and so far as I could find, Travis has the private key that it doesn't disclose. This isn't very encouraging either, but is the lesser evil. You should therefore make sure you provide just the bare minimum access to this key. 

## Custom domain name
Once you are done with the GitHub portion, your website can be accessed at _username_.github.io. You can now forward your domain name to this github location. GitHub has some decent documentation about it [here](https://help.github.com/articles/setting-up-a-www-subdomain/). It boils down to creating a _CNAME_ record if you want a subdomain like _blog_.somedomain.xyz or an _A_ record with your DNS provider pointing to your GitHub repo (_username_.github.io). If you only do this, the links within your website will still show _username_.github.io. You can add a file named _CNAME_ to your site that has your sub-domain name. And you are done! 

## Gotchas
* I found out that I had to make _master_ branch the default branch so GitHub start hosting the site in it. That means that it won't let you delete the branch. Oddly, just switching the default branch back to _source_ seemed to work. GitHub has no trouble reading the website in the master branch now, even though it is not the default branch.
