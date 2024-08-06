<!--
author:  André Dietrich

email:   LiaScript@web.de

version: 0.0.1

comment: This is a simple example for a collaborative drawing tool, that can be
         used in classrooms. It is based on the idea of a shared whiteboard,
         where students can draw together.

persistent: true

@onload
window.getRandomColor = function() {
  return `rgb(${Math.floor(Math.random() * 256)}, ${Math.floor(Math.random() * 256)}, ${Math.floor(Math.random() * 256)})`
}

window.isSubSet = function(A, B) {
  return [...A].every(element => B.has(element))
}
@end

@Collaborative.lines: @Collaborative.lines_(@uid,@0,@1,@2)

@Collaborative.lines_
<script run-once="true">
const canvas = document.getElementById("canvas_@0");
const ctx = canvas.getContext("2d");
const color = window.getRandomColor();
const dots = new Set();
let drawing = false;
let lastX = 0;
let lastY = 0;

function publish() {
  if (LIA.classroom.connected) {
    LIA.classroom.publish("dots_@0", JSON.stringify(Array.from(dots)));
  }
}

function getPos(event) {
  const rect = canvas.getBoundingClientRect();
  const scaleX = canvas.width / rect.width;
  const scaleY = canvas.height / rect.height;
  let clientX, clientY;

  if (event.touches) {
    clientX = event.touches[0].clientX;
    clientY = event.touches[0].clientY;
  } else {
    clientX = event.clientX;
    clientY = event.clientY;
  }

  return {
    x: (clientX - rect.left) * scaleX,
    y: (clientY - rect.top) * scaleY
  };
}

function drawLine(x1, y1, x2, y2, color) {
  ctx.beginPath();
  ctx.moveTo(x1, y1);
  ctx.lineTo(x2, y2);
  ctx.strokeStyle = color;
  ctx.lineWidth = 4;
  ctx.stroke();
}

function redrawDots() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  dots.forEach(dotString => {
    const dot = JSON.parse(dotString);
    drawLine(dot.lastX, dot.lastY, dot.x, dot.y, dot.color);
  });
}

function startDrawing(event) {
  drawing = true;
  const { x, y } = getPos(event);
  lastX = x;
  lastY = y;
  dots.add(JSON.stringify({ x, y, color }));
  publish();
}

function draw(event) {
  if (!drawing) return;
  const { x, y } = getPos(event);
  dots.add(JSON.stringify({ x, y, lastX, lastY, color }));
  publish();
  drawLine(lastX, lastY, x, y, color);
  lastX = x;
  lastY = y;
  if (event.touches) event.preventDefault(); // Prevent scrolling on touch devices
}

function stopDrawing() {
  drawing = false;
}

canvas.addEventListener('mousedown', startDrawing);
canvas.addEventListener('mousemove', draw);
canvas.addEventListener('mouseup', stopDrawing);
canvas.addEventListener('mouseout', stopDrawing);

canvas.addEventListener('touchstart', startDrawing);
canvas.addEventListener('touchmove', draw);
canvas.addEventListener('touchend', stopDrawing);
canvas.addEventListener('touchcancel', stopDrawing);

LIA.classroom.on("connect", () => {
  setTimeout(() => {
    console.log("connected");
    LIA.classroom.publish("join_@0", null);
  }, 1000);
});

LIA.classroom.subscribe("dots_@0", (message) => {
  const receivedDots = new Set(JSON.parse(message));
  const allDots = new Set([...dots, ...receivedDots]);

  if (!window.isSubSet(dots, receivedDots)) {
    receivedDots.forEach(dot => dots.add(dot));
    publish();
  } else {
    receivedDots.forEach(dot => dots.add(dot));
  }

  redrawDots();
});

LIA.classroom.subscribe("join_@0", publish);

console.log("painting on canvas_@0");
</script>
<canvas
  id="canvas_@0"
  width="@1"
  height="@2"
  style="border: 1px solid black; width: 100%; background: url('@3') center/cover no-repeat;">
</canvas>
@end

@Collaborative.dots: @Collaborative.dots_(@uid,@0,@1,@2)

@Collaborative.dots_
<script run-once="true">
const canvas = document.getElementById("canvas_@0");
const ctx = canvas.getContext("2d");
const color = window.getRandomColor();
const dots = new Set();

function publish() {
  if (LIA.classroom.connected) {
    LIA.classroom.publish("dots_@0", JSON.stringify(Array.from(dots)));
  }
}

