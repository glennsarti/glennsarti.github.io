---
title: Presentation - How to become a SHiPS wright
excerpt: "How to become a SHiPS wright - Building with SHiPS"
category:
  - presentation
header:
  overlay_image: /images/header-pssummit-2019.png
  overlay_filter: 0.75
  teaser: /images/teaser-pssummit2019-ships.png
tags:
  - powershell
  - summit
  - 2019
  - ships
read_time: true
modified: 2019-05-20
---

### [PowerShell Summit North America 2019](https://powershell.org/summit/)

A Shipwright an artisan skilled in one or more of the tasks required to build vessels. A SHiPSwright is an artisan skilled in one or more of the tasks required to build PowerShell Providers. The SHiPS toolkit has been around for a while but it can be a little difficult to get started.

---

[Event details](https://app.socio.events/MjQ4Nw/agenda/14445/session/61497)

[Presentation](https://speakerdeck.com/glennsarti/how-to-become-a-ships-wright-building-with-ships)

[Recording](https://www.youtube.com/watch?v=iX62wii_r6g)

## Reviews

<blockquote class="twitter-tweet" data-cards="hidden" data-lang="en"><p lang="en" dir="ltr"><a href="https://twitter.com/GlennSarti?ref_src=twsrc%5Etfw">@GlennSarti</a> I&#39;ve been meaning to look into SHiPS! This really helped!!!<a href="https://t.co/nV9VhVHTdz">https://t.co/nV9VhVHTdz</a></p>&mdash; Irwin Strachan (@IrwinStrachan) <a href="https://twitter.com/IrwinStrachan/status/1130043257612840960?ref_src=twsrc%5Etfw">May 19, 2019</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## Resources

Ravikanth Chaganti
> PS Conf EU 2018 - SHiPS: Walk-through a bare-metal system configuration

[https://github.com/psconfeu/2018/tree/master/Ravikanth%20Chaganti/SHiPS](https://github.com/psconfeu/2018/tree/master/Ravikanth%20Chaganti/SHiPS)

SHiPS GH Repo

[https://github.com/PowerShell/SHiPS](https://github.com/PowerShell/SHiPS)

SHiPS PS Gallery

[https://www.powershellgallery.com/packages/SHiPS](https://www.powershellgallery.com/packages/SHiPS0)

[https://www.powershellgallery.com/packages?q=ships](https://www.powershellgallery.com/packages?q=ships)

about_format.ps1xml

[https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_format.ps1xml?view=powershell-5.1](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_format.ps1xml?view=powershell-5.1)


Writing a PowerShell Formatting File

[https://docs.microsoft.com/en-us/powershell/developer/format/writing-a-powershell-formatting-file](https://docs.microsoft.com/en-us/powershell/developer/format/writing-a-powershell-formatting-file)

SHiPS Default formatting

[https://github.com/PowerShell/SHiPS/blob/master/src/Modules/SHiPS.formats.ps1xml](https://github.com/PowerShell/SHiPS/blob/master/src/Modules/SHiPS.formats.ps1xml)

Images

[https://unsplash.com](https://unsplash.com)


## Transcript

_This is an approximate transcript of the presentation_

``` text
I tell this tale, which is stricter true,
Just by way of convincing you
How very little since things was made
that things have altered in the shipwright's trade.

In Blackwall Basin yesterday
A China barque re-fitting lay,
When an old shipwright with snow-white hair
Came up to watch us working there.

Now there wasn't a knot which the riggers knew
But the old man made it—and better too;
Nor there wasn't a sheet, or a lift, or a brace,
But the old man knew its lead and place.
```

Paraphrased from [Rudyard Kipling](https://mainlynorfolk.info/peter.bellamy/songs/thebricklayerandtheshipwright.html)

Shipwrights are an artisan skilled in the tasks required to build vessels. They were highly sought after back in the days of wooden ships. A little less now with modern ship fabrication techniques. But still, they were very, very highly skilled

However there is a certain amount of irony. That song was from Rudyard Kipling back in 1910 and ends with;
> How very little, since things was made,
> Anything alters in any one's trade !

Now we're probably all in the IT trade and I'm pretty sure there's a whole lot altering in our trade!  So today I'm going to help you navigate through the rough waters of building with SHiPS
The Simple Hierarchical in PowerShell provider!

So we'll start the obvious question "What is SHiPS"? Obviously NOT boats.

ShiPS is a powershell module which makes it easier to develop PowerShell Providers.  Which then begs the question what is a PowerShell provider? PowerShell Providers provide users access to the things that normally be difficult get at via the command line, for example, The Windows Certificate Store.  And then presents them in a consistent known format, a filesystem. Providers have been around for a long time, since PowerShell 1 and you already using them, you just may not know.

You can very quickly see what providers are available by running the Get-PSProvider command. And then use file system like commands; for example Get-ChildItem on the HKEY Local Machine System registry key. And you can use the other regular commands like Get and Set Location, New-Item, Remove-Item, Move-Item and so on.

Ok so Providers have been around a long time and we all use them.  Why would I need SHiPS then? Providers are great.  They hide so much of the complexity and difficulty dealing with things like the Windows Certificate Store or SCCM.  You just create a Drive and start browsing. They're so great, that many people start going, "I want to write my own provider" So surely there's plenty of help, documentation and examples in the community use?

Yeah, not so much ...

Writing providers is hard.  Here's an example. The Microsoft Documentation has a quickstart which is nice but the first things you need to do are;

- Install Visual Studio
- Install the PowerShell SDK
- Create a class library – What's that?
- Create a project reference – Nope no idea
- And start PowerShell with a command line that uses reflection; whatever that is!

That is an immediate barrier for most PowerShell users. Most of us don't even use Visual Studio

Next, you need to have a solid understanding of C# and the tooling that goes with it. Again, most of us don't have that

And finally you need to good understanding of the inner-workings of PowerShell and Providers using the PowerShell SDK. Oh also,  which versions of PowerShell will you target? Are you going to support non-windows platforms like Mac or Linux? To give you an idea of the size of the problem here, I went through the PowerShell codebase looking at the standard providers ...

On the left is the Provider and on the right is the number of lines of code for that provider. So for example, the Registry provider has 4,373 lines of code. But look at the Variables provider.  Now some people may be going, 239 lines, that doesn't seem all that much. But if you take into account all of the inherited classes just to create a basic provider, that number jumps up considerably. That's a LOT of C# code, well over 3000 lines you need to read and understand in order to make what we would consider "simple" provider. And this is where SHiPS comes in....

What SHiPS does is hide of all that really complex C# code away from you and simplifies developing providers. It also lets you create, just the bits you need, in a language familiar to you; PowerShell. Ok so now we understand why we would use SHiPS and what problems it solves. But if providers have been around for a while, then when did SHiPS become a thing?

The history of SHiPS starts back in 2014, so this has been 5 years in the making!  (Now some of this I had to reverse engineer, so hopefully it's all true). Back in 2014 a community member called Jim Christopher (Beefarino) created a module called Simplex which is "A powershell module used to create powershell providers using a simple DSL". So in a way Simplex is the pre-cursor to SHiPS.

Not long after he also released P2F, the PowerShell Provider Framework which he describes as "The PowerShell Provider Framework (P2F) performs the heavy lifting for developing PowerShell Providers.". From what I gather he took Simplex and removed the DSL part, which resulted in a framework which anyone could use to create any provider in C#.

And then sometime in 2017 the PowerShell team was working on the Azure Cloud Shell, which needed a Provider.  They ended up creating SHiPS, using P2F, to create a framework so you could create providers in PowerShell script, instead of C#.  In October Cloud Shell went public preview and SHiPS went open source not long after in September. Since then there have been multiple releases with the latest being 0.8.1.

This history is important because it sets the scene for how SHiPS works and how you write modules for it. What you see here is the architecture of SHiPS: You, the module author, writes PowerShell Classes which SHiPS uses to interact with P2F which interacts with PowerShell.cAll of that complexity below that dashed line is hidden away from you by SHiPS.

SHiPS makes it so easy, I can create a Provider which you can mount, and traverse like a file system in 16 lines of PowerShell (not 3000+).  This is working code right here. I'll be going through how to write a SHiPS module later on, but this is how small a provider can be.

So now I know more about SHiPS but what can _I_ do with it?  What things can I SHIPSify? There are quite a few SHiPS modules out in the wild and here are just a sample:

- Deepak has a DHCP Server drive

- I have a Puppet drive which can browse a Puppet Master server. I also have a text adventure game called Pirate Booty which is a SHiPS drive

- Ravi has a bunch including browsing the PS Gallery or Eventlog as a drive. He even has a SHiPS module to browse other SHiPS modules

- Patrick has a drive to browse the Abstract Syntax Tree of PowerShell scripts

- And of course the powershell team at the bottom there has the Azure Drive and CIM Drive

But that doesn't really help you. what can YOU SHiPSify? Working for Puppet it provided me with the perfect answer! Our company logo. This is a heart shaped DAG and you can "ships-ify" anything that can be represented as a DAG.

(And if you're from Australia or happen to be a sheep farmer, it's not those kinds of dags!)

The kind of DAG I'm talking about is a Directed Acyclical Graph. Let's break this down.

What is a graph? A graph in this context is made up of vertices, nodes, or points which are connected by edges, arcs, or lines. So the nodes are the yellow squares and the connections, or edges, between them are the white lines.

What does directed mean? It means that the connections between nodes has a direction. Note that the connections can't be bidirectional, with an arrow at both ends.  That would be two separate edges in two different directions. It also makes it easier to traverse the graph. For example let's say we want to get from Node A to node C, then the path we need to take is A then B then C.

What does acyclical mean? It means there are no cycles or circular references in the directed graph.  In this example there are no cycles so it is Acyclical. But if we do this; This causes a cycle between A, B and C that goes on forever. But you _can_ do this though. This creates two paths to get to C, but there is no cycle.

So that's a brief introduction to graph theory and DAGs but what does that have to do with real life.  What does a DAG look like out in the wild?

Well the FileSystem and Registry ...  They are DAGs – Nodes connected by edges. AD Org units and certificate store - DAGs . Azure Resource Groups - DAGs. Azure DevOps Pipelines - DAGs. The AWS CloudFormation designer displays your templates as a DAG!

Org units, networks, language, CMDBs, Application menus ... The list is HUGE.  DAGs are everywhere once you start looking! But ... So what?  Well, you can take a DAG and define it in PowerShell classes, which is then used by SHiPS as a provider. SHiPS just calls nodes and edges by different names.

This is a DAG representation of the SHiPS example documentation And what I've done is assigned each node and edge with the name you need to use in SHiPS. Nodes with children are called SHiPSDirectory and nodes without children are called SHiPSLeaf. The links between nodes come from the GetChildItem function. We'll cover this in more detail in the demo.

Also as a side-note, I'm not sure why the designers used mixed the metaphors here.  Surely it should be Parent and Child, or Branch and Leaf.  Not Directory and Leaf.

---

So let's build a SHiPS module.  We'll build a module which we can use to see the agenda for PowerShell Summit. Before I jump into this, you should have some basic knowledge of how to write PowerShell modules and be able to read PowerShell classes.  Again I'll have some links at the end of this talk if you want to read more about this. Each step that I go through is in my github repo for this talk. So if your listening to the recording, you can follow along! Here's a quick demo of what we are going to build ...

_DEMO_

The first thing to do is close your computer and get out a pen and paper. We need to plan what the DAG will look like. In this case the Summit has speakers and sessions.  And sessions happen on different times of the day. So I drew a quick diagram. And this is pretty much what you saw in the module demo. It may be a bit hard to see on the screen so I cleaned it up a but We have the speakers on the right listed under the speakers directory In the Agenda we can either list All sessions, or easily select only a particular day's sessions. Now we have an idea of what we're going to create, it's time to start some PowerShell!

You'll need the SHiPS module.  You can either install it via the PowerShell Gallery or go to that link for instructions on how to build it yourself.

The first thing I created was the module manifest and the root object.  This was was just enough code so I could import the module and create a new PS Drive. If you remember the DAG, this is the root object up the top left. So here's a cutdown version of the Module Manifest.  The only important bit here is the `RequiredModules` section. Next we create the module script.  Much like that other example I showed you earlier, the amount of code you need to just create a SHiPS drive is very small. At the top we have the using statement.  This is a "feature" of PowerShell classes and without it SHiPS won't work.

Next we define the root object for the drive.  You can pretty much call it any name you want but try and keep it small and simple. Note that it inherits from the SHiPSDirectory object.  Remembering back to what I said earlier.  Anything that has child objects is a Directory.  Anything that doesn't have child objects is a Leaf. Next is the constructor for the class.  It takes a single parameter called name.  And we then pass that to the base class to process. This is mandatory for all SHiPS objects. Even if the constructor does nothing like this one, you still need to define it.  In later steps I use more complicated constructors.

And lastly the GetChildItem method.  This gets called when user wants to get the children of this object.  Self explanatory I hope.  For now this just returns an empty array but we will add to this later. This is now enough code that we can now import the module and mount the PS Drive. Some people may have noticed this line at the top.  For the moment don't worry about it.  SHiPS has a caching ability which I will talk about later.

So let's try using this. What we want is to import the module, create a new PS-Drive and then see that there are no child items. So we import the module. Next thing we do is create a new powershell drive.  I've wrapped this over multiple lines so you can read it easier.

So we create a drive called Summit2019, with a provider called SHiPS.  The Root parameter is a little trickier.  It has the name of your module, then a hash, and then the name of the root object. So in our case the module name is PSSummitNA2019 and then name of the root object is Summit2019.  You could call your class root or something else. And the we get the child items and there's nothing, which is exactly what we expected.

So now we have a drive, time to create some directories. So we'll create the Speakers and Agenda directories first. So first we add a basic Directory called 'Speakers'.  Remember the constructor must always call `base` with a name. And then an empty child list in `GetChildItem`. And a similar directory called 'Agenda'. Now we have the two child directories, Speakers and Agenda, we need to modify the root object to create the them when GetChildItem is called. So what we do is create an instance of the Speakers object and pass in the name of the child. This new object is then added the to array `$obj`. Then we do the same for the Agenda object. So we should now have GetChildItem return an array with two items in it. Let's try it.

So like before, we import the module and create a new PS Drive. And when we get the child item we got two items called Speakers and Agenda.

Ok this is nice, but not really useful. Now it's time to actually display something useful.  Let's display the speaker information. We'll be creating these parts of the DAG next. So we need the speaker information.  Fortunately, the web app for the Summit accidentally publishes the entire speaker list as a JSON file.  What I did is download this file and save it in a directory that the module could find. Once I had the data file, I created some private helper functions.  `Get-SpeakerObject` and `Remove-HTML` to help read and parse the information.

* `Get-SpeakerObject` reads the JSON file and converts it into PowerShell custom objects

* `Remove-HTML` is used to strip HTML tags from text as unfortunately the data from the app is a combination of raw text, markdown and HTML markup

Now we can create the Speaker object.  Note that this is SHiPSLeaf not SHiPS directory because it does not have child objects. Next we define the public properties for this object.  A Speaker has a Name, Firstname, Lastname and a Bio.  Now when we call Get-ChildItem all of these properties will appear, not just the Name which is the default. The constructor for Speaker is a little different.  We have the same `$name` like the directories before, but now we have an extra parameter called data. We can add additional parameters to the constructor however we must always pass "something" back the base object, in this case the parameter called name. In the constructor we call the `PopulateFromData` method which parses the data object and extracts the information we need to populate the public properties, Firstname, Lastname etc.

Note we use the `Remove-HTML` helper function for Bio. Now we could've stuck all the code in the `PopulateFromData` method directly in the constructor and that's also fine.  I just wanted to show you can create private functions within the class.  We could've also put this in a normal PowerShell function as well.  And you'll see why this can be useful when I talk about testing later on.

Now we have a speaker object, we can modify the Speakers directory. So instead of an empty list, for each item in the JSON list, Create a new Speaker object and pass in the name and the JSON data. And this is why I have that additional data parameter in the constructor.  I already have all of the speaker data right here, so why not just pass it into the Speaker object when I create it.  Otherwise I'd need to parse the JSON file again EVERYTIME a speaker is created which is completely un-necessary.

So let's so this in action ... I'll skip over importing the module and creating the PS Drive. Now when when get the child items we have the speakers and if we expand all of the properties of a speaker you can see the Public properties we created, Name, Firstname Bio and so on.

So hopefully you're seeing a pattern here how I create this module.

- Create a small thing

- Test that it works

- Expand on that small thing

- Test that it works

- And repeat.

So let's expand this further.  Time to add the agenda information. We'll be creating the objects under the Agenda. So how do we get the information? Just like the session JSON files, I also needed the agenda JSON file which the PS Summit app also has.  So again, I downloaded the file and created some helper functions to parse it.

* `Get-SessionsObject` which, like the `Get-SpeakersObject`, retrieves the agenda JSON file and converts it into PowerShell Custom objects

* `Get-Sessions` allows me to return all session which match a filter which I'll show soon

* `ConvertFrom-EpochTime` – Yeah, for some reason only known to the app developers, the timestamps in the JSON data are Unix Epoch numbers so converting them to Pacific Daylight Savings time was tricky!

So again,  create an object called AgendaSession as SHiPSLeaf. We then create the public properties for a session; The Session ID, Name etc.. Note the Hidden Data property at the bottom.  This is how can mark private properties.  So the SHiPS provider won't display them to users. And the constructor for the agenda session.  Just like the speaker object we pass in a data object too.

Notice the use of id at the top, not name which we used previously. We have to use the Session ID for Agenda sessions for two reasons:

1. Session Titles may not be unique, for example the session called Lunch will appear multiple times and object names MUST be unique within a directory.

2. Session Titles may contain illegal characters for a Leaf Name.  The Id is a number so it's safe to use

Ok so now we have an Agenda Session, time to create an AgendaTrackSummary directory object. The track summary object are the "All", "Day 1" nodes on our DAG.  Now the interesting thing is in SHiPS we don't have to have a single object per node.  The same SHiPS object can be used for multiple nodes.  And we use this trick for the AgendaTrackSummary object.

So we create an object called AgendaTrackSummary. Notice that Directories can have properties too, not just Leaf objects. It has a name, the number of sessions in the AgendaTrack and a private filter property.  This property is what we'll use to know which sessions this track summary will show.  For example, for the track called "All", the filter will be empty.  For the Track called "Day 1" the filter will be "Day = 1" and so on.  This is how we can use the same SHiPs object for multiple nodes in the DAG.

Then we have the constructor.  Like normal it has name. It also has the filter that this Track summary will use. And you can see how we use the `Get-Sessions` helper function down the bottom there.  Makes it easier to read what's going on. And finally, because this is a SHiPSDirectory we need the GetChildItem method. Where we get all of the Sessions based on the filter, and the create AgendaSessions for each item.

So now we have the AgendaTrackSummary object, time to modify the Agenda directory object to create all of the summaries. So here's the old Agenda GetChildItem method. And the new code. So the first part which is highlighted is somewhat straightforward. The agenda has 5 track summaries.  The top one called "All" has an empty filter, so all sessions
The next one down called "Day 1 – Mon" has a filter for Day = 29.  Because Day 1 is the 29th of April. The Day 2 etc.

This next part is a little more complicated. I found in the data, there are actual talk tracks.  For example this SHiPS talk is in the "PowerShell Language" track. So. The first loop there goes through every single session and finds ALL of the unique track names. The second loop then takes all of the track names and then for each item, creates an AgendaTrackSummary object with a filter of "Track equals the track name". And with that, all of the agenda information is created so let's see it in action.

Lets get the TrackSummary name and the number of sessions in each track as a table. And tada.  There are 80 sessions in total.  With my favourite Track Meal having 9 sessions!

So the AgendaTrackSummary object is working, what about the actual session information. Let's see what the first 5 sessions on Tuesday are ... Huh, not really useful.  Let's try that again

Let's get all of the properties instead the default ones. Much better. Breakfast is the first session on Tuesday! There's too much to show on one slide, but you get the idea.

Phew....that was a lot to take in.  Right now, our module is fully functional.  You can find speakers and sessions with some nice filtering. But we can make it even better.  We can modify our Leaf objects to give content.  So that when users use the Get-Content cmdlet it will actually give them something useful.

And we do that by adding the GetContent method to our Leaf classes, which returns a string. So this is the Speaker object and in it we return a markdown file of the Speakers name and their Bio. And this is the AgendaSession Leaf object. And we return some markdown text with the Sessions name, time, location.  And the long description of the session.

So let's see what this looks like. So we can use the Get-Content cmdlet on speakers.  So this is me! And what about a session on Tuesday.  Note we have to use the Session ID, not it's name. And there's the session information. Of course if you're using PowerShell 6 you can use the Show-Markdown cmdlet to make this look pretty.

We can make the module even more useful.  In the sessions it doesn't actually say who's speaking which was an oversight.  Also in the web app version you can't see the sessions for a speaker, which is really useful.  What we're doing here is creating additional links between leafs which is a little dangerous.  Remember that we don't want to create cycles in the DAG. So in our DAG for this module.  If Speakers had links to their Sessions, and Sessions had links to their Speakers we'd end up with this; A loop! So what can we do then?

Well we can give "hints" but not direct links. So the session objects can give the NAMES of the speakers and speaker objects can give the Session IDs.  That way a user can use cd or Set-Location using those hints So the Speaker object. We add some information. We add some public properties e.g. How many sessions the speaker has, the name and time of the sessions and most importantly the SessionIDs at the top there.

And then in the Populate method we use the Get-Sessions filter to find all the sessions the speaker is speaking at and populate the public properties. Now the user has more information about a speaker, with the Session IDs if they need it, and we haven't created any loops in our DAG. Next is to modify the AgendaSession object with speaker information.

Again, we add a new public property called speakers. And then we populate this by going through all of the speaker IDs for the session and then finding those IDs in the Speaker JSON file. Now the user has more information about a session, with the Speaker Names, and we haven't created any loops in our DAG. So if we have at look at my speaker information now we can see all the Session information. And if we look at a session we can see the speaker names.

So the module is working fine, but it doesn't look very nice.  The default properties that are showing aren't that useful.  So let's make it look pretty. PowerShell uses XML formatting files to tell PoweShell how to display objects.  I'm not going to into detail about this as there's plenty of other blogs and documentation on it. But you can start with the powershell help system and about_Format.ps1xml. And there's probably people here in this very conference that can help you too!

So we create the formatting XML file and then we can make the module use that in the Module manifest by specifying the `FormatsToProcess` setting. And let's see some before and after comparisons

So let's get the first 3 speakers as a table.  This is what it used to look like ... and now it looks a lot nicer and shows the most common information you need right away.  No need to use Select-Object to get the properties you want.

What about the sessions ... Yeah, not useful at all.   But now ... Much better!!!

---

And that is how to create a SHiPS module for the PowerShell Summit Agenda.  I went through a lot of things, so I'll quickly recap.

* Start with some planning – Draw a picture of what the module will show the user.  This is the DAG.  The DAG may change WHILE you're developing the module and that's ok. But I find it's really useful to have some idea of what the module will look like before I start writing PowerShell.

* Create the root object first, Then create the directories, Then create the leaves.  Don't try and do them all at once

* A process I found REALLY useful was to make small changes and test that they work. And then repeat that loop.  It's easy to be overwhelmed when first starting out and this loop really helps to stop being overwhelmed.

* And lastly don't worry about making it pretty at the beginning.  First get it working and then make it pretty

Alright what about some more advanced information about SHiPS

---

So my example module used static data files which is fine.  But what about using remote services, like an API.  Well, it's all PowerShell so whatever you can do in PowerShell you can use in SHiPS. So if you can query it with Invoke-WebRequest or Invoke-RestMethod. Here's an example of another SHiPS provider I wrote.  It calls the GitHub REST API and present github repos as a filesystem. So the github view on the left and the provider view on the right there. And this is the master branch of the puppetlabs-powershell module. Note how the look VERY similar. Which brings us to the next topic ...

How do we manage credentials for a provider.  You need credentials for the Github module I just showed you. The Azure SHiPs module does too. Currently the SHiPS provider doesn't allow you to pass credentials from the New-PSDrive cmdlet. And you'll end up with this lovely error message and be sad. So what _can_ you use then? you have a couple of options:

* Firstly add your comments and plus ones to github issue 110 in the SHiPS repo so the maintainers know this is something we want!

* You can store tokens in environment variables, like I did for my github provider

* But you could also store the authorization information in a file or in the registry.

* Pretty much anything outside the "realm" of PowerShell. In particular global or module scope variables

Which seems like an odd thing to say but this is due to an architecture decision in SHiPS

The variable scoping or more specifically the runspaces used in SHiPS will trip you up.  Took me ages to debug this. When SHiPS creates a drive, it also creates it's own runspace for that drive to operate in.  That means exported functions, or global functions will appear in both the default runspace, which is where you import your module into, but there will be an independent copy when the PS Drive get's created. I'll run you through an example ...

So we're in PowerShell.  Let's import our github SHiPS module. We import the module and it sets up a global variable called GithubToken so we can store our API token to talk to Github. Next then set our Github Token using the Set-GitHubToken cmdlet to abc123.    Excellent, let's create our new PSDrive and use our new token. We create our new PS Drive, and behind the scenes it creates a runspace and imports our github module into that. Note now we have two variables called $Global:GitHubToken and two functions called Set-GitHubToken. And importantly the drive runspace DOESN'T have our github token in it.

Anything we do with that drive e.g. cd, getchilditem, get-content happens in the drive runspace. And it will have an empty token. Even IF we call Set-GitHubToken again, it get's called in the default runspace context. The drive runspace is effectively hidden from us. This is why you need to store credentials outside of a runspace. E.g environment variables.

Earlier I mentioned that SHiPS has a caching mechanism. The SHiPS cache is turned off by default, however when turned on SHiPS will cache the directory tree as it gets used which makes it great when users are traversing up and down the directory tree. Of course if you want to, you can use your own caching mechanisms too. You can turn it on using the SHiPSProvider attribute on a class.

SHiPS has other features, some of which you may have used on other Providers ... Earlier I showed you how to use Get-Content to get the Speaker and Agenda content.  SHiPS also supports the Set-Content command too so you can write information. So for example, what if I could update my own Bio text, I could use Set-Content to do that.

We're probably all used to using the pipeline to filter, but sometimes it's preferable to do the filtering at the provider level. SHiPS supports filtering on the SHiPSDirectory object. So here I'm getting all the speakers and using the filter of "Glenn*".  The top one would be using the pipeline, and the bottom example is using SHiPS filtering.

Dynamic Parameters are additional parameters passed from GetChildItem.  For example I could define a dynamic parameter called Track which takes a value of General.  So it would return all sessions that are in the General Track. Or Perhaps all the talks on Day 2, or from the Speaker called Glenn. Or you could combine them together for example all General Tracks on Day 2. There's a lot of content here and the best place to get more information from the DynamicParameter sample in the SHiPS github repository.

Ahh Testing... a topic dear to my heart.  Yes you can test your SHiPS module using Pester, remember this is just PowerShell classes and functions and you can test it like any other PowerShell module.

---

So you've used SHiPS and you like it, but it has some significant limitations right now;

* Doesn't support passing credentials.

* It only has a tiny list of supported cmdlets.  So you can't use New-Item or Remove-Item etc.

* It also doesn't pass through the drive information into the ProviderContent.  So you have no idea where in the path a SHiPS object is, or provider full paths for hints.

BUT, you can make changes to SHiPS, because it's open source however the SHiPS project is still quite young in terms of an Open Source Project. Why does this matter? Well, it's still difficult to contribute and figure out how the project works. For example, there are no tags in the github repo, the changelog is incomplete, it uses a different git branching workflow than the rest of the other official PowerShell projects I've seen.

There are no unit tests for SHiPS or P2F, only integration tests. Makes it difficult to make changes with confidence. The test suite is still Azure Cloud Shell focused. The reference documentation is incomplete however the narrative document is excellent.

I have high hopes that Microsoft make more time to invest effort into this project; to make it easier for the community to contribute!

### Wrapping up

So wrapping up ....

* SHiPS is a User Experience module.  It wants to make the life of the user easier; so always remember: what will the USER experience

* If you feel overwhelmed when you start; make small changes and test it and loop.

* Read the documentation. it is useful and has full examples of the advanced features

* And always remember the DAG.  Graphs are everywhere.
