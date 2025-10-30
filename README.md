# secodn-dsjl
adding new one for continue
<!doctype html>
<html lang="per">
<head>
  <meta charset="utf-8" />
  <title>Plane Following Birds (p5.js)</title>
  <style>
    html,body { margin:0; padding:0; height:100%; background:#111; color:#eee; font-family:sans-serif; }
    #info { position: absolute; left:10px; top:10px; z-index:10; background: rgba(0,0,0,0.35); padding:8px 12px; border-radius:6px; }
    a { color: #8fd3ff; }
  </style>
  <script src="https://cdn.jsdelivr.net/npm/p5@1.6.0/lib/p5.min.js"></script>
</head>
<body>
  <div id="info">
    <strong>Plane follows birds</strong><br>
    Press <kbd>C</kbd> to toggle: <span id="mode">center</span> — Drag mouse to steer flock bias, click to add a bird.
  </div>

<script>
/* Plane following birds - p5.js
   Save as index.html and open in browser.
   Controls:
     - C : toggle plane target mode (center / nearest)
     - Click : add a bird at mouse
     - Drag mouse : temporary attraction point for birds
*/

let boids = [];
let plane;
let mode = 'center'; // 'center' or 'nearest'
let mouseAttractor = null;

function setup(){
  createCanvas(windowWidth, windowHeight);
  // create flock
  for(let i=0;i<80;i++){
    boids.push(new Boid(random(width), random(height)));
  }
  plane = new Plane(width/2, height/2);
  textSize(13);
}

function draw(){
  background(18,22,30);

  // update & show boids
  for(let b of boids){
    b.flock(boids);
    if(mouseIsPressed && mouseAttractor) b.applyForce(p5.Vector.sub(mouseAttractor, b.pos).setMag(0.05));
    b.update();
    b.edges();
    b.show();
  }

  // Update plane target: either flock center or nearest bird
  let target;
  if(mode === 'center'){
    target = flockCenter();
  } else {
    target = nearestBoid(plane.pos) || flockCenter();
  }

  plane.pursue(target);
  plane.update();
  plane.edges();
  plane.show();

  // HUD
  fill(255,180);
  noStroke();
  textAlign(LEFT, TOP);
  text('Boids: ' + boids.length, 12, 60);
  text('Mode: ' + mode, 12, 78);
}

function keyPressed(){
  if(key === 'c' || key === 'C'){
    mode = (mode === 'center') ? 'nearest' : 'center';
    document.getElementById('mode').textContent = mode;
  }
}

function mousePressed(){
  // add a bird
  boids.push(new Boid(mouseX, mouseY));
  mouseAttractor = createVector(mouseX, mouseY);
}

function mouseDragged(){
  mouseAttractor = createVector(mouseX, mouseY);
}

function mouseReleased(){
  mouseAttractor = null;
}

// compute center of flock
function flockCenter(){
  let c = createVector(0,0);
  if(boids.length === 0) return c;
  for(let b of boids) c.add(b.pos);
  c.div(boids.length);
  return c;
}

// find nearest boid to a given position
function nearestBoid(pos){
  if(boids.length === 0) return null;
  let best = null;
  let bestD = Infinity;
  for(let b of boids){
    let d = p5.Vector.dist(pos, b.pos);
    if(d < bestD){ bestD = d; best = b; }
  }
  return best ? best.pos.copy() : null;
}

/* --------- Boid (bird) --------- */
class Boid {
  constructor(x,y){
    this.pos = createVector(x,y);
    this.vel = p5.Vector.random2D();
    this.vel.setMag(random(1,2.2));
    this.acc = createVector();
    this.maxForce = 0.06;
    this.maxSpeed = 3.2;
    this.r = 5;
  }

  applyForce(f){
    this.acc.add(f);
  }

  flock(boids){
    let sep = this.separate(boids).mult(1.6);
    let ali = this.align(boids).mult(1.0);
    let coh = this.cohesion(boids).mult(1.0);

    // slightly bias toward mouse attractor if present
    if(mouseAttractor){
      let m = p5.Vector.sub(mouseAttractor, this.pos);
      let d = m.mag();
      if(d < 200){
        m.setMag(map(d,0,200,0.1,0.6));
        this.applyForce(m);
      }
    }

    this.applyForce(sep);
    this.applyForce(ali);
    this.applyForce(coh);
  }

  update(){
    this.vel.add(this.acc);
    this.vel.limit(this.maxSpeed);
    this.pos.add(this.vel);
    this.acc.mult(0);
  }

  edges(){
    if(this.pos.x > width + 20) this.pos.x = -20;
    if(this.pos.x < -20) this.pos.x = width + 20;
    if(this.pos.y > height + 20) this.pos.y = -20;
    if(this.pos.y < -20) this.pos.y = height + 20;
  }

  seek(target){
    let desired = p5.Vector.sub(target, this.pos);
    desired.setMag(this.maxSpeed);
    let steer = p5.Vector.sub(desired, this.vel);
    steer.limit(this.maxForce);
    return steer;
  }

  separate(boids){
    let desiredSeparation = 20;
    let steer = createVector();
    let count = 0;
    for(let other of boids){
      let d = p5.Vector.dist(this.pos, other.pos);
      if(other !== this && d < desiredSeparation && d > 0){
        let diff = p5.Vector.sub(this.pos, other.pos);
        diff.normalize();
        diff.div(d);
        steer.add(diff);
        count++;
      }
    }
    if(count > 0){
      steer.div(count);
    }
    if(steer.mag() > 0){
      steer.setMag(this.maxSpeed);
      steer.sub(this.vel);
      steer.limit(this.maxForce);
    }
    return steer;
  }

  align(boids){
    let neighborDist = 48;
    let sum = createVector();
    let count = 0;
    for(let other of boids){
      let d = p5.Vector.dist(this.pos, other.pos);
      if(other !== this && d < neighborDist){
        sum.add(other.vel);
        count++;
      }
    }
    if(count > 0){
      sum.div(count);
      sum.setMag(this.maxSpeed);
      let steer = p5.Vector.sub(sum, this.vel);
      steer.limit(this.maxForce);
      return steer;
    }
    return createVector();
  }

  cohesion(boids){
    let neighborDist = 50;
    let sum = createVector();
    let count = 0;
    for(let other of boids){
      let d = p5.Vector.dist(this.pos, other.pos);
      if(other !== this && d < neighborDist){
        sum.add(other.pos);
        count++;
      }
    }
    if(count > 0){
      sum.div(count);
      return this.seek(sum);
    }
    return createVector();
  }

  show(){
    // draw as small triangle pointing along velocity
    let theta = this.vel.heading() + radians(90);
    push();
    translate(this.pos.x, this.pos.y);
    rotate(theta);
    noStroke();
    fill(230, 190, 90);
    beginShape();
      vertex(0, -this.r*2);
      vertex(-this.r, this.r*2);
      vertex(this.r, this.r*2);
    endShape(CLOSE);
    pop();
  }
}

/* --------- Plane --------- */
class Plane {
  constructor(x,y){
    this.pos = createVector(x,y);
    this.vel = createVector(0,0);
    this.acc = createVector(0,0);
    this.maxSpeed = 5.4;
    this.maxForce = 0.12;
    this.size = 26;
  }

  pursue(target){
    if(!target) return;
    // predictive pursuit: estimate where target will be
    let desired = p5.Vector.sub(target, this.pos);
    let dist = desired.mag();
    let speed = this.maxSpeed;
    // lookahead time proportional to distance
    let lookAhead = constrain(dist / 60, 0, 2.5);
    // If target is a boid position, try to anticipate by using its velocity if available
    // We'll simply nudge target forward in direction from plane to target
    let anticipated = p5.Vector.add(target, p5.Vector.mult(p5.Vector.sub(target, this.pos).normalize(), lookAhead * 30));
    this.applyForce(this.seek(anticipated));
  }

  seek(target){
    let desired = p5.Vector.sub(target, this.pos);
    desired.setMag(this.maxSpeed);
    let steer = p5.Vector.sub(desired, this.vel);
    steer.limit(this.maxForce);
    return steer;
  }

  applyForce(f){
    this.acc.add(f);
  }

  update(){
    this.vel.add(this.acc);
    this.vel.limit(this.maxSpeed);
    this.pos.add(this.vel);
    this.acc.mult(0);
  }

  edges(){
    // wrap around more gently
    if(this.pos.x > width + 50) this.pos.x = -50;
    if(this.pos.x < -50) this.pos.x = width + 50;
    if(this.pos.y > height + 50) this.pos.y = -50;
    if(this.pos.y < -50) this.pos.y = height + 50;
  }

  show(){
    push();
    translate(this.pos.x, this.pos.y);
    rotate(this.vel.heading() + PI/2);
    // body
    noStroke();
    fill(140, 200, 255);
    beginShape();
      vertex(0, -this.size);
      vertex(-this.size*0.6, this.size*0.6);
      vertex(0, this.size*0.2);
      vertex(this.size*0.6, this.size*0.6);
    endShape(CLOSE);
    // tail
    fill(100,160,220);
    triangle(-this.size*0.5, this.size*0.4, -this.size*1.1, this.size*0.2, -this.size*0.5, -this.size*0.2);
    // cockpit
    fill(20,30,50,180);
    ellipse(0, -this.size*0.45, this.size*0.6, this.size*0.35);
    pop();

    // draw line to target for debugging lightly
    // stroke(255,50); line(this.pos.x, this.pos.y, target.x, target.y);
  }
}

function windowResized(){
  resizeCanvas(windowWidth, windowHeight);
}
</script>
</body>
</html>
