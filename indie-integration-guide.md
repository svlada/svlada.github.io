# Indie/Old-School Blog Theme Integration Guide

## Available Themes

1. **Terminal Green Theme** (`indie.css`) - Classic green phosphor terminal look
2. **Amber Monitor Theme** (`indie-amber.css`) - Warm amber CRT monitor aesthetic

## Quick Integration

To use these themes with your existing blog, replace the Tachyons CSS link with one of the indie themes:

```html
<!-- Replace this: -->
<link rel="stylesheet" href="https://unpkg.com/tachyons@4.12.0/css/tachyons.min.css">

<!-- With this (for green theme): -->
<link rel="stylesheet" href="/css/indie.css">

<!-- Or this (for amber theme): -->
<link rel="stylesheet" href="/css/indie-amber.css">
```

## HTML Structure Updates

For the best experience, wrap your content in the indie container:

```html
<body>
  <div class="indie-container">
    <!-- Your content here -->
  </div>
</body>
```

## Optional Enhancements

### ASCII Art Header
```html
<pre class="ascii-header">
Your ASCII art here
</pre>
```

### Terminal-style Boxes
```html
<div class="ascii-box">
  <h2>Section Title</h2>
  <p>Content here</p>
</div>
```

### Retro Buttons
```html
<button class="retro-button">CLICK ME</button>
<!-- or -->
<button class="dos-button">DOS STYLE</button>
```

### Typewriter Effect
```html
<p class="typewriter">This text will type out<span class="cursor"></span></p>
```

### Glitch Effect Headers
```html
<h1 class="glitch" data-text="GLITCHY TITLE">GLITCHY TITLE</h1>
```

## Features Included

- CRT scanline effects
- Phosphor glow on text
- Blinking cursor animations
- ASCII box decorations
- Terminal-style navigation
- Retro color schemes
- Mobile responsive
- No JavaScript required (except optional effects)

## Customization

Edit the CSS variables in `:root` to customize colors:

```css
:root {
  --bg-main: #0c0c0c;
  --text-primary: #00ff00;
  /* etc... */
}
```

## Demo

Check out `indie-demo.html` for a full example implementation.