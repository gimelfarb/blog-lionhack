# Dual-mode NT Service

Developing and debugging NT/Windows Service in .NET can be challenging, because by default the resulting EXE will not run unless it is **installed** as a Windows Service. This is inconvenient and breaks the usual quick write-debug(F5)-fix development cycle. One has to attach to the running Windows Service to debug, or use `Debugger.Break()` hacks to halt execution.

In this post I will show you how to achieve a good development workflow in Visual Studio when developing Windows Services, and be able to start/debug them with a simple F5.

## Traditional workflow

When you create a "Windows Service" project in Visual Studio, by default, the code in the Main() method resembles something like this:

<pre><code class="language-clike">
static void Main()
{
    ServiceBase[] ServicesToRun;
    ServicesToRun = new ServiceBase[] 
    { 
        new Service1() 
    };
    ServiceBase.Run(ServicesToRun);
}
</code></pre>

Trying to run the project (F5) results in a message box pop up:

> Cannot start service from command line or a debugger. A Windows Service must first be installed (using installutil.exe) and then started with ServerExplorer, Windows Services Administrative tool or NET START command.

Disappointing, at the very least. Now you have to drop to command-line to register it as a Windows Service first:

    > sc create Service1 binPath= "<fully qualified path to EXE>"

Then to debug, you'd have to start it first (e.g. via `> sc start Service1`), then attach debugger to the running process. You will have to restart Visual Studio as an Administrator for this.

If you make a correction in the code and try to recompile, the build might fail saying it cannot copy "EXE" file to build output. You'll have to **stop** the Windows Service first, to unlock the file, and only then compile the code, and **restart** the service again.

If you wanted to debug the service start-up code (i.e. because it fails when getting started), then there are a few more hoops to jump through. You'd have to insert `System.Diagnostics.Debugger.Launch();` line at the start of Main() method, which will cause process to halt and ask for debugger to attach (note that Debugger.Break doesn't work here).

If all of the above doesn't sound painful then either you're a masochist, or you just haven't spend much time developing and debugging a Windows Service.

## Better way

A common technique to solve this madness and make debugging for Windows Services a much more pleasant experience is to allow EXE to start in command-line mode, i.e. as a Console Application, rather than as a Windows Service. This way you can start it via F5 in Visual Studio and debug away.

Many people [do this via a command-line switch][cmdswitch], which, if provided, causes application to start as a Console app, rather than Windows Service. All you have to do is switch project type to "**Console Application**", and add code to Main() method:

<pre><code class="language-clike">
static void Main(string[] args)
{
    if (args.Length > 0 && args[0] == "/console") {
        // Console version
    } else {
        // Windows Service version
        ...
        ServiceBase.Run(ServicesToRun);
    }
}
</code></pre>

The nice thing about being able to run in "command-line mode" is that you can always start the application by double-clicking it and have it run with diagnostic output sent to standard output (i.e. console screen). This is great for diagnostic runs on test and production systems.

**Main problem** with this approach is that you have **two code paths to maintain** - one for Windows Services and one for "console mode". Still there is an issue in debugging problems that occur when initializing in Windows Services mode.

[cmdswitch]: http://stackoverflow.com/questions/809082/create-a-combo-command-line-windows-service-app

## Combining two modes

Resolving the two code paths issue is simple - make both modes [run through the same piece of logic][simulate]. The idea is that we don't modify any of our code, and have the "console mode" simulate starting the Windows Service by instantiating our `ServiceBase`-derived class.

<pre><code class="language-clike">
static void Main(string[] args)
{
    ServiceBase[] ServicesToRun;
    ServicesToRun = new ServiceBase[] 
    { 
        new Service1() 
    };

    if (args.Length > 0 && args[0] == "/console") {
        // Console version
        ConsoleRun(ServicesToRun);
    } else {
        // Windows Service version
        ServiceBase.Run(ServicesToRun);
    }
}
static void ConsoleRun(ServiceBase[] services)
{
    var method = typeof(ServiceBase).GetMethod("OnStart", BindingFlags.Instance | BindingFlags.NonPublic);
    foreach (var service in services)
    {
        Console.WriteLine("Starting {0} ...", service.ServiceName);
        method.Invoke(service, new object[] { Environment.GetCommandLineArgs() });
    }
    Console.WriteLine("Service(s) running - Press ESC to exit");
    ConsoleKeyInfo key;
    while (true)
    {
        // Loop until 'ESC' is pressed
        key = Console.ReadKey(true);
        if (key.Key == ConsoleKey.Escape) break;
    }
    method = typeof(ServiceBase).GetMethod("OnStop", BindingFlags.Instance | BindingFlags.NonPublic);
    foreach (var service in services)
    {
        method.Invoke(service, null);
    }
}
</code></pre>

Now running through Visual Studio F5, or from command-line, this will simulate starting of a Windows Service, and will run the `OnStart()` method. Program can be stopped at any time by hitting ESC key, and it'll simulate calling `OnStop()` method, as if Windows Service was stopped manually.

The code still has a few issues with it. Firstly, there is no way for the program itself to stop in "console mode". If it is running as a Windows Service, we can always issue `ServiceBase.Stop()`, which will gracefully terminate the service.

