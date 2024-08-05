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
<canvas
id="canvas_@0"
width="@1"
height="@2"
style="border: 1px solid black; width: 100%; background: url('@3') center/cover no-repeat;">
</canvas>

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

function getMousePos(event) {
  const rect = canvas.getBoundingClientRect();
  const scaleX = canvas.width / rect.width;
  const scaleY = canvas.height / rect.height;
  return {
    x: (event.clientX - rect.left) * scaleX,
    y: (event.clientY - rect.top) * scaleY
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

canvas.addEventListener('mousedown', (event) => {
  drawing = true;
  const { x, y } = getMousePos(event);
  lastX = x;
  lastY = y;
  dots.add(JSON.stringify({ x, y, color }));
  publish();
});

canvas.addEventListener('mousemove', (event) => {
  if (!drawing) return;
  const { x, y } = getMousePos(event);
  dots.add(JSON.stringify({ x, y, lastX, lastY, color }));
  publish();
  drawLine(lastX, lastY, x, y, color);
  lastX = x;
  lastY = y;
});

canvas.addEventListener('mouseup', () => drawing = false);
canvas.addEventListener('mouseout', () => drawing = false);

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
@end


@Collaborative.dots: @Collaborative.dots_(@uid,@0,@1,@2)

@Collaborative.dots_
<canvas
id="canvas_@0"
width="@1"
height="@2"
style="border: 1px solid black; width: 100%; background: url('@3') center/cover no-repeat;">
</canvas>

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

canvas.addEventListener('click', function(event) {
    const rect = canvas.getBoundingClientRect();
    const scaleX = canvas.width / rect.width;
    const scaleY = canvas.height / rect.height;

    const x = (event.clientX - rect.left) * scaleX;
    const y = (event.clientY - rect.top) * scaleY;

    dots.add(JSON.stringify({x, y, color}));
    publish()

    drawDot(x, y, color);
});

function drawDot(x, y, color) {
    ctx.beginPath()
    ctx.arc(x, y, 5, 0, 2 * Math.PI)
    ctx.fillStyle = color
    ctx.fill()
}

function redrawDots() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    dots.forEach(dotString => {
        const dot = JSON.parse(dotString);
        drawDot(dot.x, dot.y, dot.color);
    });
}

LIA.classroom.on("connect", () => {
  setTimeout(function() {
    console.log("connected")
    LIA.classroom.publish("join_@0", null)
  }, 1000)
})

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
  publish()
})

console.log("painting")


console.log("painting on canvas_@0");
</script>
@end

-->

# Collaborative Drawing

Adds collaborative drawing to classrooms

@Collaborative.dots(640,320)

@Collaborative.lines(640,320,https://upload.wikimedia.org/wikipedia/commons/3/31/A_large_blank_world_map_with_oceans_marked_in_blue-edited.png)

## Implementation

<canvas
id="canvas"
width="640"
height="320"
style="border: 1px solid black; width: 100%; background: url('https://upload.wikimedia.org/wikipedia/commons/3/31/A_large_blank_world_map_with_oceans_marked_in_blue-edited.png') center/cover no-repeat;">
</canvas>

<script run-once="true">
const canvas = document.getElementById("canvas");
const ctx = canvas.getContext("2d");
const color = window.getRandomColor();
const dots = new Set();
let drawing = false;
let lastX = 0;
let lastY = 0;

function publish() {
  if (LIA.classroom.connected) {
    LIA.classroom.publish("dots", JSON.stringify(Array.from(dots)));
  }
}

function getMousePos(event) {
  const rect = canvas.getBoundingClientRect();
  const scaleX = canvas.width / rect.width;
  const scaleY = canvas.height / rect.height;
  return {
    x: (event.clientX - rect.left) * scaleX,
    y: (event.clientY - rect.top) * scaleY
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

canvas.addEventListener('mousedown', (event) => {
  drawing = true;
  const { x, y } = getMousePos(event);
  lastX = x;
  lastY = y;
  dots.add(JSON.stringify({ x, y, color, type: 'dot' }));
  publish();
});

canvas.addEventListener('mousemove', (event) => {
  if (!drawing) return;
  const { x, y } = getMousePos(event);
  dots.add(JSON.stringify({ x, y, lastX, lastY, color }));
  publish();
  drawLine(lastX, lastY, x, y, color);
  lastX = x;
  lastY = y;
});

canvas.addEventListener('mouseup', () => drawing = false);
canvas.addEventListener('mouseout', () => drawing = false);

LIA.classroom.on("connect", () => {
  setTimeout(() => {
    console.log("connected");
    LIA.classroom.publish("join", null);
  }, 1000);
});

LIA.classroom.subscribe("dots", (message) => {
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

LIA.classroom.subscribe("join", publish);

console.log("painting");
</script>
