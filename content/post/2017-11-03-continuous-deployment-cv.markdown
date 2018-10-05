---
date: "2017-11-03T18:18:37Z"
title: Continuous deployment for your CV
tags:
- automation
---

The goal of today's post is to get a little whiff of Continuous Deployment,
especially build pipelines.

First off, I need you to realize that sending CVs by Email is not acceptable, because this is 2017. Are we on the same page? OK.

We're going to put our CV into version control and build a PDF out of it as soon as we push a new commit.

# Frequently Asked Questions
- Is this necessary? **No.**
- Are we doing it anyway? **Yes.**
- Why? **Firstly, because we can. Secondly, because it teaches us about automation.**

# The Plan

What are we specifically trying to achieve?
- Being able to change our published CV by pushing a new commit to a Git repository
- Having a stable URL at hand that always refers to the latest and greatest version of our CV

Secondary priorities:
- Not having to host anything ourselves (not because we can't, but because it's not necessary and less effort)

# Automatic building of PDF
Our goal is to push a Git commit and somehow get a PDF out of it.

To keep it super simple, we are looking for a Git repository provider offering build pipelines so we don't have to wire up Hooks to trigger build jobs. I considered [Bitbucket](https://bitbucket.org/) and [Gitlab](https://gitlab.com). Eventually chose Gitlab since Bitbucket severely limits the amount of available build time and makes public downloads harder.

Build pipelines are declarations of what needs to happen to build something. Usually they run automatically when a new push a Git repository happened. Our Gitlab Pipeline will do exactly that for us, build us a PDF whenever we push.

Here is a Gitlab pipeline definition.
{{< highlight yaml >}}
stages:
  - build

compile-cv:
  stage: build
  image: blang/latex
  script:
    - bash build-cv.sh
  artifacts:
    paths:
      - "build/CV.pdf"
{{< /highlight >}}

**Quick walkthrough:**
1. `stages` just allows grouping your pipeline into stages - what a shocker.
   That's pretty cool for running things that don't depend on each other in parallel.

2. `compile-cv` is a "step" in the build that is run in a Docker container
    - based on the Docker image `blang/latex`, which provides a LaTeX build environment
    - running a shell script that basically just executes `pdflatex`
    - defining the `pdflatex` result - `build/CV.pdf` - as an `artifact`

[Job artifacts](https://docs.gitlab.com/ce/user/project/pipelines/job_artifacts.html) in Gitlab pipelines are super neat, since they are downloadable, and allow us to link to the most recent artifact under a stable URL!

# Making the PDF publically available
This pretty much just requires you to set the Gitlab repository visibility to `public`. I'm sure you'll figure out how to do that if you really want to.

Eventually, you will be able to call an URL that looks a bit like this, and get a PDF download (**Rockstar developer tip:** Log out of Gitlab to see if it is actually publically available).
{{< highlight html >}}
https://gitlab.com/jfornoff/cv/-/jobs/artifacts/master/raw/build/CV.pdf?job=compile-cv
{{< /highlight >}}

Obviously your URL will vary, the general scheme is:
{{< highlight html >}}
https://gitlab.com/<namespace>/<project>/-/jobs/artifacts/<ref>/raw/<path_to_file>?job=<job_name>
{{< /highlight >}}

# Getting a pretty URL
This URL is totally not memorable and you'll need to look it up, we can do better.

I'll give you two options that might work for you:

#### 1. URL shortener
If you just want the quick-and-not-quite-nice version of the publically available, Continuous Deployment, DevOps, Microservices, Cloud, Disruption CV, then just go for something like [TinyUrl](https://tinyurl.com/), yielding something like:
{{< highlight html >}}
https://tinyurl.com/jfornoff-cv
{{< /highlight >}}

#### 2. Using your own domain

Serving files directly from the Gitlab artifact under a nice URL usually requires you to run a HTTP server somewhere, redirecting or URL rewriting to your artifact URL.

You say that's too much effort? I agree.
After some digging around, I stumbled upon [redirect.name](http://redirect.name/).

It allows you to configure a DNS domain to use `TXT` entries for routing redirections, they host everything for you.

Assume you own `greatdomain.com`, and would like to have `cv.greatdomain.com` be the link to your hosted CV.

**Step 1:**

In this case, you would set a so-called `CNAME` record to make `cv.greatdomain.com` resolve to a `redirect.name` IP address (since we want them to take care of the redirection).

**Two-sentence intro to DNS and `CNAME` records:**
- DNS associates ("resolves") `google.com` (hostname) to `172.217.22.78` (IP address)
- `CNAME` tells a DNS lookup "but our princess is in another castle - here is another hostname to resolve to get an actual IP address" (Sorry.)

Here's how it would look:
{{< highlight dns >}}
cv.greatdomain.com.      <TTL>   IN      CNAME   alias.redirect.name.
{{< /highlight >}}

**Step 2:**

`redirect.name` does not know how to redirect yet. We tell them in a `TXT` DNS record.

**One-sentence intro to `TXT` records:** There's nothing to know, they just allow you to store some arbitrary text under a certain hostname. `redirect.name` uses it to read your routing definitions from there.

That would give you something along the lines of this:
{{< highlight dns >}}
_redirect.cv.greatdomain.com. <TTL> IN   TXT     "Redirects to https://gitlab.com/gitlabuser/my-cv/-/jobs/artifacts/master/raw/build/CV.pdf?job=compile-cv"
{{< /highlight >}}

Now you can visit `cv.greatdomain.com`, and you get redirected to your downloadable PDF hosted in Gitlab!

## Conclusion
I'll be honest, this didn't save any time, probably not even in the long run. However, it shows how to be deliberately intolerant to repetitive manual steps, which I personally consider a productive habit.

Hope you liked it and until next time!

## Appendix
If I handwaved over something, you can find all the pipeline code for building a CV from LaTeX [in my public CV repository](https://gitlab.com/jfornoff/cv).

