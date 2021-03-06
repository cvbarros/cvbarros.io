---
title: "Building Builds - TeamCity Pipelines as Code using Terraform"
coverImage: /images/teamcity_pipelines.jpg
shortUrl: http://bit.ly/2QkjvCw
# thumbnailImage: /images/teamcity_pipelines.jpg
thumbnailImagePosition: right
comments: true
autoThumbnailImage: no
coverSize: partial
metaAlignment: left
date: 2018-11-06
keywords:
- teamcity
- terraform
- terraform provider
categories:
- Automation
tags:
- ci/cd
- pipeline-as-code
- teamcity
- terraform
- terraform-provider
summary: CI/CD pipelines are a core aspect of modern software development delivery automation. We'll examine how to use Terraform to reap the benefits of _pipeline as code_.
---

<!-- toc -->
# Introduction

## Terraform Provider
This post talks about the reasoning behind the project [terraform-provider-teamcity](https://github.com/cvbarros/terraform-provider-teamcity). We'll be looking on how to apply pipelines as code for Jetbrains' TeamCity CI server using Terraform.

## Why Pipelines as Code? 

Nowadays, _Continuous Integration_ is a common practice for most software development workflows. The idea of having code being managed, tested and deployed from a central location is mainstream enough that most can take it for granted. This practice's maturity allowed the evolution of _Continuous Delivery_ and _Continuous Deployment_, which states that a given repository is always kept in a state stable enough to be shipped to production, at any time.  

To successfully enable this practice, however, most teams rely upon heavy automation and plumbing, creating _pipelines_ to deliver code from development to production using tools such as CI systems (Jenkins, TeamCity, Gitlab, Travis, etc...) to perform required steps in an automated fashion.  

Assembling a pipeline manually for a single project is not a hard task in itself. Issues start to arise whenever you are member of a Platform/Build team that creates pipelines for other teams' developers, need to expose CI/CD as a service to other users. Microservices are also a big driver for adoption - as organizations need agility to quickly manage lifecycle of deployable services, sometimes from hundreds to thousands. At a certain scale, it starts to feel clumsy to manually configure and maintain these effectively.  

Jenkins users have already been reaping several benefits of this technique for quite some time. It blipped on [ThoughtWorks's Radar](https://www.thoughtworks.com/radar/techniques/pipelines-as-code) as "Adopt".
{{< blockquote "Badri Janakiraman and David Rice" "//www.gocd.org/2017/05/02/what-does-pipelines-as-code-really-mean/" "What does pipelines as code really mean?" >}}
"Build as code and pipeline as code would have much the same definition. They are an approach to defining your builds and pipelines in diffable, readable files checked into source control that can then be treated just like any software system."
{{< /blockquote >}}

For managed services ([TravisCI](https://travis-ci.org/), [AppVeyor](https://www.appveyor.com/), [CodeShip](https://codeship.com/), to name a few), it is common to have some file to describe a pipeline to expose its extension points while encapsulating intricacies, creating a powerful abstraction that makes CI/CD very easy to setup and get started.  

# TeamCity
JetBrains' TeamCity is a very powerful, user-friendly build server, that just works&trade;. I've worked with other build systems in the past, but grew quite fond of TeamCity the longer I used it, for its simplicity and reliability. It has a proprietary version of Pipelines as Code using _Kotlin DSL_, where you can either export an existing project's settings to Kotlin format or create everything from scratch. [They have a blog series](https://blog.jetbrains.com/teamcity/2016/11/kotlin-configuration-scripts-an-introduction/) that goes into detail on how to use this feature.

**Why not Kotlin?**  

I've long had a desire to configure TeamCity builds via code, but I've hit several limitations. First, Kotlin DSL was not yet available, the only option was to use _XML-style_ settings, which would be used to version configurations, but that is not exactly _code_, right?  

Second, I've found out that information on the (back then) recently-released _Kotlin DSL_ was [basically a page in TeamCity documentation](https://confluence.jetbrains.com/display/TCD18/Kotlin+DSL) and the blog series mentioned previously. Found it hard to find deeper information, struggled with inability to reuse code for my configurations and lacked a proper development environment. These reasons pushed me away from using _Kotlin DSL_ for what I was trying to achieve.

# Terraform
[HashiCorp's Terraform](https://www.terraform.io), an _infrastructure-as-code_ tool usually well-known in the community for cloud providers and other systems, provides a powerful, declarative way of defining configurations for several types of upstream systems. 
Tapping my previous dabblings with it, I remember using Terraform to not just provision and maintain infrastructure, but rather configure platforms such as [Datadog](https://www.datadoghq.com), [GitHub](https://www.github.com) and [RunScope](https://www.runscope.com/). Managing these platforms required _custom providers_, plugins developed either by HashiCorp or the community that extended it to many other uses beyond provisioning infrastructure. 
{{< pullquote center >}}
By extending Terraform with custom providers, it is possible to maintain any API-enabled system by leveraging its core as a powerful resource-manipulation framework. The benefits of "infrastructure as code" are enabled to any configurable API.
{{< /pullquote >}}

Terraform configurations are expressed in a simple language, <acronym title="Hashicorp Configuration Language">HCL</acronym>, and are imperative in nature, dictating the _desired state_ rather than adopting a procedural style. This allows the full state being captured in code. Other Terraform features were very atractive, such as:  
<br />

- Codify configuration for multiple systems together
- Easy to validate, effectively turning configurations into a _mini-DSL_
- Reusability, with versioning, via [Terraform Modules](https://www.terraform.io/docs/modules/usage.html)
- Zero-friction CI, easy to setup _pipelines to build other pipelines_

## Writing a custom Terraform provider for TeamCity
After deciding that Terraform was the way forward, the challenge was to write a Terraform Provider in Golang, an ecosystem I had no experience with. Custom provider development can be trivial if you have the experience and a Golang client for the API you're trying to automate. Unfortunately, I had neither :cry:.  

At first, I tried auto-generating a Golang client for TeamCity's API by using [go-swagger](https://goswagger.io/). That resulted in a very unfriendly API, leaking a lot of peculiarities of TeamCity's API to the provider implementation, resulting in convoluted code that was hard to maintain.  

Since there was no other usable open-source Golang client for TeamCity, I had to write one, considering the use cases I had in mind for the provider. You can [find the code for this project here](https://github.com/cvbarros/go-teamcity). Experiences on that will serve as input for future writings.  

Thus, I've created a custom provider that uses this client, which is published at https://github.com/cvbarros/terraform-provider-teamcity.

## Modelling TeamCity Resources
TeamCity base resources are _projects_, _vcs roots_ and _build configurations_, so they were the starting points for the provider implementation. A Terraform configuration for a very simple pipeline, that performs only one build step and had one configuration would then look something like this:

{{< codeblock "sample.tf" "terraform" >}}
resource "teamcity_project" "project" {
  name = "Simple Project"
}

resource "teamcity_vcs_root_git" "project_vcs_root" {
  name = "${var.github_repository}"
  project_id = "${teamcity_project.project.id}"

  fetch_url = "https://github.com/cvbarros/${var.github_repository}"

  default_branch = "master"
  branches = ["+:refs/(pull/*)/head"]

  username_style = "author_email"

  auth {
    type = "userpass"
    username = "${var.github_auth_username}"
    password = "${var.github_auth_password}"
  }
}

resource "teamcity_build_config" "pull_request" {
  project_id = "${teamcity_project.project.id}"
  name = "Inspection Build"
  
  step {
    type = "powershell"
    name = "Invoke build.ps1 with 'pullrequest' target"
    file = "build.ps1"
    args = "-Target pullrequest -Verbosity %verbosity%"
  }

  vcs_root {
    id  = "${teamcity_vcs_root_git.project_vcs_root.id}"
  }
}
{{< / codeblock >}}

This sample code manages a _Project_, named "Simple Project", a Git _VCS Root_ with some basic settings and a _Build Configuration_ that has a simple powershell step invoking a build script place into the repository root folder.  

However, let's examine some interesting aspects covered in this basic example.  

**_Resource Dependencies are handled automatically by Terraform_**  

Notice the `teamcity_build_config.pull_request` resource references the Project and VCS Root by using `${teamcity_vcs_root_git.project_vcs_root.id}` and `${teamcity_project.project.id}` interpolated variables? This dependency graph handling is done automatically by Terraform, ensuring resources are managed in the right order. In our case, it's not possible to create VCS Roots or Build Configurations without having a Project (for VCS Roots you can create them in the _\_Root_ project but that's an exception :sunglasses:)

**_Variables_**  

Using `${var.github_auth_username}` and `${var.github_repository}` allows these values to be specified via the environment or directly when planning/applying configuration.

**_Configuration is fully captured in code_**  

You can still rely on TeamCity's interface to confirm everything is correct. But the code is simple and concise enough to look and understand how your build is behaving.  
Terraform's declarative nature makes it simple to realize what has been configured.

## A More Advanced example
However, we're talking about pipelines as code. In the real world scenario, we'd have multiple builds chained together, deployment steps and integrations. Things can get messy. The planned MVP for the provider had to cover one of our standardized pipelines for .NET applications, represented by the diagram: 

### Pipeline Anatomy
{{< image classes="fancybox fig-100 clear" src="/images/standardized_pipeline.png" >}}

_Inspection_ runs basic and fast tests, such as unit tests and linting, and gives a green status if a pull request is eligible for merging. Once merged, changes trigger a _Build Release_, that performs the same inspection and additional integration tests, producing a deployable artifact.  

Further build configurations (in purple) form a _deployment pipeline_, promoting the artifact across environments once checks are performed, increasing the artifact's degree of confidence. In each of those stages, the deployed artifact is validated on the environment. In a peculiarity, a requirement demands that only an agent on a given environment is able to perfom checks such as system-wide acceptance tests, so agent requirements will be needed.  

Each stage is represented by a different _Build Configuration_ in TeamCity, chained together with the help of _Build Triggers_ and _Dependencies_. The final step, _Deploy to Production_ would be manually triggered by an user, in this example. _Continuous Deployment_ is enabled automatically up to the Staging environment.

### Full resource graph
Representing fully this workflow in Terraform, would require the following resources:

- Project
- VCS Root
- Build Configuration
- Commit Status Publisher
- Snapshot Dependencies
- Build Trigger - VCS
- Build Trigger - Finish Build
- Agent Requirements
- Several parameters for interacting with Github, Deployment Tool and external dependencies

{{< image classes="fancybox fig-100 clear" src="/images/apply_teamcity.png" title="a sample apply of the configuration">}}  

And the result in TeamCity interface:

{{< image classes="fancybox fig-100 clear" src="/images/created_pipeline.png" title="complete pipeline">}}

### Show me the code!
All the code for the previous example can be found on <a href="https://github.com/cvbarros/terraform-teamcity-samples"><i class="fa fa-lg fa-github"></i></a>

# Conclusion
Having your pipelines defined in code can greatly improve the quality, consistency, reproducibility and maintainability. It creates the right atmosphere for a team to automate the automation. Heck, you can even have the server build its own pipelines that were defined in code. How :sunglasses: is that?
This practice allows a whole different class of improvements such as linting, testing, code generation and reuse, that wouldn't be possible otherwise.  
For next posts we'll dive into how to compose pipeline features by creating abstractions and how to automate several systems together using Terraform.  

<a href="https://github.com/cvbarros/terraform-provider-teamcity" target="_new">The provider project is available in <i class="fa fa-lg fa-github"></i></a>.  
Feel free to contribute by opening issues, pull requests and giving feedback!

Happy automating!

_Feedback for this article? Use the comments below.  
Want to receive updates?_ RSS <a class="sidebar-button-link" href="/index.xml"><i class="sidebar-button-icon fa fa-lg fa-rss"></i></a>  
{{< twitter_followme >}} 
