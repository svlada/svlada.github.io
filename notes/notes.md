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
<div class="dt-published small-text" datetime="01 May, 2023">01 May, 2023</div>
<p>
Domain/business logic code must be separated from technology. It matters. 

I'm not even mentioning the intriguing ability to swap technos quickly. No. Separate domain and infra-tech code. It’s important. 

<strong>Project structure</strong>

```markdown
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