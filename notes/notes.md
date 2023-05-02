---
title: Notes
description: Short notes on programming and everyday life
tags:
  - notes
  - software-engineering
layout: layouts/notes.njk
permalink: "notes/index.html"
---
<p></p>

<hr/>
<div class="dt-published small-text" datetime="02 May, 2023">02 May, 2023</div>
<p>
I first thought about creating algorithm to find pidgions in video stream. After identifying it as a pidgion, the toy rocket launcher fires a missile that destroys the bird. Don't those rats that fly deserve a worse fate? I've chosen to follow a different path until I can get my hands on a missile launcher.

First idea was to create an algorithm to detect pidgions in video stream. Upon pidgion detection toy rocket launcher would fire missle and destroy that ball of feathers. Those flying rats doesnt deserve anything better than that, right? However, I've opted-in for another strategy, until i find missle launcher on the black market.
</p>


<hr/>
<div class="dt-published small-text" datetime="01 May, 2023">01 May, 2023</div>
<p>
Domain/business logic code must be separated from technology. It matters. 

I'm not even mentioning the intriguing ability to swap technos quickly. No. Separate domain and infra-tech code. It’s important. 

<strong>Project structure</strong>

```shell
- **Application/Core**
  + Defines
    + UseCase interface 
    + UseCase implementation 
      + Repository interfaces specified by the **Domain**
    + Command objects are self-validating 
  + References
    + Domain project
- **Domain**
  + Entities
    + Self-validating
  + SPI
    + **Output Port** - Repository interface
- **Infrastructure**
  +  Adapters
    +  Rest
      +  Includes application module
      +  Spring Controllers
        +  DTO Validation
        +  Create command and invoke use case
    +  Persistence
      +  Includes domain module
      +  Implement repository using JDBI
- **Bootstrap**
  +  SpringBootApplication launcher
  +  Maven project includes modules
  +  Load Application/Domain services into the 
  Spring context by scanning @DomainService annotations.
```

</p>
<hr/>