
/*
# RN: Art + Animated + Hooks

## Introduction

brief study on fluid vector animations using RN Art + Animated + Hooks, you may also use some SVG and a graphic vector editor to play around

**you may also find this project on expo**
[https://exp.host/@rafaelnco/frontend-application-service-painter-v0a2](https://exp.host/@rafaelnco/frontend-application-service-painter-v0a2)

<img src="https://raw.githubusercontent.com/rafaelnco/painter/master/gifs/whatsapp.gif" width="24%"> <img src="https://raw.githubusercontent.com/rafaelnco/painter/master/gifs/bird1.gif" width="24%"> <img src="https://raw.githubusercontent.com/rafaelnco/painter/master/gifs/bird2.gif" width="24%"> <img src="https://raw.githubusercontent.com/rafaelnco/painter/master/gifs/bird3.gif" width="24%">

> notice the blue/red colors are actually gradients not supported by android

### Introduction: Get ready

> This README.md is actual valid `JavaScript` that you can just inject in a newly created expo project.

> This project sticks to _pure React Native_ solutions avoiding third-party

```
$ expo init project

$ cd project

$ wget https://raw.githubusercontent.com/rafaelnco/painter/master/README.md -O App.js

$ wget https://raw.githubusercontent.com/rafaelnco/painter/master/frames.json -O frames.json

$ expo start
```
---
## Principles

The first thing to think about animations in react native is that it's a problem already solved, one could easily download ready-to-go lottie animations on the web and that's about it. However, if we are willing to find a way around closed source then react native ART is the right way to go: underlying probably most of the available solutions we may find this same secret source to draw and animate vectors on mobile. Combine this with SVGs and dig that old graphic vector editor to compose some hand-made frames. RN Animated + Hooks will fit like a glove for everything running on latest RN versions.

### Principles: Animations, Vectors and JSON

Animations, Vectors and JSON are by far the most basic concepts we need to comprehend about how actual developers solve this problem in the market. A quick look at lootie's JSON animations may be the perfect hint for us right now as animations can be thought of as the traditional sequence of over positioned layers getting swaped into each other in a fast paced manner actually simulating an animation. The key concept here is that vectors are way more optimized than raster images, so my educated guess is that those SVG frames are being translated to JSON and with enough metadata the engine can reproduce each step of the animation.

### Principles: Morph

To get smooth animations going on we may get to know another important concept called morph: it is how our vectors are going to be transitioned into one another in a smooth way. RN Art is able to create each intermediary frame between two shapes (morphing them) and that can be used to simulate movement between frames, although this "movement" is strictly linear. However, there's nothing stopping us from doing the math to try to morph points in some rotation pattern instead of linearly, but more on that later. For now it's enough to know that for every vector frame we have in our animation we could animate transitions between corresponding points and use that to smooth transitions of whole frames.

### Principles: Simple Vector Graphics and RN Art

Once we get to know the basics of RN Art it's hardly unnoticeable how it resembles us of SVG. In fact RN Art lets us recreated entire SVG vectors with components like `Surface` (the RN Art version of `<svg>`) and `Shape` (the RN Art version of `<path>`). Althougth we are not able to just copy paste SVG contents directly on our JSX App render method, we can use the same path syntax for `<path d="..."` on `<Shape d="..."`  component prop to draw a path and even more complex forms.

### Principles: Animations on SVG

SVG may be not the perfect solution for animations on mobile (as it's not natively supported) but for now it's our best option for creating entire animations out of graphic vector editors that were not intended for that use. As for every quick study we will be using very tight constraints and simple assumptions about our solution:

- **Responsiveness**: we'll be scaling-to-fit our vector on the mobile screen, for that it's necessary to constraint file dimensions to known sizes

- **Single vector frame**: keeping tight on the "framed" concept, let us avoid parsing SVG and just understand each `<path>` as a frame so every frame has all it's vectors inside the same `path.d` string

- **Layered frames**: we're going to extract paths from SVG so try to order them from bottom-up (first-last)

- **Vector nodes edition order matters**: keep in mind that the resulting animation is hiper-sensible to the way you edit vectors on your graphic editor as it's the graphic editor writing your SVG

---
## Preparing

Before we begin implementing the mobile frontend we may need some resources to be presented on it

### Preparing: our first animation

Follow the steps to get your first two framed vector animation ready: 

1. Dig your best graphic vector editor

1. Set up a `100x100` document with `scale=1`

1. Import some single colored vector for our first initial test

1. Duplicate the just imported vector

1. Cut the duplicate vector in parts and apply a simple transformation to them before combining again (not necessarly merge paths but you could try that as well, results are better for isolated parts)

1. Center both vectors in the canvas and make sure they stack from bottom up

1. Export the file as Plain SVG

> Remember to export a plain svg file as some graphic vector editors may persist SVG on proprietary format and scramble things up for us

> Important: some editors won't apply transformations directly to paths when using groups, try to keep that in mind and avoid grouping paths, instead try to combine them

### Preparing: SVG-to-JSON

Next step is to extract the frames from the SVG, that's how fun begins. remember our assumption about each path to be a single frame and all we need to do is extract each path from our SVG file and persist is a JSON.

> Important: the following code is intended to be ran as a npm script to generate the JSON with extracted frames

> Note: yes, we could parse SVG directly on our application but let's stick to JSON as at it's already going to be used to agregate meta-data for each frame

`convert.js`:
```js
fs = require('fs')

const [
    processName,
    commandName,
    fileNameIn = 'drawing.svg',
    fileNameOut = 'frames.json'
] = process.argv

// plain svg file exported from your vector editor 
raw = fs.readFileSync(fileNameIn, 'utf-8')

// find path properties, can and must be optimized
rule = / d="(.*)"/g

// extracted paths list
paths = []

for(let match of raw.matchAll(rule))
    paths.push(match[1])

// write that JSON file
fs.writeFileSync(fileNameOut, JSON.stringify(paths))
```

Call the script as so
```
$ node convert justSavedFile.svg output.json
```


Now taht we have the basic assets to get started, let us begin implementing our mobile frontend presentation: from now on everything is React Native code.

---
## Implementation: dependencies

Starting with the basic dependencies for our quick study

```js */