function getPos(event) {
  const rect = canvas.getBoundingClientRect();
  const scaleX = canvas.width / rect.width;
  const scaleY = canvas.height / rect.height;
  let clientX, clientY;

  if (event.touches) {
    clientX = event.touches[0].clientX;
    clientY = event.touches[0].clientY;
  } else {
    clientX = event.clientX;
    clientY = event.clientY;
  }

  return {
    x: (clientX - rect.left) * scaleX,
    y: (clientY - rect.top) * scaleY
  };
}

function drawDot(x, y, color) {
  ctx.beginPath();
  ctx.arc(x, y, 5, 0, 2 * Math.PI);
  ctx.fillStyle = color;
  ctx.fill();
}

function redrawDots() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  dots.forEach(dotString => {
    const dot = JSON.parse(dotString);
    drawDot(dot.x, dot.y, dot.color);
  });
}

function handleClick(event) {
  const { x, y } = getPos(event);
  dots.add(JSON.stringify({ x, y, color }));
  publish();
  drawDot(x, y, color);
}

canvas.addEventListener('click', handleClick);
canvas.addEventListener('touchstart', function(event) {
  handleClick(event);
  event.preventDefault(); // Prevent scrolling on touch devices
});

LIA.classroom.on("connect", () => {
  setTimeout(function() {
    console.log("connected");
    LIA.classroom.publish("join_@0", null);
  }, 1000);
});

LIA.classroom.subscribe("dots_@0", (message) => {
  const receivedDots = new Set(JSON.parse(message));
  const allDots = new Set([...dots, ...receivedDots]);

  if (!window.isSubSet(dots, receivedDots)) {
    receivedDots.forEach(dot => dots.add(dot));
    publish();
  } else {
    receivedDots.forEach(dot => dots.add(dot));
  }

  redrawDots();
});

LIA.classroom.subscribe("join_@0", (message) => {
  publish();
});

console.log("painting on canvas_@0");
</script>
<canvas
  id="canvas_@0"
  width="@1"
  height="@2"
  style="border: 1px solid black; width: 100%; background: url('@3') center/cover no-repeat;">
</canvas>
@end

-->

# Collaborative Drawing

    --{{0}}--
This is a simple example for a collaborative drawing tool, that can be used in classrooms.
It is based on the idea of a shared whiteboard, where students can draw together and uses the LiaScript publish-subscribe mechanism to synchronize the drawings.
Currently it has support for 2 macros, one allows to draw lines and the other one to draw dots.
Both macros can be used with or without a background image.

**Try it on LiaScript:**

https://liascript.github.io/course/?https://raw.githubusercontent.com/LiaTemplates/CollaborativeDrawing/main/README.md

See the project on Github:

https://github.com/LiaTemplates/CollaborativeDrawing

    --{{1}}--
There are three ways to use this template.
The easiest way is to use the import statement and the url of the raw text-file of the master branch or any other branch or version.
But you can also copy the required functionality directly into the header of your Markdown document, see therefor the last slide.
And of course, you could also clone this project and change it, as you wish.

      {{1}}

- Load the macros via and set the persistence flag to true.
  Persistent will guarantee, that your drawings will be updated, even if you are on another slide.

  ```text
  import: https://raw.githubusercontent.com/LiaTemplates/CollaborativeDrawing/main/README.md

  persistent: true
  ```

  or use the tagged version:

  ```text
  import: https://raw.githubusercontent.com/LiaTemplates/CollaborativeDrawing/0.0.1/README.md

  persistent: true
  ```

- Copy the definitions into your Project

- Clone this repository on GitHub

## `@Collaborative.lines`

    --{{0}}--
This adds a new canvas to your document, where you have to define the width and height, whereby the image is always scaled to the maximum available width.
For every connected user a new color will be generated randomly, so that you can see who is drawing what.

`@Collaborative.lines(width,height)`

    --{{1}}--
The following example shows a canvas with a width of 640 and a height of 320 pixels.

      {{1}}
@Collaborative.lines(640,320)

### With background image

    --{{0}}--
As a third and optional parameter, you can also add a background image to the canvas.

`@Collaborative.lines(width,height,url)`

