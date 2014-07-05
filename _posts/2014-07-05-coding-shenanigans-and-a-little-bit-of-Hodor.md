---
layout: post
title: Coding shenanigans and a little bit of Hodor
description: "How the eternal human desire to play a prank on someone can lead to fun weekend projects."
modified:
tags: [.net, c#, c++, keyboard, interception, prank, hodor]
comments: true
---

###Idea
Both me and my wife watch Game of Thrones weekly, although I do it with much less fidgeting, sighing and dissatisfied head shaking, since I haven't read the books. So, the other day, I saw Oatmeal's [comic](http://theoatmeal.com/quiz/got_character) on the subject and immediately a wicked thought moved into the apartment of my mind, made some popcorn, put on a pair of dirty old sweatpants and sat down in front of the TV. And when they do that, there's just no kicking them out. I started wondering if I can put together an app that will make my wife's laptop be able to type only the word "hodor", no matter which keys on the keyboard she presses or which application is currently active. Well, the specification sounded simple enough, but many a night had been lost on making just such specs come to life. Never the less, I thought I'd give it a try as a weekend project, the result can be seen on [github](https://github.com/drazmazen/Hodorizer), and here's the story:

###Research
I knew from the start that I wanted to do this in C#, but if I wanted to intercept everything that's being typed on the keyboard in every application, I had to get down to a level that is lower than .NET. So, that's where Windows API and [pinvoke](http://www.pinvoke.net/) comes into play. With a bit of googling, I figured out that I'll basically need four Windows API .dll exports:

__[SetWindowsHookEx](http://www.pinvoke.net/default.aspx/user32.setwindowshookex)__
: hooks up our delegate to an event

__[SendInput](http://www.pinvoke.net/default.aspx/user32.sendinput)__
: as the name says, sends input to the system

__[CallNextHookEx](http://www.pinvoke.net/default.aspx/user32.callnexthookex)__
: is called from the hook delegate to basically say: "proceed with this event as planned"

__[UnhookWindowsHookEx](http://www.pinvoke.net/default.aspx/user32.unhookwindowshookex)__
: you guessed it, unhooks our delegate from the event
 
###Proof of Concept
The first thing I decided to make was a small console application and the logic behind it was pretty simple and straightforward: we intercept the keypress using SetWindowsHookEx and block it, and then we substitute it with our own using SendInput. There is only one little problem. Our delegate is hooked up to be called every time a keypress event is caught. This means that that it will also be executed when we call SendInput. So, when the key is pressed, our handler gets called, which calls SendInput, which triggers our handler again and so on... To avoid this, we'll just use a flag to let our handler know that the keypress came from the SendInput method and not from the user. Most of the magic can be found in the Hodor.Libraries project, in the WinApiInputIntercept class:
{% highlight csharp %}
private IntPtr HookCallback(int nCode, IntPtr wParam, IntPtr lParam)
{
    if (_isFromSendInput)
    {
        _isFromSendInput = false;
        return CallNextHookEx(IntPtr.Zero, nCode, wParam, lParam);
    }
    else if (nCode >= 0 && wParam == (IntPtr)WM_KEYDOWN)
    {
        int vkCode = Marshal.ReadInt32(lParam);
        SendInput((int)_hodor[_index % 6]);
        _index++;
    }
    return (IntPtr)1;            
}
{% endhighlight %}

As for the SendInput:
{% highlight csharp %}
private static void SendInput(int nCode)
{            
    var pInputs = new[]
                    {        
                        new INPUT()
                        {
                            type = 1,
                            U = new InputUnion()
                            {
                                ki = new KEYBDINPUT()
                                {
                                    wScan = (ScanCodeShort)nCode,
                                    wVk = (VirtualKeyShort)nCode
                                }
                            }
    
                        }
                    };
    _isFromSendInput = true;
    SendInput((uint)pInputs.Length, pInputs, INPUT.Size);
}
{% endhighlight %}
I won't go into details about INPUT, KEYBDINPUT, ScanCodeShort, VirtualKeyShort etc. here, because they are structs and enums almost completely copy pasted from the [pinvoke](http://www.pinvoke.net/) site. Basically, the above code creates a keyboard input and sends it to the system. So, to sum up, when a keyboard event is detected, we handle it by suppressing it, selecting the appropriate letter from the string 'hodor ' and sending it to the system instead. 

Now that the logic is in the library project, the console app's main method looks very simple:
{% highlight csharp %}
static void Main(string[] args)
        {
            _hook = _interceptor.SetHook();
            Application.Run();
            _interceptor.Unhook(_hook);
        }
{% endhighlight %}
And, after I've built and started the console application, it worked! No matter which application was in focus, no matter which key I pressed, all I got on the screen was 'hodor hodor hodor...' . Great, but as soon as it worked, I got another evil idea.


###Plot Thickens
Now, I could have simply made the console application invisible, sneak over to the wife's laptop, start it there and then wait in the corner and giggle like a little girl, but no... How about if I remotely start and stop the hodorization from my computer. Now that would be something. So that would mean a server and a client app. OK, I can make that. A simple WPF client that invokes two methods, one to start and one to stop the hodorization. And on the server side, a WCF service will do just fine, and since all of the logic I need is already contained in the Library project, it should be a breeze.
And so it was. In the Hodor.Service project, you can find a WCF service library which implements two methods that we need:
{% highlight csharp %}
public void Hodorize()
{
    Task.Run(() =>
        {
            _hook = _interceptor.SetHook();
            Application.Run();
            _interceptor.Unhook(_hook);
        });            
}

public void Dehodorize()
{            
    Application.Exit();
}
{% endhighlight %}
As you can see, this code looks a lot like the console application, with two exceptions:

* Dehodorize method
* wrapping stuff in a Task.Run

Dehodorize is, I think, self-explanatory, but let's take a closer look at that _Task.Run()_. If you ever had a chance to work with async-await, you already understand what's going on here, but for anyone that's seeing this for the first time: this _Task.Run()_ is basically just running the code inside the curly braces in a separate thread. And why would we all of a sudden need threading here? Well, what the _Application.Run()_ method is saying to our program is "now sit tight here and wait and listen for system messages". And like that, it blocks the main thread of the app at that line. But, if we want our service to react when we send it a call to Dehodorize() we need it to not be blocked. So, we run that part in a separate thread and our main thread is free to wait for further calls from the client. And, since both threads still belong to the same process, when we receive a Dehodorize call, executing _Aplication.Exit()_ will tell our blocked thread to unblock and proceed execution, which will in turn unhook our handler from keyboard input. Sounds simple enough, right? Well, hosting the service library in Visual Studio's WCF service host and running it confirms that it works. Now comes the question of hosting the library in some kind of process.

###Problems
Here is where I ran into the first big problem. My first choice for hosting the service was a windows service. I can easily just install it on the laptop, make it automatically start every time it boots and my job would be done. And I did all that and realized that it simply doesn't work. The reason is very simple. In Windows 7, windows services do not have access to keyboard input events. The handler simply never gets called. Considering that this is how many keyloggers work, it's not an unsound logic, but forces me to host the WCF in a console application. And this introduces two new problems: the application needs to be invisible and it needs to be automatically started when the system is rebooted. The invisibility problem is easily solved. There are several ways to make a console app not be visible to a user, but my favourite is to, once a console project is created in visual studio, go to properties of the project and from the Output Type drop-down choose Windows Application instead of Console Application. Now, for the second problem, it's going to take a bit more effort. 

###Solutions
To be honest, the only viable solution I could come up with for starting the application on system boot was the task scheduler. It has the option to execute a process on boot, retry on fail and run it as administrator, but now the setup seems to be getting complicated. Lets see, after building the application, we need to copy it to the computer, check if it's working, go into task scheduler, fill out a bunch of options etc. So, the next solution, whose job was to try and simplify all the previous solutions was to make an installer. Imagine my surprise when I found out that Visual Studio 2013 does not have an installer project. Luckily for me, the community already made a lot of fuss about this and Microsoft responded by making it possible to add this functionality in the form of an [extension](http://blogs.msdn.com/b/visualstudio/archive/2014/04/17/visual-studio-installer-projects-extension.aspx). Creating the installer was a rather elementary process and I won't go into details here. So, the only remaining thing was scheduling the task on boot. Turns out, there is a library for that, too, the [Task Scheduler Managed Wrapper](http://taskscheduler.codeplex.com/). I made a little application that schedules our console service host to be started on user logon:
{% highlight csharp %}
using (TaskService ts = new TaskService())
{
    TaskDefinition td = ts.NewTask();
    td.RegistrationInfo.Description = "Hodorizer service startup";


    LogonTrigger lt = new LogonTrigger();
    lt.Enabled = true;
    lt.Id = Thread.CurrentPrincipal.Identity.Name;
    lt.UserId = Thread.CurrentPrincipal.Identity.Name;                
    td.Triggers.Add(lt);


    td.Principal.RunLevel = TaskRunLevel.Highest;
    td.Principal.UserId = Thread.CurrentPrincipal.Identity.Name;
    td.Principal.LogonType = TaskLogonType.InteractiveToken;
    td.Settings.AllowDemandStart = true;
    td.Settings.Enabled = true;                
    td.Settings.StartWhenAvailable = true;
    td.Settings.MultipleInstances = TaskInstancesPolicy.IgnoreNew;
    td.Settings.RestartInterval = new TimeSpan(0, 5, 0);
    td.Settings.RestartCount = 3;
    

    //create path 
    var pathToExecutable = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "Hodor.ConsoleServiceHost.exe");

    // Create an action that will launch Notepad whenever the trigger fires
    td.Actions.Add(new ExecAction(pathToExecutable));

    // Register the task in the root folder
    ts.RootFolder.RegisterTaskDefinition("Hodorizer Service Starter", td);                
}
{% endhighlight %}
And all that was left was to add this scheduler to be executed in the installer.

###Epilogue
I had some fun switching hodorization on and off on my wife's laptop as she was thinking "How did I accidentally type 'hodor' again?!?", but truth be told, I had much more fun making the Hodorizer. I guess it was also a good example of how a one line specification can suddenly grow to include several projects - a service library, a service host, a client app, a scheduler and an installer. All in all, I thought it made a nice weekend project and a cool story to tell my grandchildren. Well, OK, a nice weekend project.
