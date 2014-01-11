# Hosting custom domain Ghost Blog on Heroku (for free<sup>&dagger;</sup>!)

A guide on how to setup your own blog using the beautiful open source [Ghost Blogging platform](https://ghost.org) behind your own custom domain (e.g. [www.myblog.com](#)), and run it like a pro.

Some of the instructions are geared towards software developers, who want to be able to tinker with the [engine source code](https://github.com/TryGhost/Ghost). You can skip reading those, if you just want to get set up quickly. However, they are useful to be able to update core engine with the newly released fixes & patches.

<sup>&dagger;</sup>: <small>Buying custom domain is not free, of course, the hosting is. But it's fairly cheap!</small>

## Why Ghost and Heroku?

In terms of choosing a blogging platform, there are myriads out there, and it would take [quite a lengthy blog](http://thenextweb.com/apps/2013/08/16/best-blogging-services/) to argue the merits of each. Personally I wanted a blog where I can:

  * Write in [Markdown](http://daringfireball.net/projects/markdown/), because it's [really convenient](http://brettterpstra.com/2011/08/31/why-markdown-a-two-minute-explanation/)
  * Look slick
  * Extend the platform when needed

So I chose [Ghost](https://ghost.org). It looks really elegant, it generated a [lot of buzz lately](http://www.kickstarter.com/projects/johnonolan/ghost-just-a-blogging-platform), and it is based on [Node.js](http://nodejs.oeg) - which I've been eyeing to try for a while now. The final kicker was a [post from Scott Hanselman](http://www.hanselman.com/blog/HowToInstallTheNodejsGhostBloggingSoftwareOnAzureWebsites.aspx) about putting Ghost in Azure (which I did try at first, but switched to Heroku because custom domain support in Azure is not free), and the fact that I learned about [Node.js Tools for Visual Studio](https://nodejstools.codeplex.com/) which shows Microsoft's support for Node (I am a .NET guy by day).

This kills two birds with one stone - I wanted to start a blog AND learn about Node.js! To top it off, I even got to contribute back to the [Ghost project on GitHub](https://github.com/TryGhost/Ghost), which feels good.

[Heroku](http://www.heroku.com) was chosen because I was [looking for a free hosting solution](https://github.com/joyent/node/wiki/node-hosting) that will let me run a Node app behind a custom domain. 

I tried Azure first, but realized custom domain wasn't a free option. The other two considered - [OpenShift](https://www.openshift.com/) and [Cloudnode](http://cloudno.de). Clodenode looked interesting but it was still early stage. OpenShift actually looked like a [viable alternative](https://www.openshift.com/quickstarts/ghost-with-mysql-on-openshift) and possibly [cheaper to scale](https://www.openshift.com/products/pricing) initially.

In the end I went for Heroku, because I felt it was more popular than OpenShift. A very subjective decision, but when presented with fairly similar choices and lack of experience, you have to go with instinct.

> It is worth noting that Ghost [does not officially support Heroku](http://docs.ghost.org/installation/deploy/) (or PostgreSQL, the DB of choice on Heroku). [The issue](https://ghost.org/forum/installation/69-about-heroku/) is essentially that images uploaded via Ghost interface will be stored in /content/images folder, but Heroku [disk is ephemeral](https://devcenter.heroku.com/articles/dynos#ephemeral-filesystem), i.e. is not persistent.

As long as we are aware of this, it's fine - we won't be storing images through Ghost, but rather reference static content somewhere else on the web. See "Static images on GitHub" section below.

## Local setup

Essentials:

  * [Git](http://git-scm.com/downloads) - deploying to Heroku, cloning Ghost repository
  * [Ruby](https://www.ruby-lang.org/en/downloads/) - getting Heroku client, helper gems for Ghost
  * [Node.js](http://nodejs.org/) - running Ghost blog locally for testing

> **Windows**: Ruby Windows installer can be found [here](http://rubyinstaller.org/).

Make sure all the apps above are in your `PATH` to be able to run from command-line. Installers should have set it up by default. Test it by typing `set` on command prompt, which will print all environment variables.

For example, on my machine (Windows):

    Path=...;C:\Ruby193\bin;C:\Program Files\nodejs\;C:\Program Files (x86)\Git\cmd;C:\Users\lev\.gem\ruby\1.9.1\bin;C:\Users\lev\AppData\Roaming\npm

Note that this contains the Ruby gem and Node npm folder, so that we can run global packages from command-line as well.

Now, install Heroku as a gem:

    > gem install heroku

Verify you can run it:

    > heroku --version


## Custom domain

To run your blog behind a custom domain, you need to purchase it first. Domains can be fairly cheap, as low as few bucks per year. Shop around! I got my domain through [gandi.net](http://www.gandi.net), as I like their management console. [GoDaddy](http://www.godaddy.com) is a popular choice also.

[Domai.nr](https://domai.nr/) is also a nice little resource to search for available domains.

## Heroku account

Sign up for a free account on [Heroku](https://www.heroku.com/), should take less than a minute. Then run on command-line:

    > heroku login

After you enter your Heroku credentials, a public/private key pair is generated locally and is used subsequently to establish secure SSH session with Heroku when executing commands. That way you don't have to enter your credentials every time.

That's it - we will deploy an app a bit further down. When you are done with this guide, you may want to read more of [Heroku Docs](https://devcenter.heroku.com/) to see what their awesome platform can do.

## Mailgun account

[Mailgun](http://www.mailgun.com/) is a great online email service which you can use to both send and receive emails. We will use it to configure Ghost email sending capabilities (e.g. "forgot password" feature). Additionally we'll use it to forward all emails sent to addresses under your custom domain to an email address of your choosing.

Setup a free account with [Mailgun](http://www.mailgun.com/). Follow instructions to add your custom domain and [get your domain validated](http://documentation.mailgun.com/quickstart.html#verifying-your-domain), by adding necessary DNS domain records for SPF & DKIM.

Under Domains tab, for you custom domain, note the "SMTP Authentication" credentials. You will need them to setup environment variables for the Heroku app.

To have Mailgun also handle inbound email for your domain, e.g. bob@mydomain.com, [setup MX records](http://documentation.mailgun.com/quickstart.html#verifying-your-domain):

    MX  mxa.mailgun.org.
    MX  mxb.mailgun.org.

Under Routes tab, create a new route, using `catch_all()` filter expression. Set the action to be `forward("my.email@gmail.com")`, and use your real email where you want it to be forwarded.

## Ghose release package

Ghost team does release hostable packages that can be downloaded from [ghost.org](https://ghost.org/) (signup required) or [GitHub](https://github.com/TryGhost/Ghost/releases). These allow you to start running Ghost very quickly, but you will not be able to modify and contribute back to the Ghost source code. If you want to set this up properly, then go to next section.

If, on the other hand, you want a quick setup, then just extract the zip into a folder. Then drop into command prompt, and initialize it as a Git repository:

    > git init
    > git add .
    > git commit -m "Initial commit"

Now just read about modifying configuration file under "Hosting preparation".

## Clone Ghost source

Cloning Ghost source from GitHub allows you to merge latest changes and bug fixes hot off the press, even though it means sometimes running an unstable version of the code. But it also allows, if desired, to use only the stable branch/tag, resolving any issue with instability.

I like this option because it also allows me to fix issues in the code myself, and to [contribute back to the Ghost project](https://github.com/TryGhost/Ghost/pull/1849) on GitHub by submitting Pull Requests.

Let's get started. First, I recommend setting up a [GitHub account](http://github.com/), if you don't have one yet, and "forking" [Ghost project](https://github.com/TryGhost/Ghost). Once ["forked"](https://help.github.com/articles/fork-a-repo), you should have a copy of Ghost source tree under your GitHub account, and ready to setup locally.

![Fork button on GitHub](http://static.lionhack.com/images/2013-12-31-hosting-ghost-blog/fork-ghost.png)

From command-line run:

    > git clone https://github.com/username/Ghost.git
    > cd Ghost
    > git remote add upstream https://github.com/TryGhost/Ghost.git
    > git fetch upstream

At this point you have the Git repository locally, and original Ghost repository is setup as an "upstream" remote, so you can fetch from it.

First, let's do some cleanup and remove extra branches we are not going to use - we don't need them in our fork. Following command is for **Windows only**, and it removes all branches other than master:

    > for /f "tokens=2 delims=/ " %x in ('git branch -r ^| find "origin" ^| find /V "origin/master"') do git push origin :%x

Now let's create a *hosting* branch that will contain the changes necessary to deploy the Ghost app to Heroku:

    > git checkout -b hosting master

That will setup *hosting* branch based on current state of the *master* branch, where the latest code is. If, instead, you wanted to base off a stable version, run `git tag -l` to find the latest stable version tag, and run this:

    > git checkout -b hosting tags/0.3.3

This branch will be used to make certain modifications to the source to customize it before deploying to Heroku. You can also use it to fix bugs or add features to the Ghost source, and deploy the changes to Heroku.

## Hosting preparation

In a source form, Ghost is not yet ready to be hosted. Some of its client-side Javascript files are generated using [Sass templates](http://sass-lang.com), and then minified. There is a [Grunt](http://gruntjs.com) task that does this for you, but we don't want to be running it here.

Heroku deployment works through a Git remote push, which means you'd have to have all of the files you intend for destination inside the repository. That means committing these generated files. This is not ideal, because if you later merge newer coder from Ghost on GitHub, it will not be obvious that you have to re-generate these files. Not to say that each time it will generate an extra commit in your repository, which has to be re-based every time.

A better solution is to run it during deployment to Heroku dyno, and generate these files from source every time, guaranteeing they are the latest. I just happen to have such a solution for you to re-use!

Create a file `hosting-install.sh` in the source root folder (where `README.md` is):
    
    #!/bin/sh
    
    ## Check for re-entrancy
    if [ ! -z $_HOSTING_INSTALL_REENTRANCY ]; then 
        exit 0
    fi
    
    ## Setup config.js
    if [ ! -L "./config.js" ] && [ ! -f "./config.js" ]; then
        ln -s config.hosting.js config.js || exit 1
    fi
    
    ## Install Bourbon
    export GEM_HOME=./node_modules/.gem/ruby/1.9.1
    export LANGUAGE=en_US.UTF-8
    export LANG=en_US.UTF-8
    export LC_ALL=en_US.UTF-8
    PATH="$GEM_HOME/bin:$PATH"
    gem install bourbon sass --no-rdoc --no-ri || exit 2
    
    ## Install Dev dependencies (package.json)
    export _HOSTING_INSTALL_REENTRANCY=1
    npm install
    npm install grunt-cli
        
    ## Run Grunt init tasks
    if [ ! -d './core/client/assets/sass/modules/bourbon' ]; then
        ./node_modules/.bin/grunt shell:bourbon prod
    else
        ./node_modules/.bin/grunt prod
    fi
    
Now to make it run when deploying, we have to add it as an ["install" script](https://npmjs.org/doc/misc/npm-scripts.html) in the `package.json` file:

<pre>
    "scripts": {
        "start": "node index",
        "test": "grunt validate --verbose",
        <span style='background-color: #fbb'>"install": "hosting-install.sh"</span>
    },
</pre>

With this install script, every time you push to Heroku, it will re-compile templated CSS & JS files on the fly, before updating the app. If there are any errors, the deployment will stop and will not update the app with a broken version, which is a nice bonus!

Let's also add PostgreSQL as an npm dependency to `package.json` file, since we will use it to store Ghost database (Heroku does not support Ghost's default sqlite file database, due to its ephemeral filesystem):

<pre>
    "dependencies" : {
        ... ,

        <span style='background-color: #fbb'>"pg": "2.9.0"</span>
    },
</pre>

`config.js` file is usually used for local testing and development, and as such it is excluded from source control (see `.gitignore`). It is useful to be able to modify this file locally to play with the settings while hacking away at the codebase.

To keep in line with this convention, we will create a new file `config.hosting.js`, which will house production-only configuration:

<pre>
// # Ghost Configuration
// Setup your Ghost install for various environments

var path = require('path'),
    config;

config = {

    // ### Production
    production: {
        // must not contain a path suffix after the hostname - "subdirs" are not (yet) supported! 
        url: <span style='background-color: #fbb'>'http://www.yourdomain.com'</span>,
        forceAdminSSL: true,
        mail: {
            transport: 'SMTP',
            options: {
                service: 'Mailgun',
                auth: {
                    user: process.env.MAILGUN_USER,     // mailgun username
                    pass: process.env.MAILGUN_PASSWORD  // mailgun password
                }
            }
        },
        database: {
            client: 'pg',
            connection: {
                host: process.env.POSTGRES_HOST,
                user: process.env.POSTGRES_USER,
                password: process.env.POSTGRES_PASSWORD,
                database: process.env.POSTGRES_DATABASE,
                port: '5432'
            },
            debug: false
        },
        server: {
            host: '0.0.0.0',
            port: process.env.PORT
        }
    }

};

// Export config
module.exports = config;
</pre>

You might have noticed in the `hosting-install.sh` install hook script, we'll create `config.js` during deployment and link it to this file. This allows you to still have local `config.js` file for playing/testing, and a production-only configuration that will be used on deployment.

> **NOTE:** All production values are referenced via environment variables! You do not want to hardcode things like passwords in a file that is likely to be visible in a public Git repository.

You'll set these variables in "Deply to Heroku" section below. Finally, let's create `Procfile` which will [specify how to run](https://devcenter.heroku.com/articles/procfile) this web application. It just needs to have one line:

    web: node index.js

Now you can commit the 4 files:

  * `package.json` (modified)
  * `Procfile` (new)
  * `config.hosting.js` (new)
  * `hosting-install.sh` (new)

<pre>
> git add .
> git update-index --chmod=+x hosting-install.sh
> git commit -m "Heroku: Configuration"
</pre>

Note how we had to specify "+x" mode for the `hosting-install.sh` file to make it executable on the remote Heroku Linux box. Otherwise it will fail to execute during `npm install` phase.

## Search engine support

At a minimum your blog site should contain `robots.txt` file, which [tells search bots](http://www.robotstxt.org/) what can and can't be indexed. Create `robots.txt` file in the root source folder:

    User-agent: *
    Disallow: /ghost/

This will hide /ghost admin section from search engines, and allow everything else to be indexed.

You'll want to submit your site to search engines. For example, with Google this is best done using [Webmaster Tools](https://www.google.com/webmasters/tools/home?hl=en). Once it's setup and validated, add a Sitemap to help Google discover your new blog posts. Go to Crawl > Sitemaps, and click "Add/Test Sitemap" - use RSS feed as your sitemap: `http://www.myblog.com/rss`.

As new posts are added they will be automatically discovered and indexed!

## Google Analytics

Adding Google Analytics to the blog is a good idea - it gives visibility on how many people visit,  where in the world they are coming from, which pages are most popular, etc. This is fairly easy to do. For starters, you need a [free Google Analytics account](http://www.google.ca/analytics/). 

Once it is setup and verified for your domain, there will be a snippet of JavaScript code that needs to be included in every page. The best way to do this is to create a custom theme, by copying an existing theme and customizing it.

  * Go to `content/themes` folder
  * Copy your theme, e.g. the default `casper` theme
  * Rename it to something like `my-blog`

Now you can customize it. Find `default.hbs` file inside your new theme folder, and add Google Analytics code before the closing `</body>` tag:

<pre>
    <!-- Google Analytics -->
    &lt;script&gt;
    if (document.location.hostname !== "localhost") {        
        (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
        (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
        m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
        })(window,document,'script','//www.google-analytics.com/analytics.js','ga');

        ga('create', 'UA-XXXXXXXX-X', 'myblog.com');
        ga('send', 'pageview');
    }
    &lt;/script&gt;
</pre>

To add it to the Git repository, you'll also have to modify `.gitignore` file in the root:

  * Replace line `/content/themes/**/*` with `/content/themes/*`
  * Add line `!/content/themes/my-blog` 

Commit the modified theme to Git:

    > git add .
    > git commit -m "Hosting: custom theme with Google Analytics"

## Deploy to Heroku

The first step is to create a Heroku application that will house the Ghost blog. This needs to be done once - run following from the command-line at the source root folder:

    > heroku create my-blog --buildpack https://github.com/heroku/heroku-buildpack-nodejs

This will create an app called "my-blog" and set the buildpack to Nodejs (of course, choose something unique as a name). Normally [Heroku auto-detects application type](https://devcenter.heroku.com/articles/buildpacks), but in our case, if we don't specify it explicitly, it will wrongly detect it as a Ruby app because of the `Gemfile` presence.

A remote called "heroku" will also be added to your Git repository, so that you can push to it using Git client. You can verify it's created by running `git remote -v`.

Let's also create a PostgreSQL database to store blog data on the back-end. We can do this via command-line as well:

    > heroku addons:add heroku-postgresql:dev
    > heroku pg:wait

This will create a new free instance of PostgreSQL. Run `heroku config` to see a new configuration string that was added, e.g. `HEROKU_POSTGRESQL_BLACK_URL` ("color" may be different, it is auto-assigned). Note the parameter name, and then run the following:

    > heroku pg:promote HEROKU_POSTGRESQL_BLACK_URL

This sets up this database as your primary one for the current Heroku app.

Next is setting up all the environment variables to be used at run-time. These are also only defined once, and don't need to be changed unless configuration changes. They are referenced in the Ghost `config.hosting.js` file we created earlier.

To set values for these environment variables, run from command line:

    > heroku config:set NODE_ENV=production
    > heroku config:set MAILGUN_USER=postmaster@myblog.com
    > heroku config:set MAILGUN_PASSWORD=xxxxxxxxxxx
    > heroku config:set POSTGRES_DATABASE=xxxxxxxx 
    > heroku config:set POSTGRES_HOST=xxxxxxxx 
    > heroku config:set POSTGRES_USER=xxxxxxxx 
    > heroku config:set POSTGRES_PASSWORD=xxxxxxxx 

Mailgun user/password come from [Mailgun](http://www.mailgun.com) admin panel. PostgreSQL values can be gathered from running `heroku pg:credentials DATABASE` on command line.

Finally, let's set up the custom domain for the blog:

    > heroku domains:add www.yourdomain.com

At your domain registrar (e.g. GoDaddy or Gandi) set up a CNAME record:

    www   CNAME   my-blog.herokuapp.com.

"my-blog" is the name of your Heroku app. You might also want to setup a 301 redirect from `yourdomain.com` (without `www`) to `www.yourdomain.com`. Different domain name providers call it differently (if they provide this service), [Gandi](http://www.gandi.net) calls it ["Web forwarding"](http://wiki.gandi.net/en/domains/management/domain-as-website/forwarding). 

OK, at this point we are **ready to hit the big red button**, i.e. finally deploy. Here we go:

    > git push -u heroku hosting:master

It will start the push to remote Heroku repository for your app, and show the progress of running the Nodejs buildpack, which will in turn execute `npm install`, which should pull in all the dependencies and then execute our custom install hook script to compile CSS & JS scripts. If there are any errors, deployment will stop and errors will be shown in the output.

If all is successful, the app should now be running. You can check its status via:

    > heroku ps

Any errors, warnings or status messages are also recorded in a log, which is visible via:

    > heroku logs

It's ready now!

Go ahead and navigate your browser to your custom domain (e.g. `http://www.myblog.com/`), the initial Ghost 'create account' page should display. You can also always access it via Heroku internal app URL - `http://my-blog.herokuapp.com`.

## Store local repository

Even though it has been pushed to Heroku via Git, Heroku does not guarantee that your remote Git repository there will be safe. It is best to push all your changes back to GitHub for safe-keeping.

    > git push origin hosting

## Post online

To access your Ghost blog admin console, go to `https://my-blog.herokuapp.com/ghost/`. Note the use of HTTPS to ensure that username/password are not transmitted over unsecure connection.

Through the admin console you can manage your blog and publish new posts. [Ghost Guide online](http://docs.ghost.org/) has additional information on available functionality if you ever get lost. You shouldn't be though - interface is pretty clean and intuitive.

**NOTE**: Remember the limitation mentioned earlier - because we are hosting on Heroku, we cannot upload images through Ghost UI since they cannot be stored on Heroku disk. Instead we should refer to static resources elsewhere via the full URL.

Next section is on how to easily manage your static resources in the cloud.

## Static images on GitHub

While any cloud store (e.g. [Amazon S3](http://aws.amazon.com/s3/) or [Dropbox](http://www.dropbox.com) would do, if it offers a public URL to your resource, we can use GitHub to host images for our blog for free. GitHub has a [Pages feature](http://pages.github.com/) which allows us to serve static resources via HTTP on a custom domain.

[GitHub Pages](http://pages.github.com/) allows you to host any static site content, not just images, and serve it behind your own custom domain. For this we will create a new subdomain for the static content - create a new DNS record with your domain provider:

    static  CNAME   username.github.io.

Replace `username` with your GitHub username. This will setup domain _static.myblog.com_ to point to GitHub Pages. Next is to setup a GitHub repository that will house your static content.

I create a repository where not only I have my static resources, but where I also save all my blog posts in Markdown, for additional safe-keeping (and to allow "Offline post writing" - see next section). Let's say we call repository "_my-blog_".

Let's clone it locally to create content:

    > mkdir my-blog
    > cd my-blog
    > git clone https://github.com/username/my-blog.git
    > git checkout master

According to [GitHub Pages documentation](https://help.github.com/articles/user-organization-and-project-pages), all content should go to `gh-pages` branch. But _master_ branch is still the default and will show up whenever anyone visits this repository. At a minimum we should put a _README.md_ and _LICENSE.md_ files in the `master` branch.

For _LICENSE.md_ I just use the permissive [Creative Commons license](http://creativecommons.org/choose/), so that my content can be freely linked to and shared. You can checkout how I set it up [here](https://github.com/gimelfarb/blog-lionhack).

    > git add .
    > git commit -m "License + Readme"
    > git push origin master

Now we can create `gh-pages` branch that will house static content:

    > git checkout --orphan gh-pages
    > git rm -rf .

Create file `CNAME` which has the subdomain you are mapping to this static site, e.g. "static.myblog.com". This is how GitHub Pages feature knows to serve these files when it serves a URL for _static.myblog.com_.

Create `images` folder and put all of the images you want to reference there. I also create a subfolder for each blog post to separate them.

Now commit and push to GitHub:

    > git add .
    > git commit -m "Static site contents"
    > git push -u origin gh-pages

Your images can now be accessed via _http://static.myblog.com/images/subfolder/image.png_. 

Reference directly in your blog posts, and sleep better knowing you are also benefiting from automatic CDN acceleration provided by GitHub pages, so that your content loads faster! 

## Offline post writing

You can also use the GitHub repository from previous section to save your Markdown-based blog posts, before you publish them through Ghost. It allows for an efficient offline writing process.

Firstly, download the excellent [MarkdownPad 2 for Windows](https://markdownpad.com/), which has a 2-pane view - on the left your write your post in Markdown and on the right you get a formatted preview. You can now write your Markdown-based blog posts offline on your PC, and then upload to your Ghost blog when ready.

Save your in-progress work to the `my-blog` GitHub repository (created in previous section). I save them under `_pages` subfolder, just so that I can have the benefit of [running them through Jekyll](https://help.github.com/articles/using-jekyll-with-pages), which is a feature of GitHub Pages. Although this is of questionable usefulness ...

Push your work to GitHub regularly as a way to keep a backup of your offline writing work.

Previewing images, referenced via _http://static.myblog.com/images/..._ links, can be difficult when working completely offline (i.e. no Internet connectivity). To work around this, you need to enable a redirection so that "static.myblog.com" somehow resolves to a local folder, allowing you to preview the images referenced in this way.

Modifying [system `hosts` file](http://en.wikipedia.org/wiki/Hosts_(file)) is one way to do this, but this will affect you after you go online, and it is tedious to keep remembering to edit the file. Something more dynamic would be preferred.

[Fiddler debugging proxy](http://fiddler2.com/) comes to the rescue. Under Tools menu there is a HOSTS item, which brings up an editor for a dynamic host re-mapping. As long as Fiddler is running, DNS entries are re-mapped. When you switch online, terminate Fiddler, and DNS is back to being resolved through Internet.

We add this to the Fiddle HOST remapping window:

    127.0.0.1       static.myblog.com

Now, while Fiddler is running, URLs for _static.myblog.com_ will try to resolve to a local web server. All you need now is a local HTTP server to serve files from that folder.

We can use [Node](http://nodejs.org) for this, and get it [done in 2 lines of code](http://stackoverflow.com/questions/6084360/node-js-as-a-simple-web-server#8427954)! Go to your static files repository folder, and create a file `server.js`:

    var connect = require('connect');
    connect.createServer(connect.static(__dirname)).listen(80);

On command-line install the npm package:

    > npm install connect

Then start the server when you need it:

    > node server.js

Your _static.myblog.com_ images now show up in the preview window, even though your are offline (or have not yet posted them through GitHub Pages). Neat, right?

## Updating Ghost source

Updating Ghost with the latest changes from the [open-source GitHub project](https://github.com/TryGhost/Ghost), is essentially running a number of Git commands:

    > git fetch upstream
    > git checkout master
    > git merge --ff-only upstream

This pulls latest changes from Ghost repository on GitHub, and updates local 'master' branch.

    > git checkout hosting
    > git rebase master

Updates `hosting` branch to be "on top of" latest changes in `master`. If you cloned from a stable tag before, then instead you can rebase on top of a new version (as opposed to latest unstable changes):

    > git rebase tags/0.4.0

Once "rebased" successfully, can push the new version to Heroku with:

    > git push -u heroku hosting:master

As Heroku re-deploys the app, the new version should be up an running when command completes.

Happy blogging!