`@Collaborative.lines(640,320,https://upload.wikimedia.org/wikipedia/commons/thumb/7/7e/Pieter_Brueghel_the_Elder_-_The_Dutch_Proverbs_-_Google_Art_Project.jpg/1280px-Pieter_Brueghel_the_Elder_-_The_Dutch_Proverbs_-_Google_Art_Project.jpg)`

@[Collaborative.lines(640,320)](https://upload.wikimedia.org/wikipedia/commons/thumb/7/7e/Pieter_Brueghel_the_Elder_-_The_Dutch_Proverbs_-_Google_Art_Project.jpg/1280px-Pieter_Brueghel_the_Elder_-_The_Dutch_Proverbs_-_Google_Art_Project.jpg)

    --{{1}}--
You can use also the alternative link-style for the background image.
This type of link is preferred, if you are dealing with relative paths.

      {{1}}
``` markdown
@[Collaborative.lines(640,320)](./img/example.jpg)
```

## `@Collaborative.dots`

    --{{0}}--
This is a simplification to the previous macro, where you can only draw dots, with click or touch events.
On a white canvas this might not be very useful, but if you add a background image, you can use it to mark certain points.
Also here you can apply the link style macro syntax, where the last parameter is the url of the background image.

`@Collaborative.dots(width,height)`

`@[Collaborative.dots(1000,500)](url)`

---

@[Collaborative.dots(640,320)](https://upload.wikimedia.org/wikipedia/commons/3/31/A_large_blank_world_map_with_oceans_marked_in_blue-edited.png)

## Implementation

    --{{0}}--
The implementation consists of three parts.
The first is simply a definition of some helper functions, that are used in the following two macros.

      {{0}}
``` html
persistent: true

@onload
window.getRandomColor = function() {
  return `rgb(${Math.floor(Math.random() * 256)}, ${Math.floor(Math.random() * 256)}, ${Math.floor(Math.random() * 256)})`
}

window.isSubSet = function(A, B) {
  return [...A].every(element => B.has(element))
}
@end
```

    --{{1}}--
Collaborative drawing of lines defines two macros, where the first calls the second one passes all parameters, but also adds `@uid` to generate a unique id, which is then used to separate the different canvases and api calls.
Thus, a canvas is created with the size of the given parameters and a background image, if provided.
The script access this canvas and adds event listeners for mouse and touch events, which are used to draw lines on the canvas.
The drawing is synchronized with the other users via the LiaScript publish-subscribe mechanism and by using a set, which stores all the dots that are drawn, which are then send to the other users.
In this case, this can be seen as a very simple form of a "Conflict-free Replicated Data Type" (CRDT), where the order of the operations does not matter.

      {{1}}
``` html
@Collaborative.lines: @Collaborative.lines_(@uid,@0,@1,@2)

@Collaborative.lines_
<script run-once="true">
const canvas = document.getElementById("canvas_@0");
const ctx = canvas.getContext("2d");
const color = window.getRandomColor();
const dots = new Set();
let drawing = false;
let lastX = 0;
let lastY = 0;

function publish() {
  if (LIA.classroom.connected) {
    LIA.classroom.publish("dots_@0", JSON.stringify(Array.from(dots)));
  }
}

function getPos(event) {
  const rect = canvas.getBoundingClientRect();
  const scaleX = canvas.width / rect.width;
  const scaleY = canvas.height / rect.height;
  let clientX, clientY;

  if (event.touches) {
    clientX = event.touches[0].clientX;
    clientY = event.touches[0].clientY;
  } else {
    clientX = event.clientX;
    clientY = event.clientY;
  }

  return {
    x: (clientX - rect.left) * scaleX,
    y: (clientY - rect.top) * scaleY
  };
}

function drawLine(x1, y1, x2, y2, color) {
  ctx.beginPath();
  ctx.moveTo(x1, y1);
  ctx.lineTo(x2, y2);
  ctx.strokeStyle = color;
  ctx.lineWidth = 4;
  ctx.stroke();
}

function redrawDots() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  dots.forEach(dotString => {
    const dot = JSON.parse(dotString);
    drawLine(dot.lastX, dot.lastY, dot.x, dot.y, dot.color);
  });
}

function startDrawing(event) {
  drawing = true;
  const { x, y } = getPos(event);
  lastX = x;
  lastY = y;
  dots.add(JSON.stringify({ x, y, color }));
  publish();
}

function draw(event) {
  if (!drawing) return;
  const { x, y } = getPos(event);
  dots.add(JSON.stringify({ x, y, lastX, lastY, color }));
  publish();
  drawLine(lastX, lastY, x, y, color);
  lastX = x;
  lastY = y;
  if (event.touches) event.preventDefault(); // Prevent scrolling on touch devices
}

function stopDrawing() {
  drawing = false;
}

canvas.addEventListener('mousedown', startDrawing);
canvas.addEventListener('mousemove', draw);
canvas.addEventListener('mouseup', stopDrawing);
canvas.addEventListener('mouseout', stopDrawing);

canvas.addEventListener('touchstart', startDrawing);
canvas.addEventListener('touchmove', draw);
canvas.addEventListener('touchend', stopDrawing);
canvas.addEventListener('touchcancel', stopDrawing);

LIA.classroom.on("connect", () => {
  setTimeout(() => {
    console.log("connected");
    LIA.classroom.publish("join_@0", null);
  }, 1000);
});

LIA.classroom.subscribe("dots_@0", (message) => {
  const receivedDots = new Set(JSON.parse(message));
  const allDots = new Set([...dots, ...receivedDots]);

  if (!window.isSubSet(dots, receivedDots)) {
    receivedDots.forEach(dot => dots.add(dot));
    publish();
  } else {
    receivedDots.forEach(dot => dots.add(dot));
  }

  redrawDots();
});

LIA.classroom.subscribe("join_@0", publish);

console.log("painting on canvas_@0");
</script>
<canvas
  id="canvas_@0"
  width="@1"
  height="@2"
  style="border: 1px solid black; width: 100%; background: url('@3') center/cover no-repeat;">
</canvas>
@end
```

    --{{2}}--
The second macro is a simplification of the first one, where only dots can be drawn.
The script is almost the same, but the drawing function is different and only draws dots on the canvas.

      {{2}}
``` html
@Collaborative.dots: @Collaborative.dots_(@uid,@0,@1,@2)

@Collaborative.dots_
<script run-once="true">
const canvas = document.getElementById("canvas_@0");
const ctx = canvas.getContext("2d");
const color = window.getRandomColor();
const dots = new Set();

function publish() {
  if (LIA.classroom.connected) {
    LIA.classroom.publish("dots_@0", JSON.stringify(Array.from(dots)));
  }
}

function getPos(event) {
  const rect = canvas.getBoundingClientRect();
  const scaleX = canvas.width / rect.width;
  const scaleY = canvas.height / rect.height;
  let clientX, clientY;

  if (event.touches) {
    clientX = event.touches[0].clientX;
    clientY = event.touches[0].clientY;
  } else {
    clientX = event.clientX;
    clientY = event.clientY;
  }

  return {
    x: (clientX - rect.left) * scaleX,
    y: (clientY - rect.top) * scaleY
  };
}

function drawDot(x, y, color) {
  ctx.beginPath();
  ctx.arc(x, y, 5, 0, 2 * Math.PI);
  ctx.fillStyle = color;
  ctx.fill();
}

function redrawDots() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  dots.forEach(dotString => {
    const dot = JSON.parse(dotString);
    drawDot(dot.x, dot.y, dot.color);
  });
}

function handleClick(event) {
  const { x, y } = getPos(event);
  dots.add(JSON.stringify({ x, y, color }));
  publish();
  drawDot(x, y, color);
}

canvas.addEventListener('click', handleClick);
canvas.addEventListener('touchstart', function(event) {
  handleClick(event);
  event.preventDefault(); // Prevent scrolling on touch devices
});

LIA.classroom.on("connect", () => {
  setTimeout(function() {
    console.log("connected");
    LIA.classroom.publish("join_@0", null);
  }, 1000);
});

LIA.classroom.subscribe("dots_@0", (message) => {
  const receivedDots = new Set(JSON.parse(message));
  const allDots = new Set([...dots, ...receivedDots]);

  if (!window.isSubSet(dots, receivedDots)) {
    receivedDots.forEach(dot => dots.add(dot));
    publish();
  } else {
    receivedDots.forEach(dot => dots.add(dot));
  }

  redrawDots();
});

LIA.classroom.subscribe("join_@0", (message) => {
  publish();
});

console.log("painting on canvas_@0");
</script>
<canvas
  id="canvas_@0"
  width="@1"
  height="@2"
  style="border: 1px solid black; width: 100%; background: url('@3') center/cover no-repeat;">
</canvas>
@end
```