Why would we want to do that? For example, the main thread encountered an exception it cannot recover from, and swallowing it would leave Windows Service in a "zombie state", i.e. it appears as if it is running, but it is not doing anything!

Secondly, we still require a command-line switch to tell our code to go into "console mode". We always have to drop into command prompt to execute it - double-clicking the EXE will result in the above-mentioned error message that program must be installed as a Windows Service. 

[simulate]: http://stackoverflow.com/questions/125964/easier-way-to-start-debugging-a-windows-service-in-c-sharp/10838170#10838170

## Getting better - auto-detecting the mode

![What If I Told You](http://i3.kym-cdn.com/photos/images/original/000/285/844/6b7.png)

What if I told you - you can auto-detect the mode in which application is executing in? Would you take [the red or the blue pill][redbluepill]? :)

Ok, so you took the red pill, like a good hacker you are. It is possible to use lesser known `Environment.UserInteractive` property to detect whether application is running in an interactive session (i.e. "console mode"), or in the background (i.e. "service mode"). That way you don't have to fiddle with command-line switches - application will just auto-detect it's mode.

<pre><code class="language-clike">
static void Main(string[] args)
{
    ServiceBase[] ServicesToRun;
    ServicesToRun = new ServiceBase[] 
    { 
        new Service1() 
    };

    if (Environment.UserInteractive) {
        // Console version
        ConsoleRun(ServicesToRun);
    } else {
        // Windows Service version
        ServiceBase.Run(ServicesToRun);
    }
}
</code></pre>
 
You can now just double-click the EXE and it will start in "console mode" automatically. Also running from Visual Studio now doesn't require passing any command-line parameters.

> **NOTE**: In the odd case where you are developing a Windows Service which will have ["Interact with Desktop"][interact] option turned on, there is a [more sophisticated way][detectservice] to auto-detect service mode.

[redbluepill]: http://en.wikipedia.org/wiki/Red_pill_and_blue_pill
[userinteractive]: http://stackoverflow.com/questions/125964/easier-way-to-start-debugging-a-windows-service-in-c-sharp/126163#126163
[interact]: http://lostechies.com/keithdahlby/2011/08/13/allowing-a-windows-service-to-interact-with-desktop-without-localsystem/
[detectservice]: http://stackoverflow.com/questions/2397162/how-to-determine-if-starting-inside-a-windows-service/17557099#17557099

## Even better - stopping when asked

If application issues a call to `ServiceBase.Stop()` method, then the expectation is that the service will gracefully shutdown. With the presented code in "console mode" this won't happen, because application is locked in the infinite loop waiting for the ESC keypress.

`ServiceBase.Stop()` will in turn call overridden `OnStop()` method of your class, where normally you'd implement application clean up logic for all the entities allocating during `OnStart()`.

Easiest way to make sure application is stopped at this point, in "console mode", is to create a custom base class for your Windows Services, which overrides `OnStop()` method:

<pre><code class="language-clike">
public class DualServiceBase : ServiceBase {
    protected override void OnStop() {
        if (Environment.UserInteractive && this.ServiceHandle == IntPtr.Zero)
        {
            Console.WriteLine("Service terminated - press any key to exit");
            Console.ReadKey(true);
            Environment.Exit(0);
        }
    }
}
</code></pre>

We have to check `ServiceHandle` for a case when running in "service mode", but with access to interactive desktop.

Now we just change our main Service class to inherit from `DualServiceBase` and we're set!

## Putting it all together

We can now put all of this together into a single class `DualServiceBase`, that should probably go into a common library, to be reused by any Windows Service project you're developing. Just follow these steps:

  - Make sure project type is set to **Console Application**
  - Modify `Main()` method to use `DualServiceBase.Run`
  - Change the base class from `ServiceBase` to `DualServiceBase`

<pre><code class="language-clike">
static void Main(string[] args)
{
    var ServicesToRun = new DualServiceBase[] 
    { 
        new Service1() 
    };

    DualServiceBase.Run(ServicesToRun);
}

public class DualServiceBase : ServiceBase 
{
    protected override void OnStop() 
    {
        if (Environment.UserInteractive && this.ServiceHandle == IntPtr.Zero)
        {
            Console.WriteLine("Service terminated - press any key to exit");
            Console.ReadKey(true);
            Environment.Exit(0);
        }
    }

    public static void Run(DualServiceBase[] services) {
        if (Environment.UserInteractive) {
            // Console version
            ConsoleRun(services);
        } else {
            // Windows Service version
            ServiceBase.Run(services);
        }
    }

    static void ConsoleRun(DualServiceBase[] services)
    {
        // NOTE: Don't need Reflection to call OnStart/OnStop anymore
        // because we are inside DualServiceBase class definition!
 
        foreach (var service in services)
        {
            Console.WriteLine("Starting {0} ...", service.ServiceName);
            service.OnStart(Environment.GetCommandLineArgs());
        }
        Console.WriteLine("Service(s) running - Press ESC to exit");
        ConsoleKeyInfo key;
        while (true)
        {
            // Loop until 'ESC' is pressed
            key = Console.ReadKey(true);
            if (key.Key == ConsoleKey.Escape) break;
        }
        foreach (var service in services)
        {
            service.OnStop();
        }
    }
}
</code></pre>

There you go. Enjoy a cleaner, more productive debugging experience of your Windows Service projects!