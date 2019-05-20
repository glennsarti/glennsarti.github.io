---
title: Presentation - Beyond Pester 102
excerpt: "Beyond Pester 102: Acceptance testing with PowerShell"
category:
  - presentation
header:
  overlay_image: /images/header-pssummit-2019.png
  overlay_filter: 0.75
  teaser: /images/teaser-pssummit2019-pester102.png
tags:
  - powershell
  - summit
  - 2019
  - pester
read_time: true
modified: 2019-05-20
---

### [PowerShell Summit North America 2019](https://powershell.org/summit/)

Previously in "Beyond Pester 101: Applying testing principles to PowerShell" I talked about unit and integration testing with Pester and PowerShell. Now that we're all experts there, it's time to tackle Acceptance testing, also known as End to End testing.

---

[Event details](https://app.socio.events/MjQ4Nw/agenda/14445/session/61496)

[Presentation](https://speakerdeck.com/glennsarti/beyond-pester-102-acceptance-testing-with-powershell)

[Recording](https://www.youtube.com/watch?v=L-1nXtaQ6YM)

## Reviews

<blockquote class="twitter-tweet" data-conversation="none" data-cards="hidden" data-partner="tweetdeck"><p lang="en" dir="ltr">Acceptance test is the bread and butter for User stories and defining your MVP... BDD helped me to define what the MVP looks like.. I tend to over-engineer stuff... Not helpful in trying to make the sprint goal... You nailed it!!</p>&mdash; Irwin Strachan (@IrwinStrachan) <a href="https://twitter.com/IrwinStrachan/status/1130048853586128896?ref_src=twsrc%5Etfw">May 19, 2019</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## Resources

Source Code

[https://github.com/glennsarti/PSSummitNA2019-Pester102](https://github.com/glennsarti/PSSummitNA2019-Pester102)

Cynefin

[https://www.youtube.com/watch?v=N7oz366X0-8](https://www.youtube.com/watch?v=N7oz366X0-8)

[https://en.wikipedia.org/wiki/Cynefin_framework#cite_note-SnowdenBoone2007-1](https://en.wikipedia.org/wiki/Cynefin_framework#cite_note-SnowdenBoone2007-1)

[http://cognitive-edge.com/videos/cynefin-framework-introduction/](http://cognitive-edge.com/videos/cynefin-framework-introduction/)

Rob Crowley – Testing in production

[https://speakerdeck.com/robcrowley/testing-in-production-for-the-bold-and-true](https://speakerdeck.com/robcrowley/testing-in-production-for-the-bold-and-true)

Mattermost

[https://api.mattermost.com/](https://api.mattermost.com/)

Images

[https://unsplash.com](https://unsplash.com)

Operation Validation Framework

[https://github.com/PowerShell/Operation-Validation-Framework](https://github.com/PowerShell/Operation-Validation-Framework)

Devopsing In a Microsoft World

[https://youtu.be/roFVOnRxXns?t=1500](https://youtu.be/roFVOnRxXns?t=1500)

[https://www.slideshare.net/snasello1/devops-days-seattle-2017](https://www.slideshare.net/snasello1/devops-days-seattle-2017)

InSpec

[https://www.inspec.io/](https://www.inspec.io/)

Gherkin

[https://docs.cucumber.io/gherkin/](https://docs.cucumber.io/gherkin/)

[https://powershellexplained.com/2017-03-17-Powershell-Gherkin-specification-validation/](https://powershellexplained.com/2017-03-17-Powershell-Gherkin-specification-validation/)

[https://powershellexplained.com/2017-04-30-Powershell-Gherkin-advanced-features/](https://powershellexplained.com/2017-04-30-Powershell-Gherkin-advanced-features/)

[https://www.youtube.com/watch?v=hCfcLDLTf6g Part 1](https://www.youtube.com/watch?v=hCfcLDLTf6g)

[https://www.youtube.com/watch?v=mWFkFU4Hi_g Part 2](https://www.youtube.com/watch?v=mWFkFU4Hi_g)

[https://github.com/mikejonestechno/devops/tree/master/powershell/demo-bdd](https://github.com/mikejonestechno/devops/tree/master/powershell/demo-bdd)

## Transcript

_This is an approximate transcript of the presentation_

So cast your mind back to last year, I was up here giving my talk on "Beyond Pester 101". I talked about the valley of despair and the slope of enlightenment and how software testing is a skill, like mountain climbing, that can be learned and made better by practice.

We looked at 4 types of tests; White box unit tests, integration tests, Black box unit tests and characterisation tests.

And we also looked at how to make the your test suite as good as it could be, However...

I kinda sorta left the elephant in the room.  Surprisingly no one asked me about it. I talked about Integration and Unit tests, but not Full Stack or acceptance tests.  Why did I leave them out? Quite frankly they're really hard to do, and to do well. Also, I didn't have enough time as it's a big subject.  So today we're going to talk about Acceptance testing!  What it is and what tools we have.

-	If you have any questions please hold them until the end as there is a lot of content to get through.
-	I have resource slides at the end of this talk and all of the code and examples will be available on speakerdeck and github

Before we dive into acceptance testing, we need answer an important question:

**Why do I even test?**

This is an important question to answer, as it will guide you when you need to make compromises and decisions as to what to test.  This is doubly true with acceptance testing due to it's complexity, it's vital to know that you're testing the right thing.

This is a definition that I like a lot

> "To reduce the risk that a user will experience an unexpected behavior"

And there are three important points here:

* You will never be able to achieve zero bugs but you can reduce the risk or probability they will occur.  It's a risk assessment

* Ultimately your PowerShell projects, scripts etc. will be used by a User.  They are the people we should be concerned about the most.  Testing is a user centric concern

* Bugs/issues/errors/faults, They're all behaviour that a user does not expect.  But it does raise some interesting questions; If we, the developers, think something is a bug, but the user expects the behaviour , is it a bug? If a bug does not cause the user to notice is it a bug?

- By running a test suite prior to releasing code, we gain confidence in our code so we can get it out for use faster.
- One of the disciplines of high quality code is having good tests.  High quality code is easier to debug, maintain and add features to.
- It's cheaper and easier to fix a bug BEFORE it gets released to a user
- It ensures what is created actually does what it's supposed to do.
- Therefore testing forms part of the documentation or contract of the project.

So we know why we're testing but why are acceptance tests so difficult.  Why are they such a huge mountain to climb?

So if we remember our testing pyramid.  This is an image I adapted from Martin Fowler. So the hare and tortoise pictures on the left show speed, and the dollar signs on the right show cost. Unit tests (down the bottom in pink) are fast and cheap to run. Integration tests (in the middle) are slower and more expensive to write and maintain. And on top are Full Stack, in our case, Acceptance tests.  They are the slowest and most expensive to run and maintain.

Notice how the size of each group gets smaller as we go up the pyramid. So ideally you should have many of unit tests, some integration and very few acceptance tests.  Why?  Think of it as a Quality Budget.  You've only got a finite amount to invest, and unit tests are cheap so you can have lots of them. Acceptance tests are expensive so you have less of them

But acceptance tests... They're slow. Real Slow.

For example, at Puppet we have an SQL Server module – All of the Unit and Integration tests take just over 3 minutes.  The full suite of acceptance tests takes over an hour on a good day and that's fast because we run many of those tests in parallel.  If run serially, it would take well over 2 hours

Acceptance tests very expensive to run. That full test suite consumes   8 x 2012 and  1 x 2016 and 1 x Redhat 7 Virtual machine. Not to mention disk space, memory. Also running tests is a very high IO activity.  The CPU and Disk is driven as hard as it can.

And acceptance tests are complex. That test suite is testing installing and uninstalling SQL Server in multiple configurations across different SQL versions. And running arbitrary TSQL statements.  It takes a lot of effort to even get the Virtual Machines into a state to even _START_ testing.  AND we don't test clustering.  Imagine how complex testing would be then.

And then there's the shift-left movement of testing.  Where you want move testing as close as you can to the person writing the PowerShell.  Because it's much cheaper to solve a problem or a bug when you're writing the powershell code, than it is to for someone else to find the bug while they're using your script. Acceptance testing tends to be on the right-hand side of the spectrum because it's slow, expensive and complex.

And when an acceptance test fails it doesn't give you much of an idea of where to start looking to fix it. Because the further the test is from the code that you're actually testing, the harder and more painful it is to find the bug that caused it. There are more functions, more abstractions, more things to consider; there is more cognitive load. Why?


Because there's more that can go wrong. In Unit tests, there's usually one, maybe two things that could wrong.  That's why they're called UNIT! In Integration tests, a few things could go wrong. But for acceptance tests there's a PLETHORA of things, A whole bunch, a shit tonne!

And here's the kicker ... The things that go wrong may not be in your control.  Third party dependencies can be a real problem; for example, Microsoft releasing new versions of PowerShell Core (Yes powershell is a dependency for your PowerShell scripts!) or module authors releasing code, or DSC Resource Packs being released.

Here's an example of what that looks like from my job at Puppet...

This is testing history for our Puppet PowerShell module.  The `3d2197ca` is the commit SHA of what we're testing.  Because it's the same in both tests, we know we're testing EXACTLY the same code. Note that first test (at the bottom there) at 11am the acceptance tests passed and then the next test (the one above) is a mere 28 minutes later and they failed. We changed nothing in our PowerShell module but for some MAGICAL reason the Acceptance tests failed!!  This can be infuriating and hard to diagnose.

And to make everything even worse ...

> "Customers don’t measure you on how hard you tried, they measure you on what you deliver." - Steve Jobs

The people using your PowerShell code don't care how hard it is to do acceptance testing. They don't care how hard you tried to write high quality powershell; They care that it worked for what they were doing.

So if they're that horrible and people don't care, why use them? To be honest it comes down to this:

- Acceptance tests are the closest thing to what an ACTUAL user will experience.  They are the most accurate.  People probably won't use your powershell code like they way you unit test. Users can't be mocked reliably, they work in mysterious and surprising ways.

- Unit and integration tests tend to be testing for things that you, the author, know.  A classic example of this is the regression test.  "My powershell script previously failed at X so I will write a test to make sure it doesn't fail at X again".  This is testing for known knowns or even known unknowns.

- But acceptance testing is testing for unknown unknowns.  The unexpected things, the things you wouldn't consider "normal", and unit and integration testing can't really help here.

To summarise then, acceptance tests are hard because they're slow, costly to run, can fail in many ways, can fail because of things not directly in our control and user's don't really care. So what can we do then to help us surmount these difficulties then?

There are two things;

1. Think about testing and complexity in a different way.

2. Change the scope of the acceptance testing to make things easier or less impactful

### Think differently

How can we think differently about testing to make our lives easier? Acceptance tests are complicated things, so how can we think differently about complicated or complex things. Fortunately we can draw on other fields of study to help...

The Cynefin framework was created way back in 1999 by Dave Snowden and he describes this a sense-making device.  As in how can we make sense of our own and other system's behavior. The framework was released in a paper in 2003 and since then it's been applied in many aspects of Software Design and today I'll give you a very quick tour of the framework and how you could apply it to Software Testing.

Disclaimer – I'm not going into the great depth here.  I **highly** recommend you go watch Dave Snowden's videos to learn more. This is in my link slides at the end

The framework starts with four domains:

- Obvious (also called simple)
- Complicated
- Complex
- Chaotic

Some of you may be thinking, doesn't complex and complicated mean the same thing? And I certainly did when I first saw this.  But the framework defines complicated and obvious as and ordered system.  This means that cause and effect is known or discovered.  So for example.  I make a change to my powershell script, the unit tests now fail, therefore it was the new code that cause the problem.

Complex and chaotic are unordered. "Cause and effect can be deduced from hindsight or not at all"  So in my Puppet PowerShell module example, The powershell script hadn't changed and yet the tests still failed.  There's no cause for the effect and I never did figure out why.  Or my favourite, you run the tests again and "magically" they work. So the big differences between complex and complicated is whether there's ordered cause and effect. So how does testing fit in then?....

Unit tests tend to live in the Obvious domain. The relationship between cause and effect is clear: if you do X, expect Y.  And when we get a unit test failure we use "sense–categorise–respond" to fix it. We see the Pester test output, we can the categorise what kind of failure it is. For example, a type conversion failure, or a off-by-one error, or passing in null, and then we respond by fixing it. In essence by looking at the unit test failure we know how to fix it!

Integration tests tend to live in the Complicated domain. The relationship between cause and effect requires analysis and there are a range of right answers. We see the Pester test output.  But we also need information from which tests passed and if there were any other failures.  We then analyse the data and make a decision on where to start fixing the test issue.  This analysis requires expertise and a good knowledge of the PowerShell script and how it works.  This is needed because we're testing how things integrate with each other.

Acceptance tests tend to live in the Complex domain. The relationship between cause and effect be deduced in retrospect. So typically when you see an acceptance test failure this is what you do. "I wonder what happens if I try changing this one thing?". You run the tests and they still fail. "I wonder what happens if I try changing this other thing?" You run the tests and repeat this process.  Eventually you try enough things and you figure out what went wrong.  And then you fix the script or test so the tests pass. This is Probe – Sense – Respond approach.  You prod and poke the scripts and tests, until you understand the cause and effect.

Lastly, Chaos Engineering or Testing-in-Production tests tend to live in the Chaotic domain. Now hopefully you never run tests that fall into Chaos.  This means you ran your tests and now your production systems are ON FIRE! Pester tests mean nothing right now, you need to get production back online first and THEN worry about what you did.


So this all lovely but what does this have to do with thinking differently.

Firstly, while we can use the Pester testing frame work for unit, integration AND acceptance tests, the tests themselves are quite different. For example, don't try apply the same debugging steps you use for unit tests for acceptance tests.  It very likely won't work and will cause frustration.  Conversely don't apply complex debugging to an obvious problem.  It may work but take A LOT longer.

Secondly these are just general buckets.  You can get complicated unit test failures or obvious integration test failures. As you become more and more familiar with the PowerShell code and tests for it you may find that Complex things become Complicated and Complicated things become Obvious.

Thirdly, because acceptance test failures tend to be complex, having lots of debugging information is really, really useful for when they go wrong.  Much more so than integration and unit tests.  So make sure your log as much information as possible.  For the most part logging is basically free so use it!

This is just the tip of iceberg for this framework.  I really suggest you go read more about it.


### Changing Scope

Secondly, we can change the scope of the acceptance testing to reduce the scope of failure. So with Acceptance tests, if we can reduce the scope of testing, THEN we reduce the size of what we need to test, which REDUCES the test complexity, test duration and the list of things that could wrong when a test fails. For example, let's say you have a PowerShell module that works on Windows Server. Do you really need to test it on Server 2012, 2016 and 2019, both Core and Desktop editions? Or perhaps, that same PowerShell module can be used in 20 ways, but most of those are just small variations, do you need to test all 20, or could you get away with just 5 core tests?

And this is the downside of reducing the scope of your testing. It reduces the testing coverage.  Now this in of itself isn't a bad thing.  Remember in my definition of "Why do I test" that testing is ALWAYS a risk assessment.  This is also why acceptance testing is difficult because it requires more holistic thinking about how you test. When I say change or reduce scope, what this really means it's a type of acceptance testing....Here's an example of the different type tests you could run

This is a list of possible tests you could run with a distributed application, using a microservice or mesh architecture. Now of course, many of you are probably thinking, "but Glenn my PowerShell script is not a distributed microservice application using a mesh and APIs" and you're right of course.  Many of these tests are irrelevant for PowerShell however note a few things.

Firstly there are a lot of different types of tests on the left there.  e.g. Usability, Regression, Penetration tests.  This is just narrowing the scope of the testing to one particular concern.  For example Regression testing is ONLY concerned about making sure we haven't reintroduced bugs.  Penetration testing is ONLY concerned about ensuring the security of the application.

Secondly, notice that different tests are run in different parts of the application deployment process;
We can run tests in Pre-production (which is before we deploy the app), then we can run tests once the app is deployed in production. We can apply some of these principles to our PowerShell testing suites.

### Phew!

So that's a lot of concepts and theory to take in! Remember I'll be posting this slide deck and all my references so you'll be able to go through this again and take it all in!

But let's see this in action and ACTUALLY write some tests

---

We're going to write a series of tests for a chat network integration.  Hopefully many of you are familiar with PoshBot, which is a Bot framework for chat networks.  By default it has a Slack backend, which means PoshBot can integrate with the Slack Realtime Chat API and act like a person in a channel. I've written a PoshBot backend for Mattermost which is a self hosted chat system with an API similar to that of Slack.

The architecture of the chat network looks like this. In the middle there is the Mattermost Server. It's hosted in a Docker container and exposes a Rest API.  On the right there is the Mattermost Client.  Fortunately the Server also hosts a Web Client so you can interact with the server with a GUI or, you can use the Server Rest API. And finally on the left there is PoshBot. It's all written in PowerShell and can be run with either PowerShell Core or Windows PowerShell.  PoshBot uses the Mattermost Backend to communicate with the Server.  It's basically acting an adapter.

The first set of tests we will write, will validate that the Mattermost server is operating correctly. These are called Operational Acceptance tests. Secondly we'll write some User Acceptance Tests. Where we'll simulate being a user and interact with the Bot and expect certain responses.

So lets start with the Operational Acceptance Tests.

Just a disclaimer, there are many definitions for these terms, so don't take what I'm saying as a HARD and fast rule.  Even experts the disagree!

"Operational Acceptance Testing (or OAT) is used to conduct operational readiness (pre-release) of a product, service, or system as part of a quality management system", which is just a fancy way of saying "Do I accept that this system is able to operated and supported as I intended?" Now; OAT is done BEFORE you release changes.  So for example, I would make changes to a DSC configuration script, I would then test these changes in OAT, and if that passed I would then commit the changes to the DSC configuration and whatever production server would see the changes and apply them. That's testing BEFORE release.

But, many of the tests in OAT are useful AFTER you release the changes.  This is Operation Validation Testing and is testing to ensure that the system is still in a state that is able to operated and supported as intended. So OAT is before release, OVT is after release.

And the nice thing about OVT is that it's a VERY good place to start writing acceptance tests. There's no need to setup a virtual machine, or load test data. Remember the three As of testing: Arrange (pause) , Act (pause) and Assert (pause).  In OVT, there's no test arranging just acting and asserting. So for this example we're going to ask three questions:

> Is the server actually running?
> Does the server think everything's ok?
> And are the critical configuration settings correct.?

And in total we'll write seven tests in plain Pester syntax.  I'm not going to show all of the code here, as I assume you can all write Pester PowerShell tests, but I'll quickly show you the tests for "Is the server running?".  Remember all this code is in my Github repository.

Oh a quick side note here – One of tests looks for a HTTP 404 response. This is where PowerShell can be a bit annoying. HTTP errors like a 404, throw a PowerShell error instead of returning the HTTP response.  I had to write a little helper function which can trap the HTTP Errors and still return the HTTP response object.  Otherwise I'd be forced to do string comparisons which aren't great.  Oh also the error type changes between Windows PowerShell and PowerShell Core so that was annoying!

Alright, back to the tests. We have the two basic tests to confirm if the server is actually running.

Firstly when we Invoke-WebRequest the server we should get a HTTP status code of 200. And Secondly, we check the HTML payload to ensure it has the title of Mattermost.  Why? Just because a HTTP server responds with 200, doesn't actually mean it's the mattermost server we're testing. Again note there's no test arranging here.  Just Acting and Asserting:  Query the webserver, check the response.

So on an operational server you'd see the following test output

> 7 Tests passed!

And let's say the server isn't happy ...

> 1 Failing test!

Because the server responded with Degraded instead of OK

Great, we can now test  whether our Mattermost server is operational. But you could also write tests for say, Active Directory or Sharepoint or whatever external system you have. These tests can become troubleshooting guides or part of your disaster recovery processes and playbooks.  They can replace the manual checkbox ticking you may need to do in your job. I also saw someone using Pester to validate PowerShell script syntax.  You can use these as pre-change-merge tool. But right now it's just a simple file.  What if we could package them up and distribute them easily.  And this is where The Operation Validation Framework comes in

The PowerShell Operation Validation Framework or OVF provides a way to organize and execute Pester tests which are written to validate operation (rather than limited feature tests). So what we can do is take our plain pester files and package them up as a powershell module.

So first of all we need a module directory.  For this example I used "Example Module". Next OVF needs a directory called Diagnostics and inside there we have Simple tests, which are supposed to be basic and not take too long.  And Comprehensive tests, which as the name suggests are more complicated and test in a more comprehensive manner.  And then finally  the Pester test files sit in either the Simple or Comprehensive directories. I put the "Is the server running?" tests into Simple and the others into Comprehensive.  The content of the test files hasn't really changed.  For example this is the simple tests file;

I added param block so that OVF can modify the Mattermost URL if someone wanted to use the tests on a different server. And I changed the text for the Describe and Context statements.  I'll talk a bit later why I had to do that. Apart from that the tests are identical

So OVF has two functions, Get-OperationValidation and Invoke-OperationValidation.

If we run Get-OperationValidation it will list all of the available tests. Note that I pass in the path to our module, otherwise it will search all of the modules in your module path.  I've truncated the output here to only show the Simple tests but it will also show the comprehensive tests. And then we can run Invoke-OperationValidation to actually run our seven tests. Notice the names look like the pester ones we looked at earlier and they all passed.

Just a few gotcha's I tripped over when I was converting my Pester tests to OVF

A lot of the examples you will see around OVF talk about a PowerShell module which implies you need a module manifest file.  That's not a requirement for OVF.  But it is if you intend to publish your module on the gallery

Secondly you can only have a single root describe block and the title has to be text only.  No interpolated variables, just pure text.

And lastly make sure you're on the latest version of the OperationValidation module before you start trying things.  It's amazing how many errors you can fix by upgrading ... ask me how I know!


So we've packaged up our tests into a module so what? Well you can now share your tests with others, if that's appropriate.  Even post them on the PowerShell Gallery. You can take your module and run it with a tool like Watchmen.  This then manages running the OVF tests and alerting on failures.  This can become the basis for a monitoring system.

Or you can go full circle.  You can use PoshBot to run OVF tests on demand.  So imagine you're doing a storage migration BUT you also want to play golf.  You could invoke the migration and TEST the migration, via Chat, on your phone, while hitting golf balls (and possibly drinking beer).  And that is a picture of a Columbia Sportswear Engineer doing basically that; back in 2017! But we can go one step further.

Those tests I wrote are very similar to Compliance Acceptance Testing. Which, you can probably guess, are testing that your system complies with a set of regulatory controls. What if I could write tests that not only confirm that my system can be operated correctly; but also complies with some regulatory controls? And that's what Inspec is written for.

Inspec is a testing framework written by Chef, where you can use a human readable language for specifying compliance, security and policy requirements. "But Glenn", I hear ask, "Isn't Inspec written as an extension to rspec, ruby's unit testing framework?  This is a Pester talk, not rspec". Yes, you're right but this a BEYOND pester talk. Also Pester was originally inspired by rspec so they have many similarities.

``` ruby
describe package('telnetd') do
  it { should_not be_installed }
end
```

And if you have a look at the example here, if you can read Pester, you can understand this.  It has a describe and it block.  With an assertion called "should"

But the power of Inspec isn't in the testing syntax. It's the metadata for each test that's VERY interesting. You see OVF kind of had the right idea.  Not all acceptance tests are equal.  Some tests are simple, and some are comprehensive.  But when you start talking compliance and regulations, like inspec does, the metadata needs to be far richer.

``` ruby
control 'sshd-8' do
  impact 0.6
  title 'Server: Configure the service port'
  desc 'Always specify which port the SSH server should listen.'
  desc 'rationale', 'This ensures that there are no unexpected settings' # Requires InSpec >=2.3.4
  tag 'ssh','sshd','openssh-server'
  tag cce: 'CCE-27072-8'
  ref 'NSA-RH6-STIG - Section 3.5.2.1', url: 'https://www.nsa.gov/ia/_files/os/redhat/rhel5-guide-i731.pdf'

  describe sshd_config do
    its('Port') { should cmp 22 }
  end
end
```

This an extract of the Inspec documentation.

Inspec introduced a new concept called a control.  You can see that at the top.  This is the audit control that the test will be proving. We then have a whole heap of metadata about the control: Impact, title, description and various tags and references. And then finally down the bottom we have the actual test. That is a LOT more information about tests than either Simple or Comprehensive. So lets translate the basic Mattermost tests into Inspec format.

This is the before ... Query the server, and expect status code 200 and the content contain a title of Mattermost. And After ... The same test in inspec format.

We now have a control at the top, which says these are critical tests. And the two tests down the bottom.  Is the server listening on the port and validate the server response. I converted all the tests and you can see that in my github repo, BUT I'm sure you can agree, this is quite readable and looks a little like the Pester tests you saw earlier.

But it's not all rainbows ....

- Inspec requires an installation package.  You can get inspec either via the ChefDK or you need to install Ruby on Windows and use a gem installation process. That can be painful. (Also check the licensing for Inspec as it's recently changed)

- Secondly Inspec uses rspec, but it is NOT rspec. If you come from an rspec testing background, or pester for that matter, learning Inspec can be tricky because many rspec concepts don't exist.

- Thirdly, even though Inspec and ruby are cross platform there will be spots where Windows is not supported, or is only half supported.  For example that http request test I did.  I hit an outstanding bug and calling a webserver is NOT supported on Windows in Inspec which seems weird. This is not unique to Inspec but something to be aware of.

If you want to learn more about Inspec go to inspec.io So what can we do.  What if we extended Pester to have Inspec style syntax? So that's what I did.

Introducing Inspecster or (Inspec Pester)! I took the control method and the properties for it and added it the pester function list.  I also enabled the metadata to be output so reports could be generated from it. Again, this is all on my github repo for this talk

Here's that same connection test in Inspecster format ... Notice the control function at the top and the impact title description and tags, just like Inspec and then pester tests below in the describe block.  The same ones we wrote at the beginning! So now we have very rich metadata about our compliance tests. But that's not very useful, we need to see the output in a nice to read report.  So I took the PSTestReport module from Michael Willis and modified to display all this new lovely data.

So here's a report for a degraded Mattermost server. We can quickly see at a glance the number of failures are and how impactful they are. And then we can see the individual failed controls.  We can see the tag information to make searching easier, AND we have clickable URLs we can use to get background information about the failure.  For example I'm showing links to the API documentation so you don't even need to search for it.

But these links and information could also be for CIS standards document, or a link to your internal Sharepoint site with your auditing information.  You can put whatever information YOU need for a compliance failure here.  Imagine the possibilities this opens up for your auditing and compliance reporting.

### Phew!

Phew, ok so that's a lot of information

* OAT and OVT testing are similar but occur at different times in the software lifecycle
* Thinking more broadly about validation testing means going beyond just writing tests for right now. By sharing and packaging them up, this means we can use them collaboratively in things like dashboards and ChatOps.
* We can move from "Does the system work?" to "Does the system comply?" by using Compliance Testing tools and automate auditing and compliance to make our lives easier
* We can look outside of the Pester/PowerShell world for examples on how to do acceptance testing differently.  We can take the lessons learnt there and apply it to Pester.

So let's move onto UAT

---

> "User acceptance testing, or UAT, consists of a process of verifying that a solution works for the user"

Not that the system works correctly, but that the "thing" behaves like a User would expect. In our case of a PoshBot Mattermost backend, the user will type messages and our bot should respond appropriately. Remember the three As of testing: Arrange, Act, Assert.

- Act is sending a message
- And Assert is verifying the response
- But what about the first A, Arrange. Well this is where things get difficult ...

In order to test respond to client messages, we'll need to arrange some test instances. We'll need a mattermost server (pause) somewhere to run our PoshBot backend (pause) and of course somewhere to run our tests impersonating a user. Pester is a very versatile and flexible testing framework, but it is not a provisioning framework.  To do that we'll need to move back to plain PowerShell and somehow structure the execution.

And these will be the three steps:

1. Create the systems under test.  So that's the mattermost server and poshbot stuff

2. Then run the tests

3. And finally destroy the systems under test

Seems simple enough *smirk* How hard could it be?

I spent days trying to get this working, and I eventually did.  To contrast that, the actual tests themselves took an hour or so to get right.
Doing this provisioning work is HARD!: HARD to get right and HARD to get consistent.  And I can't stress this enough! without it being consistently right, the tests are useless because you'll be chasing ghosts. All of the source code for this is on my github repo and all up there's just over 200 lines of PowerShell just to start up the Systems under test

And I took a shortcut here, I ran PoshBot on the local computer not in the docker image.  Strictly speaking you shouldn't run Systems Under Test on your local computer as it's prone to the "It worked on my Computer" problem.  That is UAT passes on your computer because of some special configuration you forgot about.

An interesting side note here ... How did I know that my Systems Under Test were valid?  Because I could run the Operational Validation Tests that we wrote previously, against them.

So all of those steps I wrapped up into a single powershell script called Pre-Pester.ps1 and we can create our systems under test and we can be sure that it's ready to be tested. Now for the tests...

One of great things about user tests, is that; I am a user, _you_ are a user. So it shouldn't be too hard for us to come up with tests as a user.

So here's a screenshot of me USING mattermost.

You can see I've typed the message `! help`. Then after a short time, poshbot responds. It adds a white checkmark to my message and responds with a new message with all of the available commands. Starting with the text `FullCommandName`

And we can turn this workflow into a test. This is the equivalent Pester code. First we got some information about our test user, what channels it has, authentication tokens etc.. We then craft a message to the MatterMost API submitting the message `! help`. We than wait a few seconds and query MatterMost to give us information about the message I just sent. Now we can start the assertions. That little checkmark is a message reaction.  So can check firstly that the updated message actually has at least one reaction, and that the message reaction is the "white check mark".

Next we need to make sure that poshbot responded with the help text. So we query the Mattermost API and ask for all the messages AFTER the one I just sent. And then we assert that there should one and only one additional message. And that that message should start with the Markdown triple backtick, a newline and then the words FullCommandName. We converted a manual user experience into a series of actions and assertions using PowerShell.

Once you start to see the flow of Act-Assert, you can start to see other tests beginning to emerge. For example, What if I try to get help from a command that doesn't exist? Well it should react with a check mark and send me a message that says the command could not be found. Or What if I try to get help and give it too many parameters, Well, it should react with a red exclaimation mark and send me message that says the command errored. You can use the similar pester code I just went through and slightly adapt it for each of the tests.

And then you can go further, what if I sent Poshbot a private message instead of in a public channel? What if I had another user that was continually talking gibberish at the same time, would poshbot react the same?

So we created our systems under test and then ran our User tests. Lastly is cleanup.  Stop and remove the docker container and Stop the Poshbot Process.

And then we can wrap the whole process up into a powershell wrapper script ...

Step 1 - Inside a try block, we run the Pre-Pester script which provisions the docker container and poshbot process,

Step 2 - we then run the Pester tests

Step 3 - inside the finally block, we run the Post-Pester script which removes the docker container and stops the poshbot process

Step 4 - We exit powershell with the number of tests that failed.  This is useful when the tests are automated.

So we now have an automated User Acceptance Test Suite which we can run whenever we're making changes. Now, wouldn't it be nice if there was a framework or tool we could we use to make our lives easier here. I mean surely provisioning things like docker containers are solved problems.  And sadly this is not the case in PowerShell land.  The closest we can get is maybe Chef's Test-Kitchen product.  I had major issues getting it to work in my case and you can find me later and talk about the challenges .  So if you want a new side project, how about writing a generic Acceptance Testing Framework that's PowerShell first

### Thinking different-er

I've just gone through writing some OAT, and UAT tests, but what are some other ways of thinking differently about acceptance testing to make our lives better. Why confine ourselves to the Pester syntax?  The Pester module has support for using the Gherkin testing language.  Let's quickly show you what this looks like. This is that UAT test we just wrote to test the help message ...  I've deliberately squeezed it onto this slide. And here's the equivalent test in Gherkin.

This is so much easier to read and understand exactly what you're testing.  This is the feature and some background information.  And then a plain English description of what we are going to test. This is executable text.  It's not just a text document, Pester EXECUTES this plain english text.  If you want to change the delay from 2 seconds to 5 just change it to "and waiting 5 seconds", you don't go hunting through the PowerShell code for Start-Sleep or Pause or whatever.

You focus on the behavior of the test, not how we test it.  Remember those other examples of UAT Tests we thought of, we could write them right NOW. So up top is the screen grab of the me testing manually, and below is the equivalent in Gherkin. I can write WHAT I want to test first and then focus on writing the Powershell to implement the test later.  Separating the behavior we want to test, from the code on how we test it.

Now don't get me wrong, this does require some effort to set up, and Gherkin tends to be far more useful in the Acceptance and possibly integration testing phases. Remembering back to the Cyenfin framework, acceptance tests are different then unit and integration, and may require thinking differently about our testing language. And thinking differently about how you describe your tests can help you focus on what's important; The user experience

Just like for unit and integration tests, not all acceptance tests are equal.  And we saw this in the OVF when it categorized tests as simple and comprehensive.  I cover Test Tiering in my Pester 101 talk. Short version is categorise your tests according to how important they are and run the most important ones first.  Acceptance tests take a long time and shortening the feedback loop is fantastic.

I said earlier that when acceptance tests fail they're hard to diagnose.  So to combat this, make sure you have a broad range of instrumentation of how your scripts, modules or whatever are functioning when under test. Here's an example.  In my testing of the PoshBot module, I should take a COPY the poshbot log BEFORE I destroy the docker container.  This log contains a LOT of information which is useful to diagnose a failed test.  I could probably do the same thing for the mattermost server.  Save these logs along with the testresults.  Also this logs are useful even when nothing fails, as it represents what success looks like. Remember to always have timestamps in your logs and your tests so you can correlate events.  This is no different than your normal processes when something goes wrong.  When AD fails or SCCM has a hiccup, the first thing you do is consult the logs.

Oh also, Structured logs are better than freetext – e.g. Poshbot uses CSV structured logs.  It's much easier to parse and search later on

And finally ... Your local acceptance tests should be automated and run in the same manner that you would in an remote system like Azure DevOps pipelines or Appveyor.  This means firstly if you want to run the acceptance tests you don't need to depend on a shared, limited, remote system to do it for you.  And secondly if you can run them locally then you can easily try out new things, add new features and have a high confidence that they'll pass when you push the changes to your automated pipeline ... You do have an automated pipeline for testing don't you? *question* If not, you should get one!

### Wrapping up

We've come to the end of my talk but it's not truly the end.

Acceptance testing is a complex beast and requires you to think differently about processes and failure, compared to integration and unit testing. We need to instrument our tests differently, realise that test failure may be out of our direct control, BUT these tests can give us significant value; and confidence that our users will not see unexpected behaviour.

Confidence that you can make changes easily, Confidence that you can deploy safely, Confidence you won't break stuff.

I showed examples of OAT, OVT and UAT tests in a real world example, using some modern tooling like Docker.  And how we can take tools outside of the PowerShell world and apply some of their principles and techniques.  Acceptance testing is more than just tests, but can form the basis for compliance audits and on-demand automated troubleshooting, even when out of the office. But this is just the beginning...

Learning testing is a "lifetime" software skill.  You can take what you've seen today and apply it to C#, Python, Ruby and Typescript.  It's not just a PowerShell thing. I like this picture because even though she's at the mountain and can't go any higher, the flag says "Keep Exploring".  And I'd like you to keep this in mind.  Keep looking, to find better ways to test, to reduce your test suite and still give you the same confidence that your code does what it's supposed to do.

Keep exploring this large world of software testing!