/* React Hooks */
import React, { useState, useRef, useEffect } from 'react';

/* React Animated + Art */
import { View, ART, Text, Dimensions, Animated } from 'react-native';

/* React Art */
const { Surface, Shape } = ART

import Morph from 'art/morph/path';

/*
```
---

## Implementation: Loading frames, scaling to fit

That's how our algorithm loads animation frames persisted in JSON files and then scales it up to fit the full screen's width.

> remember assumptions taken on last sections

> important: 100 is not a random magic number, it's supposed to have a direct relation to the SVG dimensions

> we'll revisit this scaling algorithm later but it's enough for now that it can make the path fit the entire screen width

```js */

const { height, width } = Dimensions.get('screen')

/* Simple scale-to-fit algorithm assuming SVG width = 100 */
const scale = path => path.split(' ').map(part => {
  const factor = (width / 100)
  if(part.indexOf('.') !== -1) {
    return part.split(',').map(value =>
      factor * Number.parseFloat(value))
  } else
    return part
})

/* Load frames and apply scaling right away */
const frames = require('./frames.json').map(frame => scale(frame))

/*
```
---
## Implementation: State management

We're going to use a couple of states to manage our simple animation. 

> Please notice this may not be the most optimized way to go

```js */

export default function App() {

  /* Enable ART.Surface after mount */
  const [show, setShow] = useState(false)

  /* Frame transition parameter reference */
  const parameter = useRef(new Animated.Value(0))

  /* Frame transition parameter reference */
  const [parameterValue, setParameterValue] = useState(0)

  /* Frame transition current path */
  const [path, setPath] = useState(Morph.Tween(frames[0], frames[1]))

  /* Current frame reference */
  const frame = useRef(0)

/*
```
---
## Implementation: Effects

Two simple effects we need to set.

> important: why to enable surface _after mount_ https://github.com/facebook/react-native/issues/17565

### Effects: safe surface display + auto play

```js */

  useEffect(() => {
  
    /* Enable ART.Surface after mount */
    requestAnimationFrame(() => setShow(true))

    /* Animation auto-play once*/
    setTimeout(() => animate(frames.length-1), 500)

    /* Animation auto-play endless*/
    //setTimeout(() => animate(-1), 500)
  
  }, [])

/*
```

### Effects: morph + animated smooth transition

Transition between frames on the current animation parameter value

> remember: at this point one could try to use setNativeProps to tune performance

```js */

  useEffect(() => {
    parameter.current.addListener(({value}) => {
      requestAnimationFrame(() => {
        path.tween(value)
        setParameterValue(value)
      })
    })
  }, [parameter.current])

  /*
  ```
  ---
  ## Implementation: Animation steps
  ### Animation steps: trigger animation for N frames

  Use this to animate the next `steps` frames or repeat whole animation five times when called without arguments. One could also pass a negative number so that recursion goes endless.

  ```js */

  const animate = (steps = frames.length * 5) => {
    if(steps || steps < 0)
      nextFrame(() => animate(steps - 1))
  }

  /*
  ```
  
  ### Animation steps: calculate the next frame

  Here we have two types of configuration for our animation, one which loops and other that doesn't. You may want to experiment with this before choosing one or another.
  
  ```js */

const nextFrame = (callback) => {
    
    /* Last frame without loop on end */
    const newFrame =
      frame.current + 1 == frames.length?
        0 : frame.current + 1
    
    /* Last frame with loop on end */
    //const newFrame = (frame.current+1)%frames.length

  /*
  ```
  
  ### Animation steps: trigger smooth animation

  > if you're still reading you may want to take a look at this: https://en.wikipedia.org/wiki/Uncanny_valley
  
  ```js */
    /* Using spring to smooth frame transition */
    Animated.spring(
      parameter.current,
      {
        /* Avoid uncanny valley, optimize for vivid, fluid animations*/
        restDisplacementThreshold: 0.1, 
        restSpeedThreshold: 0.1,
        bounciness: 0.5,
        speed: 15,

        /* One frame at a time */
        toValue: 1
      },
    )

  /*
  ```
  
  ### Animation steps: prepare next transition

  > important: the following method is a *callback* and it is supposed to be executed on animation end, more precisely when an absolute frame is being reached and it needs to prepare to the next frame

  ```js */
      .start(() => {

        /* Prepare next frame to be morphed into */
        setPath(Morph.Tween(
          frames[newFrame],
          frames[Math.min(newFrame + 1, frames.length-1)]
        ))

        /* Reset the animated parameter */
        parameter.current = new Animated.Value(0)

        /* Point to the new frame index */
        frame.current = newFrame

        /* Invoke callback, which may treigger recursion */
        requestAnimationFrame(callback)
      })
  }

/*
```
---
## Implementation: Markup

Pretty basically what we need to accomplish simple animation presentation which won't bug on startup. Also, something that is able to display some loading indicator.

> Notice how `onTouchEnd` is used to replay the entire animation one more time.

```js */

  return (
    <View
      style={{
        flex: 1,
        alignItems: 'center',
        justifyContent: 'center'
      }}
      onTouchEnd={() => animate(frames.length)}
    >
    {
      show? (
        <Surface width={width} height={height}>
          <Shape fill='black' d={path} />
        </Surface>
      ) : (
        <Text style={{fontSize: 36}}>LOADING</Text>
      )
    }
    </View>
  );
}

/*
```
---
## That's it

Althought verborragic this demo may not cover all the corner cases for some hybrid approach on displaying animations on mobile applications. However, for this first chapter I think we've met our point of having some functional prototype for an open source animation solution that can be used as an alternative for a set of simple animations.

![](https://www.ciaoagenciadigital.com/images/high-bg.jpg)

*/