# Software Versioning Strategies

Software Versioning can be one of those areas where you never feel like you got it exactly right. There is no definite guidance out there with a solution that would satisfy everyone. Mostly software teams are either confused about the subject, or are choosing to ignore it. This guide aims to fill the gap, and offer a practical look at various popular strategies and trade-offs.

Some of the techniques will be geared towards Microsoft stack (Windows, .NET), as it's what I am most experienced with, but the principles apply in general. Linux, Node.js, Python & Ruby are also lightly touched upon.

## Versions Everywhere

We are all pretty used to the term "version" nowadays. Most commonly used in the software world, it had leaked into the media and other industries. Movie sequels are being versioned - ["Fast & Furious 7"][ff7] (7!?), shoes are being versioned - ["Air Jordan XX8"][ajxx8], and, most popularly, books are being versioned - ["One Minute Manager, 1984 edition"][om1984]. Actually, looking at books, people have been versioning for some time now - ["Encyclopedia Britannica"][eb], since 1768!.

The premise is simple - as products live on and continue to improved, newer releases have to be distinguished from the previous ones. Product name does not change, because market has already become familiar with it, so something is appended at the end to indicate that it is **newer** (or **different**).

While versioning existed long before the digital age, software really pushed the issue forward. Modifying and releasing new copy of software is a very quick process, magnitude of times faster than it is to change an industrial production line to produce a new piece of clothing or print a new book edition. Thus software cycles are much **faster**, and a potential for a lot of **concurrent editions** is much greater.

Simply using years (or even months), as in book editions, is not sufficient. New versions of software can be produced within minutes. In addition, software has a massive parallel aspect to it - **software streams** - where several major versions can exist, and all can be continuously updated at the same time. This rarely happens with your shoes. (I wish it did though, sometimes I just don't want to upgrade to this year's catalog model, I want an improvement to my old pair!)

[ff7]: http://www.imdb.com/title/tt2820852/
[ajxx8]: http://www.footlocker.com/_-_/keyword-history+of+air+jordan
[om1984]: http://www.amazon.com/One-Minute-Manager-Quickest-Prosperity/dp/B001ADUOT8/ref=tmm_pap_title_11?_encoding=UTF8&sr=1-1&qid=1393195279
[eb]: http://en.wikipedia.org/wiki/Encyclop%C3%A6dia_Britannica

## Why Version?

Before diving into **how** to implement versioning, let's stop and consider **why** we would want to do it in the first place! After all, if we know the exact reasons why it is useful, then we can better judge whether proposed solutions are a fit.

We've alluded to this in the previous section, referring to what's called a **public version**. This is the version that is publicly visible, and mostly carries marketing weight (it is most likely to be defined by marketing/sales department). "Windows 7", "iPhone 5S", "Office 2013" - are all examples of a public version.

**Public version** is intended to be simple and memorable, indicating to customers that it is new & shiny (assuming people generally want "new & shiny"). People don't understand "10.6.6527.14789" - but they get "2013" or "5". It has been increasingly popular to use the year of release as the public version number, as it simply and powerfully conveys up-to-date status. Car manufacturers have been doing it for a long time.

**Private version** is what we're used to in the software world. An internal stamp that (hopefully) uniquely identifies a particular piece of software. Software, like a car, can be made of many parts. Taking the car analogy further, car's "private version" is the [VIN chassis number][vin]. Manufacturers release and maintain massive catalogs of parts, mapping to car "version numbers". A mechanic can then order an exact part that would fit your vehicle.

Without a "private part number", you wouldn't be able to service your software out in the wild, since you wouldn't know the exact "shape" that a replacement module has to be to fit into the overall system. Imagine if you were forced to change your whole car when a tail light broke.

