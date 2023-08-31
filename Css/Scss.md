# SCSS

## Variables/变量

```SCSS
// scss
$font-stack: Helvetica, sans-serif;
$primary-color: #333;

body {
    font: 100% $font-stack;
    color: $primary-color;
}
// css
body {
    font: 100% Helvetica, sans-serif;
    color: #333;
}
```

## Nesting/

```SCSS
nav {
    ul {
         margin: 0;
         padding: 0;
         list-style: none;
     }
    li { 
        display: inline-block;
    }
    a {
        display: block;
        padding: 6px 12px;
        text-decoration: none;
    }
}
nav ul {margin: 0;padding: 0;list-style: none;}
nav li {display: inline-block;}
nav a {display: block;padding: 6px 12px;text-decoration: none;}
```

## Partials and Modules

1. a Partials. (begin with underscore)

    ```SCSS
    // _base.scss
    $font-stack: Helvetica, sans-serif;
    $primary-color: #333;
    body {
        font: 100% $font-stack;
        color: $primary-color;
    }
    ```

2. a scss

   ``` scss
   // styles.scss
   @use 'base'; // when you use a Partials, you need't write _base.scss but base 
   
   .inverse {
     background-color: base.$primary-color;
     color: white;
   }
   ```

3. compiled output css

    ```css
    body {
      font: 100% Helvetica, sans-serif;
      color: #333;
    }
    
    .inverse {
      background-color: #333;
      color: white;
    }
    ```

## Mixins

```scss
// scss
@mixin theme($theme: DarkGray) {
  background: $theme;
  box-shadow: 0 0 1px rgba($theme, .25);
  color: #fff;
}

.info {
  @include theme;
}
.alert {
  @include theme($theme: DarkRed);
}
.success {
  @include theme($theme: DarkGreen);
}
```

``` css
// css
.info {
  background: DarkGray;
  box-shadow: 0 0 1px rgba(169, 169, 169, 0.25);
  color: #fff;
}

.alert {
  background: DarkRed;
  box-shadow: 0 0 1px rgba(139, 0, 0, 0.25);
  color: #fff;
}

.success {
  background: DarkGreen;
  box-shadow: 0 0 1px rgba(0, 100, 0, 0.25);
  color: #fff;
}
```

## Extend/Inheritance

```scss
/* This CSS will print because %message-shared is extended. */
%message-shared {
  border: 1px solid #ccc;
  padding: 10px;
  color: #333;
}

// This CSS won't print because %equal-heights is never extended.
%equal-heights {
  display: flex;
  flex-wrap: wrap;
}

.message {
  @extend %message-shared;
}

.success {
  @extend %message-shared;
  border-color: green;
}

.error {
  @extend %message-shared;
  border-color: red;
}

.warning {
  @extend %message-shared;
  border-color: yellow;
}
```

## Operators

Doing math in your CSS is very helpful. Sass has a handful of standard math operators like `+`, `-`, `*`, `math.div()`, and `%`. In our example we're going to do some simple math to calculate widths for an `article` and `aside`.

```scss
// scss
@use "sass:math";

.container {
  display: flex;
}

article[role="main"] {
  width: math.div(600px, 960px) * 100%;
}

aside[role="complementary"] {
  width: math.div(300px, 960px) * 100%;
  margin-left: auto;
}
```

```css
// css
.container {
  display: flex;
}

article[role="main"] {
  width: 62.5%;
}

aside[role="complementary"] {
  width: 31.25%;
  margin-left: auto;
}
```