Therefore, **private version** number is used just like a catalog identifier. It is is intended to be used when troubleshooting or servicing your software. (I like Jeff Attwood's ["dogtag" analogy][jattver]!) It must map to a description of what that software piece is like - what's its shape and function. And what better "description" than the **original source code** itself!

> The use essentially boils down to:
>
>  - **Identifying original source code** for a software part, to enable incremental patching and to confirm defective operation
>  - Identifying whether one part is **"compatible"** with another, or whether it can replace it

All of this is accomplished with a **private version** number. **Public version** is simply a marketing moniker, and it maps to one or more internal software parts, each having its own **private version**. (Just like Toyota Corolla 2011 [contains][toydiy] a ZRE142 frame and a 32000-12420 torque converter)

[vin]: http://en.wikipedia.org/wiki/Vehicle_identification_number
[jattver]: http://www.codinghorror.com/blog/2007/02/whats-in-a-version-number-anyway.html
[toydiy]: http://www.toyodiy.com/parts/p_U_2011_TOYOTA_COROLLA_ZRE142L-AEPDKA_3502.html

## Version Use

### Windows

In Windows you'd be pretty used to seeing a concept of a version number, it is supported by a layer of the operating system. Version numbers are embedded into all binary executable files, and can be seen when hovering over EXE/DLL in Windows Explorer, or when viewing Properties. In fact, any file that can have "resources" can have a version, since it is stored in the [VERSIONINFO][verinfo] resource.

It uses the common format we're all used to: **`major.minor.build.revision`** (e.g. "1.2.360.0"). It is important to note that each number is limited to 16-bit, and so cannot exceed 65535. This has certain implications on what we can represent with these numbers.

Note that label for these numbers are not strictly defined - they are simple 4 short integers. The first two are referred to as **major** and **minor** pretty unanimously. The last two is where we see some variation, depending on the versioning scheme.

This version is most prominently used during the Windows Update process, which utilizes Windows Installer (MSI) technology to update various parts of the system. Essentially, Windows Installer follows [certain rules][msifv] to determine whether the update it is installing is newer than what's already installer. If the version is greater, then it's ok to update.

### .NET

Naturally this concept flows to the .NET Framework, which was built around many existing Windows concepts. Thus we have the [`Version` class][verclass], which follows the 4 integer paradigm. We can also define `AssemblyVersionAttribute` and `AssemblyFileVersionAttribute`, which specify an assembly version and Windows version resource respectively. 

In .NET, assembly version exists separately from the underlying Windows `VERSIONINFO`-based version, which is what you see in Windows Explorer (or file Properties). It forms part of the assembly strong name, and is used exclusively by the .NET Framework when resolving assemblies. The two versions, assembly version and Windows file version, **can be different** - but more often they are the same, as it is much less confusing.

.NET uses version for **dependency tracking**, i.e. noting the versions of assemblies being referenced, thus making it obvious when an update breaks compatibility for application that depend on a particular library. This is a step forward from native Windows file version, which was only used during the update process, and not when referencing a library, leading to the infamous [**"DLL Hell"**][dllhell].

It is worth noting that .NET's `Version` allows 4 32-bit integers, while `AssemblyFileVersionAttribute` is limited to 16-bit, as it maps directly to `VERSIONINFO` resource. Thus, if we want `AssemblyVersionAttribute` and `AssemblyFileVersionAttribute` to be the same, this effectively **places a limit on assembly version** components as well.

[dllhell]: http://en.wikipedia.org/wiki/DLL_Hell

### Linux

Linux, in general, uses a [different method][linuxso] to address versioning. Binary files don't contain an embedded version stamp, like most Windows binaries do. Instead, a shared library filename indicates its version, e.g. `/usr/local/lib/mylib.so.1.5`.

A number of symbolic links are created, e.g. `mylib.so -> mylib.so.1` and `mylib.so.1 -> mylib.so.1.5`. An application can reference a library via symbolic link, like `mylib.so.1`, and get the latest **1.x** compatible version installed.

This works fairly well, as long as [everyone follows][linuxver] this [convention][linuxso]. Each library can then, in turn, load libraries it depends on in the similar manner.

Linux users would also be quite familiar with the popular "Advanced Package Tool", [apt-get][aptget], ubiquitously used on Debian-derived systems like Ubuntu. Being a true Package Manager it supports installing side-by-side versions and tracking dependencies between packages. We take a closer look at advantages of Package Managers in the following sections.  

[verinfo]: http://msdn.microsoft.com/en-us/library/windows/desktop/aa381058(v=vs.85).aspx
[libtool]: https://www.gnu.org/software/libtool/manual/libtool.html#Versioning
[msifv]: http://msdn.microsoft.com/en-us/library/aa368599(v=vs.85).aspx
[verclass]:http://msdn.microsoft.com/en-us/library/system.version(v=vs.110).aspx
[linuxso]: http://www.dwheeler.com/program-library/Program-Library-HOWTO/x36.html
[linuxver]: http://stackoverflow.com/questions/3839756/how-do-applications-resolve-to-different-versions-of-shared-libraries-at-run-tim

## Version Number Schemes

There are several popular version numbering schemes for software, but all of them are a variation of the same theme and share common traits. Having **major** and **minor** version components is the same across the board. What they represent is fairly consistent:

  - **Major** number increase: represents major breaking changes in the software system, often not backwards compatible, or addition of large amount of new functionality
  - **Minor** number increase: represents less substantial evolutionary changes, mainly updates or improvements in existing functionality, or addition of a smaller new feature set

Above is only one guideline - there are no set rules about what **major** and **minor** versions are supposed to represent. Only that they are supposed to increase as more features are added to the software with time.

Windows and .NET binaries specify a [4-part version scheme][msver]: **major . minor . build . revision**. The last two components are fairly free-form, there are many variations in what they represent - some use incremental build counters, some use date/time of the build, and some derive them from source control internal revision numbers. 

Many ignore **revision** number, and focus only on **build**. Windows Installer, for example, [only has 3 components][msiprodver]. If you want your version to span both binaries and the containing package, then it is best to limit yourself to just three numbers: **major**.**minor**.**build**.

In any case, the general pattern: the greater the version number, the more recent the software is.

A popular versioning scheme in recent years (especially among open source projects) has been dubbed **Semantic Versioning** (aka SemVer), and documented at [semver.org][semver]. It introduces a few other components, and makes version an alphanumeric string, rather than a pure number - opening a few interesting possibilities.

SemVer pattern is: **major**.**minor**.**patch**-**prerelease**+**metadata** (1.3.567-rc1+010814)

The first three components are the same as what we already discussed, with **patch** being optional. **Patch** is pretty much equivalent to the **build** component, but semantics can be different. [Semantic Versioning][semver] actually prescribes when each component should be incremented (based on "public API" changes). 

The **prerelease**, if specified, is an alphanumeric string that is used to tag a version as one that precedes the final release. For example, 1.3.567-rc1 will precede 1.3.567. This is useful to attach more meaning to the version label than by simply using numbers.

**Metadata** is another optional component, which allows further tagging of the version label (usually with a build timestamp), but it does not participate in version ordering, i.e. versions that only differ in **metadata** are considered the same.

**Prerelease** is useful with Package Managers like [NuGet][nuget], which treat them differently - they are considered unstable and are not visible to general public, unless explicitly requested. This allows releasing alpha/beta versions without affecting those relying on stable releases.

**Prerelease** tags can also be useful in the internal release flow when dealing with parallel hotfixes and private builds, as discussed later in this article.

[msver]: http://en.wikipedia.org/wiki/Microsoft_version_numbering
[semver]: http://semver.org/
[msiprodver]: http://msdn.microsoft.com/en-us/library/aa370859(v=vs.85).aspx

## Versioning Non-Binary Files

We covered how we can stamp a version on **binary files**. But what about **other files** comprising a software system - configuration files, images, documents, fonts, etc? How do you stamp a version on them?

What about web frameworks like ASP.NET (or Ruby, Node.js, Python, etc) where source files and pages can be modified in-place, and automatically updated? How can we patch a web system, i.e. update few target files, and still keep it versioned?

The answer is - **don't update individual files**! There is no way for you to keep a meaningful version number for your software application, if individual non-binary files can be updated ad-hoc as hotfixes. Update using a **package** instead.

Below we discuss the importance of a **package** and options for creating one.

## Importance of Build and Package

When you hear the term "build", normally **compilation** comes to mind - most compiled languages, such as C#, C++ or Java, have to be compiled into a binary before being able to be executed. And so **building** is commonly associated with the process of **compiling**.

But that's not an entire picture. Some languages or frameworks, such as Python or ASP.NET, don't strictly require compilation. They can be either interpreted, in Python's case, or compiled on-the-fly, in ASP.NET's case. What should a **build** do for these systems? How do you "build" a Python app?

That's why it is more helpful to think of **build** as an **assembly process**, or simply **packaging**. Just like a line of consumer goods, such as shoes, gets packaged before shipping to the stores, so does a software system, before being released.

**Package** concept is essential to **versioning**, because a package is a single collection of the pieces that comprise a software system, or part of it, and can therefore be identified, and **stamped with a version**. With a right Package Management system (which we look at in the next section), it can be deployed and updated, and specify dependencies on other packages.

Software today is never a single binary executable file - it is a collection of various binaries, libraries, documents, configuration files, images, and other resources. A **package** is what helps us group them together, version and release to the outside world.

A **package** doesn't have to be sophisticated, although it helps in some situations (e.g. databases). It can even be a simple ZIP file, that can contain version in the file name, or embedded as a text file. In fact, many open source projects do just that - the release is a ZIP or a .tar.gz archive.

The important thing is that a **package** is a single unit, that is released and updated at the same time, leading to consistency. It is common to have several packages, for example, representing "client" and "server" components, or any other logical grouping applicable to a software system. Each **package** can then be updated on its own.

## Packaging

Let's take a look at some of the common **packaging** methods, the versioning approach, and which application they are best suited for.

### Windows Installer

**Best Suited**: Complete Windows GUI Applications, Windows Services, or Drivers

The oldest, and for a long time the only recommended way, to install applications on a Windows platform. It has built-in versioning support and a sophisticated (some would say "complicated") [set of rules][msifilever] for determining when to update components. While a Windows Installer package (.msi) is a single file, in essence it is a collection of small logical components (down to single files) that can be updated independently.

Windows Installer will actually [check each individual file][msifilecheck] that is being installed, whether it has a version and whether the version is greater than a file with the same name already installed. That means it is important to version not just the installer package, but each file contained in it. It also means that **it is incredibly difficult to do downgrades** (i.e. rollbacks) with Windows Installer.

It is best suited for traditional Windows Applications (GUI, services, drivers) that are released to the public. It is, however, not the best choice for internally developed & distributed applications, any kind of Web applications, or database systems.

It was also used to deploy distributable libraries (native DLLs) and COM objects, but with today's focus on .NET, it is not the mechanism for distributing .NET libraries.

[msipatching]: http://msdn.microsoft.com/en-us/library/aa370579(v=vs.85).aspx
[msifilever]: http://msdn.microsoft.com/en-us/library/aa368599(v=vs.85).aspx
[msifilecheck]: http://msdn.microsoft.com/en-us/library/aa368267(v=vs.85).aspx

### Web Deploy

**Best Suited**: Web Applications (IIS, ASP.NET)

[Web Deploy][webdeploy] technology was specifically designed for deploying and synchronizing applications on Microsoft IIS web servers. IIS Web Farm replication uses Web Deploy commands and packages behind the scenes to synchronize sites across a set of servers. IIS Manager has an extension (enabled by installing Web Deploy) to "Import Application", which can install or update a web application using a Web Deploy zip package.

Its biggest disadvantage is that it can only be used for web applications on Microsoft IIS platform, and the limited mechanism for customizing installation. While it could be suited for simple web applications, it can quickly become frustrating for anything more sophisticated, i.e. variables, conditional logic, databases, etc.

In addition, it has **no inherent support for versioning**.

[webdeploy]: http://www.iis.net/downloads/microsoft/web-deploy

### Package Managers

**Best Suited**: Shared Libraries, Dependencies, Command-line Utilities

Package Managers are great for releasing and versioning shared components, and tracking dependencies between them. For example, if you have a shared library that you want others to use, then Package Manager allows you to publish multiple versions side-by-side, and for consumers of the library to reference the version they depend on. Package Managers can resolve all inter-package dependencies, and retrieve only the versions that are expected. In effect, Package Managers solve the ["DLL Hell"][dllhell] problem.

They are best used during development, to resolve library dependencies. However some Package Manager, like [Chocolatey][cinst] for Windows or [apt-get][aptget] for Ubuntu, are geared towards installing complete software.

Most importantly, Package Managers are **designed around the versioning concept**. So they are a perfect mechanism for distributing versioned software libraries.

For .NET we have [NuGet][nuget]. A lot of open-source libraries have been published to its online repository, and it is now the defacto standard for distributing 3rd party components. It is encouraged that every team [sets up][nugetown] **their own NuGet repository** to share and publish internally developed libraries in a versioned manner.

[NuGet][nuget] can even be used to release complete software systems - see next section.

Other development environments have their own - [npm][npm] for Node.js, [pip][pip] for Python, [gems][gems] for Ruby, [apt-get][aptget] on Linux. Package Managers have been proven to be extremely useful, and have exploded in popularity.

[nuget]: https://www.nuget.org/
[npm]: https://www.npmjs.org/
[pip]: https://pypi.python.org/pypi/pip
[gems]: http://rubygems.org/
[cinst]: https://chocolatey.org/
[aptget]: https://help.ubuntu.com/12.04/serverguide/apt-get.html
[nugetown]: http://docs.nuget.org/docs/creating-packages/hosting-your-own-nuget-feeds

### Octopus Deploy

**Best Suited**: Internally Developed & Deployed Software

Using [NuGet][nuget] as the packaging and versioning shell, it is similar to an installer, only driven by [PowerShell][powershell], meaning infinite flexibility in how the software is to be deployed. PowerShell has already great support for configuring Windows Services, IIS Web Applications, Scheduled Tasks, SQL Server, and more.

For internally developed and distributed software (i.e. for a company running home-grown software solutions) this is a perfect release management vehicle. Packages are versioned and pushed to a shared NuGet feed (e.g. a network share), from where [Octopus Deploy][octopus] can release and deploy each package into the appropriate environment.

NuGet here plays a role of the application package/container, with a version stamped on it. Package can be built once, and then deployed as many times as needed to whatever environment.

[octopus]: http://octopusdeploy.com/
[powershell]: http://technet.microsoft.com/en-us/library/bb978526.aspx

## Versioning & Packaging Databases

**Database versioning** is one of the biggest challenges in software projects. Almost every team I encountered, either completely ignored it or had something inadequate in place. It certainly presents a challenge - database systems **mix data/scheme definition with actual live data**, and there is no single "file" that can be effectively versioned.

We have to recognize the database as an integral part of the software system. One that executes on a proprietary 3rd-party platform (SQL Server, Oracle, PostgreSQL, etc), but the source of which is part of the software definition. It can be compared to **script-based** systems, such as Node.js or Python, only the scripts are written in a SQL dialect.

There are essentially three popular approaches to database versioning, that support automated deployments (I am not considering manual approaches, because they are error-prone, and have nothing to do with real versioning!).

### DB - Migrations

"Migrations" is a concept where developers keep a set of organized SQL script files, numbered sequentially, where each script applies modifications to the target DB to bring it to the expected state. Whenever a change is needed to the application database, a developer creates a new _migration_ script that applies the delta changes.

All of the scripts are kept as part of the source control, and are packaged with the application (either embedded into the executable binary, or installed along-side). A _migrations_ library then checks the target database for a dedicated table which holds the last "migration script number" applied, and then runs scripts with number greater than that, in order, effectively applying all of the changes in turn.

While this approach is simple to implement, and is favored among several popular frameworks ([Ruby Rails][railsmigrate], [Entity Framework][efcodefirstmigrate]), it has a number of **significant short-comings**. Firstly, there is **no single source view** of all database objects (i.e. tables, stored procedures, etc), they are sprinkled through the multiple _migration_ scripts. It is not clear which of the scripts contains which of the modifications. One has to "replay" them all to generate a database, and then look directly in the database (rather than source code).

Secondly, the _migration scripts number_ becomes the "version" of the database, which is different from the software package version number for the rest of the application. This is somewhat confusing. In addition, this "version" **does not really identify the state** of the database, since a database can be changed outside an application without updating the "version". This may potentially break future installs, because _migration scripts_ expect the database to be in a certain state to work.

Thirdly, developers **have to be disciplined enough** to follow the structure and apply ALL changes through _migration scripts_. Furthermore, when developing and debugging locally, one often has to go through several iterations before getting that table or store procedure change right. Yet only the final changes should make it into the _migration script_, meaning they have to be remembered and written manually. Otherwise, _migration scripts_ would contain all of the intermediate changes made by all developers on the project. It is easy to see how that can grow out of proportion quickly.

Finally, there is an argument that _migration scripts_ are a "history of changes", and it is a bit of a redundancy to store them in source control, which already IS a "history" of code changes. We would be storing **a history of a history**. There's something philosophical about that.

**Pros:**

  - Supported by some frameworks and libraries ([Rails][railsmigrate], [DbUp][dbup], [RoundHousE][rhdb], [EF Code First][efcodefirstmigrate])
  - Can work with any database
  - Potentially high degree of control over SQL scripts

**Cons:**

  - Have to manually maintain all migration scripts
  - Tracking changes through source control is difficult
  - Not robust against target database out-of-band changes

[rhdb]: https://github.com/chucknorris/roundhouse/wiki
[dbup]: http://dbup.github.io/
[railsmigrate]: http://guides.rubyonrails.org/migrations.html
[efcodefirstmigrate]: http://msdn.microsoft.com/en-us/data/jj591621.aspx

### DB - SQL Compare

Most often this is used in _manual_ approach, comparing a database between two environments (e.g. development vs test) to copy over the changes. We are considering an _automated_ approach, suitable for the packaging and versioning strategies being discussed.

In source control, database is represent by a series of _creation scripts_ (e.g. to create tables, stored procedures, triggers, etc), such that a new database with the right schema can be created from scratch. Usually each script file logically represents a corresponding object in the database, e.g. _Table1.sql_ would be the create script for _Table1_  table. All of the scripts are included in the released package (sometimes even combined into a large single create script, by concatenating them).

The idea is that during automated package deployment a temporary fresh database copy is created, by running all of the _creation scripts_, and then a SQL Compare tool is executed to compare the pristine copy with the target database to generate a _migration delta script_ on the fly.

The advantage of this approach is that it is robust against the target database out-of-band changes, since **_delta script_ is generated during _deployment_**, rather than during development. SQL Compare tools (such a [RedGate's SQLCompare][rgsqlcomp] or [XSQL Compare][xsqlcomp]) are sophisticated and mature enough tools that we can have some confidence in the generate SQL code. Each can be controlled by a multitude of options to fine-tune behavior with respect to renames, reordering columns, avoiding drops, etc.

In this case, target database is considered as a _runtime environment_, and we **avoid having the issue of _versioning_ it**. Instead we _version_ the package that contains all of the _creation scripts_, which is much easier, and use it to synchronize target database with what's expected in each version.

Big disadvantage of this approach is the difficulty of getting it right - there is no off-the-shelf framework that would support it, and it has to be developed. For SQL Server, read the next section for a better approach. For others, some day I may put together the set of scripts and logic necessary to achieve this, based on some of my prior work (unless someone else beats me to it).

[rgsqlcomp]: http://www.red-gate.com/products/sql-development/sql-compare/
[xsqlcomp]: http://www.xsql.com/products/sql_server_schema_compare/

**Pros:**

  - Automatically detect and migrate changes, regardless of target DB state
  - Only maintaining DDL (i.e. create) scripts in source control, meaning easy change tracking

**Cons:**

  - More difficult to setup, especially to be automated
  - Having to create a temporary database during each deployment (need "_create database_" permission)

### DB - DACPAC (SQL Server)

For SQL Server there is now a new recommended approach - DACPAC, and it can be produced by Visual Studio 2012 and above, if using SQL Server database project. Really this is a slick variation of the "SQL Compare" method above, just Microsoft has done all the heavy lifting for you!

Essentially, DACPAC is a zip package which contains an XML schema model of what the target database should look like. It is compiled by Visual Studio based on the _creation scripts_ in your project. In fact, it represents that _temporary pristine database_ that we would have had to create manually. Only it is done automatically and the schema represented in an XML format. The real bonus is that a **DACPAC can be _versioned_**, i.e. its metadata supports storing a version number.

[SQL Server Data Tools][ssdt] can be used to [deploy a DACPAC package][sqlpackage], which really performs a SQL Compare operation between the in-memory database model loaded from DACPAC and the target database. It does the **same thing as SQL Compare, but avoids having to create the extra temporary database copy** to do the comparison.

For applications having SQL Server as a back-end, a DACPAC can be included as one of the deployable packages, stamped with appropriate _version_ generated during the build. Starting with SQL Server 2008 R2, database can be [registered as a Data-Tier Application][regdac], and the latest DAC version is [tracked in a system view][sysdac] that can be queried.

**Pros:**

  - Can package the whole DB definition into a single package (or several packages)
  - Can apply the same version to the package as the rest of the software system
  - Same advantages as the SQL Compare method

**Cons:**

 - SQL Server only
 - Need to treat lookup data in a special way (post-deploy MERGE script)

[ssdt]: http://msdn.microsoft.com/en-us/jj650015
[sqlpackage]: http://msdn.microsoft.com/en-us/library/hh550080(v=vs.103).aspx
[regdac]: http://technet.microsoft.com/en-us/library/ee633653.aspx
[sysdac]: http://technet.microsoft.com/en-us/library/ee240830.aspx


## Build Auto-versioning

Given the importance of consistent versioning discussed above, it makes sense to implement a strategy for automatically generating and stamping a version number during the software automated build process. We want the version number to be applied to the produced packages, and also applied to all the binaries generated through compilation.

There are several well-known and not so well-known ways of achieving this. We look at pros and cons of each.

### Applying Build Number

There are some who prefer to update the version number manually just before a release. I will argue that this is a bad practice. Firstly, it is easy to forget to do it, if you don't have an automated system for incrementing the version build number. And, if it is easy to forget, it will be forgotten at some point.

Secondly, without automatically updating build number, there will be multiple packages produced from the source code that have the same version number, but different functionality (as more commits are made to the source control). This will be confusing to say the least.

It is better to have a process, like ones described below, where version number **build** component is **automatically updated** whenever a non-local build is made.

### Multiple Versions for Multiple Components

If there are multiple software components, where each needs to have its own version number, then it is best to split them each into its own separate build. Don't mix multiple version numbers in the same build, as it unnecessarily increases the complexity, and raises a question about which of the build numbers should be used to label the build itself (in addition to having to tag each source sub-tree separately).

Basically, **One build** = **One version**!

### Developer vs Continuous vs Release Builds

**Release** build is the one that will potentially be released to public or a particular environment - test, staging, production, etc. That's the build that needs to be consistently versioned to keep track of changes that are included and to link back to the source code at the time of compilation.

Note that **Release** builds can scheduled - it is popular to have a **Daily** or **Nightly** build. In most situations it should be the **Release** build, i.e. it should be versioned and packaged ready to be released.

**Continuous Integration** builds run whenever someone commits to the repository and are used to validate that the code compiles, and passes unit tests. There is no need to version this build, as it is not intended to be released. 

Developers must also be able to do a **Developer build**, whether it is to test/fix the build process itself, or to generate shared software components to be used in development. Such builds are intended to be run locally only and should never be publicly released.

You can default the **build** part of the version number to "0". This will identify **Developer** builds, i.e. ones that are not supposed to be released. For **Release** builds pass the build number to your build scripts as a property. Have MSBuild [stamp a version number][msbuildver] on all generated assemblies and packages. 

[msbuildver]: http://www.lionhack.com/2014/02/13/msbuild-override-assembly-version/

### Tagging Source Control

Since one of the primary reasons for having a version number is to be able to link back to source code used to build the software (see beginning of the article), it is important to create tags/labels in source control that identify the state of source code at the time that version was built.

Various systems call it differently - TFS has "Labels", Git has "tags". Tag should include the full version (including the build number) of the build, so that it can later be found, if needed.

### Build Number - Version File Auto Increment

Common technique is to record version number together with source code, usually in a separate file (e.g. "version.txt"). The build process then finds the file, reads the version, increments the build number portion, and commits the file back to repository.

If the commit message also includes the version number, e.g _"Auto-increment: 1.3.156.0"_, then it comes in handy when viewing commit history. You can see the changes that occurred between versions clearly by seeing the commits between the two _"Auto-increment: ..."_ messages.

This works fairly well, but has a few drawbacks. Mainly due to the fact that "version" becomes part of the source code. When merging changes between say release branch and main, you have to resort to **"cherry-picking"** (i.e. selecting just the code changesets) to avoid merging the modified version number. That requires being always careful, because you can accidentally change the versioning sequence of another branch just by merging the "version file" into it. 

**Pros:**

  - Control over the build number sequence (i.e. sequential)
  - Can make it easy to see changes between versions in source control history

**Cons:**

  - Difficult to control merging between code branches in source control

### Build Number - External

Overcoming the drawbacks of the auto increment approach, it is possible to track the build number outside of the source tree. Build server software such as CruiseControl.NET or TFS Builds can do that - they track a build number internally for each "project" and are able to pass it as a parameter to MSBuild.

Version file is still used, but it records **major** and **minor** versions only, and doesn't have to change between each build. This makes it easier to merge changes from release branches back to main and others, since they will contain only code changes, without being intermingled with version increments. **Major/minor** version changes would occur early in the development cycle, when starting work on the next update, and are already set by the time release branch is created.

**Pros:**

  - Not modifying source tree on every build makes merging between branches easier
  - Versioned builds are forced to be built by a dedicated build server

**Cons:**

  - Relies on a build system that can supply a build number (e.g. CruiseControl.NET, TFS Builds)
  - Changing build number sequence can be difficult (e.g. TFS Builds)

### Build Number - Derived from Date/Time

A popular alternative is to derive build number for the date/time of the build. The advantage being that it carries more meaning (useful in diagnosis), and each build inherently should get a different build number (with later builds getting a higher number).

The trick, of course, is fitting all this into a **16-bit** number, if using the standard 4-part Windows version number. While some solve it by using both, the **build** and **revision** components, I cannot recommend it, because **revision** cannot always be applied to external packages (like Windows Installer, or NuGet), which use only a 3-part version number.

Example: **9-bit** (months since Jan, 2000) + **5-bit** (day of the month) + **2-bit** (hour % 4)

_This only allows only 4 unique builds per day, which is not a lot, unless all you want is a **daily build**._

**Pros:**

  - Not depending on keeping track of the last build number
  - Build number can be given more meaning, if it derives from a date

**Cons:**

  - Build number is not sequential (but it increases nevertheless)
  - Limited to 16-bit (maximum 65535), so some overflow into **revision** (4th) number

### Build Number - Derived from Source Control

A variation of the previous technique is to derive build number from a unique property in source control. With a centralized SCM like Subversion or TFS, a **revision or changeset number** is an ever increasing number that is tied directly to the source code. The big problem with it is that it can quickly overflow the **16-bit** limit, meaning you may have to accept build numbers looping back to zero.

An alternative in distributed SCM, like Git, is to use the **size of the commit history log** as the **build** number. This will monotonously increase for any single branch, as new commits are made. It too can overflow the **16-bit** limit, but goes a lot further than the global revision number.

Example: `git rev-list HEAD --count`

**Pros:**

  - Not depending on keeping track of the last build number
  - No possibility of "forgetting" to update version file, or accidentally merge it to/from another branch

**Cons:**

  - Commit history size will grow beyond 65,535 at some point, overflowing the 16-bit **build** number

## Parallel Branches

It's no secret that developing for multiple versions requires [multiple branches][gitflow] in source control, each representing a "version" stream for the software. They can be roughly divided into:

  - **Development** branches - where unstable code for the next version lives, and where developers commit daily work
  - **Feature** branches - veering off from **development** branches, encorporating larger feature development, that would otherwise disrupt other team members
  - **Release** branches - representing versions of released software, or a release undergoing stabilization

Each **release** branch needs to have an identifying version, and is usually named after it, e.g. **"1.7"**. A decision of whether to create a new **release** branch depends on how long it is expected that it will be in stabilization mode before releasing, and whether concurrent live versions are permitted (i.e. for packaged software). If you need to be able to maintain & hotfix the current released version, while a new version is being tested & stabilized, then create a new branch.

**Development** and **feature** branches need to have a version number that is above any of the existing **release** branches to avoid confusion. For example, if a **1.7** release branch is created, for the upcoming 1.7 release, then immediately update **development** branch version sequence to **1.8**.

Versioning **feature** branches is more difficult, since you don't want to start a new versioning sequence for every **feature**. Nothing should be "released" from **feature** branches, so this version is for internal purposes only. If using [Semantic Versioning][semver], attach a **prerelease tag** to clearly indicate this is a version for a **feature** branch, e.g. **1.8.781-dev-feature-x**.

In any case, you wouldn't deploy anything built from a **feature** branch to the shared **testing** or **production** environment, or release a package from it. So it is acceptable to have version sequence overlap with that of **development** branch.

Finally, in the next section we look at how to version patches & hotfixes that are applied to **release** branches.

## Handling Patches / Hotfixes

Devising a system to handle patches depends heavily on the rest of the software development cycle, which is what many teams forget when searching for the "one, true way" of handling concurrent patching of the released/production software in parallel with working on the new version.

For example, having a short QA/test cycle, where most of the tests are automated, results in a more simplified and robust system, which does not have to deal with multiple parallel hotfixes "in test".

### Overlapping hotfixes

One difficulty that comes with managing parallel development is consistent versioning and deployment strategy that would overcome inherent conflicts. Consider following scenario: you have recently released a software package 1.5.167. Two urgent show-stopping issues have slipped past your QA process and now require a quick fix. You assign two developers to work on each one in parallel. How would they commit their fixes to minimize conflicts? How do you test each fix? How do you release one independent of the other?

This is a good example of the complexity of software release processes that can be encountered in real-world teams. It applies both to internal software and packaged software, but distribution of the hotfix might be slightly different for each one.

First, let's consider what happens if we **remove concurrency**. In the case where the two issues are worked **one after the other**, the solution becomes simple. The first fix gets committed into the **maintenance/hotfix** branch for **1.5** release stream, a new build is generated, with an incremented build number. Build goes through a **quick QA cycle** to make sure there is no regression, and then it is ready to be deployed. Same process repeats for the second fix.

The problem with **concurrent** approach is the time when development is in parallel, creating the entangled case where there is no **build/package** that contains only **one of the fixes**, i.e. independent of the other. This problem is magnified by a **slow QA cycle**, usually meaning there are no automated tests. While one fix is in test, if a commit for a second fix is made to the same branch, and a problem is discovered with the first one, it becomes very difficult to separate the two now.

The **culprit** here is, of course, the concept of a **partial fix** - the state where the fix is not complete. It has been committed, but has a problem with it, **requiring further commits**. This can easily create the case of a hotfix branch where the two fixes are "entangled" (quantum physics on the code level!).

**Solution** is to **remove possibility of a partial hotfix**.

This means that each hotfix has to be coded and tested in a separate code stream, independent of the other. Once tested, and ready for release, it is merged into the main hotfix release branch, where the automated build can create a new package and apply versioning (i.e. increment build number, for example, to 1.5.168).

Second hotfix, once tested, also has to be merged into the main hotfix release branch. But, because during the work on this second hotfix, the first hotfix got released, we first **merge the first hotfix into the second hotfix's branch**! This ensures that we can test how the second hotfix operates, when applied on top of the first hotfix, and merge any code conflicts, if any.

In the end, you want a system with **both hotfixes applied** - that is the "next" version. So it makes sense that whatever hotfix is "second", it is applied on top of the "first" one. And creating a packaged release from the single hotfix release branch ensures that the version number is consistently incremented for the whole system.

Of course, above means that we must create a separate branch for each hotfix. Some version control systems, namely [Git][git], make this very easy and part of the expected [developer workflow][gitflow]. If you are using a version control system like [TFS][tfs], then creating new branches for each hotfix is a bit more painful. In [TFS][tfs], I suggest using named Shelvesets feature to emulate Git's process, and perform initial QA tests for a hotfix from a Shelveset-branch build. Then commit Shelveset into the hotfix branch to build the official hotfix package (and perform necessary merging).

What about the **versioning of the interim hotfix builds**? The main hotfix release branch would have a standard versioning scheme applied (as discussed above), either incrementing a build number, or using a timestamp. Each new hotfix, applied on top of all previous hotfixes, gets an increased **build number**, and the software version keeps moving forward.

However, when building from the developer hotfix branch (or Shelveset in TFS), we also need to apply a version to distinguish it from other builds, and be able to deploy it into QA/test environment. We want to be able to test each hotfix in isolation, applied on top of an existing released version of the software system. This becomes **problematic, if you have a single test environment**.

You do not want to apply both hotfixes into one test environment, because there is no guarantee that they won't conflict or affect each other. If you are able to quickly spin up a test environment for a hotfix development branch, then you can truly parallelize team efforts. **For a shared test environment, they have to be applied one at a time**:

  1. Force install latest release version (e.g. 1.5.168) to bring environment to a known state
  2. Install the hotfix version to be tested
  3. Perform the tests (preferably automated)
     - For **shared test environnments** this is the bottleneck, since no other hotfixes can be tested at  the same time (automation can help minimize the time spent in this step)
  4. Repeat 1-3, until tests are satisfactory

What this means is that each hotfix has to have its **build version number** greater than the latest released version, the one it is being applied on top of. There are several ways to achieve that. If using a **derived build number**, this should just work out of the box. If **incrementing** or using **external** build numbers, then the easiest option is to simply force the build for hotfix development branch (or Shelveset) to use a number greater than latest released version (i.e. .168). 

With [Semantic Versioning][semver], we can setup hotfix builds to use a "prerelease" tag that clearly marks it as a hotfix-test build. For example - **1.5.169-check14761**, where the trailing number could be a reference to the issue tracking system. This works especially well when using NuGet as the packaging mechanism.

Once tested, the changes can be merged into hotfix release branch, and an official build generated, with incremented build version number.

**NOTE:** Above process to resolve concurrent hotfixes is undoubtedly complicated. It is intended to solve a particular real-world scenario, but one that does not happen too often. If there are no concurrent fixes expected, you can simplify your life by applying fixes directly to the hotfix release branch.

[git]: http://git-scm.com/
[tfs]: http://msdn.microsoft.com/en-us/vstudio/ff637362.aspx
[gitflow]: http://nvie.com/posts/a-successful-git-branching-model/

### Patching a large system

If applying hotfixes to a large system, we don't want to upgrade the whole thing, which may involve a lot of different components - services, GUI applications, scheduled jobs, databases, etc. Instead, we want to apply the fix only to affected parts.

This is where **splitting the system into multiple packages** helps. Each corresponds to a logically contained piece of the system - for example, each service, application, database, etc is its own package. That means they can be patched independently by **applying just that package**.

Care must be taken about dependencies, if hotfix affects multiple packages at once. Although, in that case, ask yourself is it really a hotfix or a new minor version?

### Patching for specific installation

Some software shops may have developed the practice of patching the software for individual customers (for packaged software), in other words creating a "custom" version for just that installation, without including this fix in the rest of released software streams. This is one of the worst situations to be in, with regards to versioning, since it creates a large number of variations that have to be maintained separately.

Instead, release a **general update**, moving the overall software version forward for that release stream. **Adopt a "feature" system**, where parts of the software can be turned on & off based on configuration. If a specific fix is needed for a particular installation, then that code can be encapsulated behind a configuration switch which turns this section of the code on or off. **That particular customer can turn it on**, while the rest can have it off!

This is also a popular technique in web applications, of which only one installation exists (on the server), where various "features" can be enabled **based on "configuration" for each user**, or a set of users.

### Patching the changes only

There is often the temptation to simply patch in the changes to the live/production system by editing/replacing one file, or updating one table or stored procedure. The change is small, and it seems like the fastest way to solve the imminent issue, without changing anything else in the system. 

**Please don't do it!**

While it seems like a smaller risk to make only the necessary updates directly, it makes it a whole lot harder to **know the state of the system** in the future. As more and more such "small" patches get applied, there is no longer any reliable way to link the running system back to the original source code, making **further maintenance exponentially more complicated** (and, ironically, increasing the risk).

Updating individual non-binary (e.g. config files) or altering database objects **does not update any version number**. That means it is difficult to tell which changes have been made to the system, leading to "maintenance hell" (a variation of the infamous ["DLL Hell"][dllhell]).

Rule of thumb: **Any change to the system should change the version number.**

_NOTE_: Windows Installer allows a so called ["small update"][msismallupdate], where product version number does not have to change, used for small hotfix patches. I believe this creates too much confusion, and so I do not recommend it. Windows Installer does track each patch, through package code, so you always know which patches have been applied. But it means now having to track and remove patches on subsequent product updates, which complicates the process. It may work for Microsoft Windows and Microsoft Office, but I wouldn't recommend using it for any system.

[msismallupdate]: http://msdn.microsoft.com/en-us/library/aa371855(v=vs.85).aspx

---

### Final words

This turned out to be a much longer article than I originally anticipated when I sat down to write about **versioning**. I am hoping it proves useful for software engineers out there looking for some guidance on how to apply these concepts in their own projects.

Still this seems like only a partial treatment of the topic.

Everything I wrote above has been learned through the painful process of trial & error over the years. If just a few readers have an "aha!" moment while reading this, then I have achieved my goal!