# Rigidbodies

<p>Experimental physics engine covering basic mechanical physics.</p>
<p>This engine tackles the following:</p>
<ul>
    <li>Forces (gravitational, normal, friction, etc)</li>
    <li>Impulse</li>
    <li>Torque (angular velocity + acceleration)</li>
    <li>Mass properties of loaded meshes (volume, center of mass, inertia tensor)</li>
</ul>

Convention: the contact normal $\hat{n}$ always points from object B toward object A. Two objects that are approaching each other therefore have $v_n < 0$, and A receives $+\vec{J}$ while B receives $-\vec{J}$.


### Forces

<p></p>

Forces can be denoted as:

$$ \vec{F} _{net} = \sum \vec{F} = m\vec{a} $$

(or $\vec{F} = m\frac{d^2\vec{x}}{dt^2}$ which can be helpful to calculate impulse) as described by Newton's 2nd law of motion.

<p></p>

Some forces include gravity: $\vec{F} = m \vec{g}$ , kinetic friction: $\vec{f}_k = -\mu _{k} N \hat{t}$ , and various damping forces. Here $\hat{t}$ is the unit vector along the tangential velocity, so friction always opposes the sliding direction and its magnitude is $\mu_k N$.

<p></p>

The gravitational force must be decomposed into separate tangential and normal components such that:

<p></p>
The normal component of gravitational acceleration is:

$$ \vec{g} _{\perp} = (\vec{g} \cdot \hat{n})\hat{n} $$

and the component of the tangential gravitational acceleration is:

$$ \vec{g} _{\parallel} = \vec{g} - \vec{g} _{\perp} $$

Therefore, the force trying to slide an object on some inclined plane is $\vec{F} = m \vec{g} _{\parallel}$.

<p></p>

*The contact normal $\hat{n}$ (and the penetration depth) is derived from the collision detection step. GJK only tells us whether two convex shapes intersect, so it is followed by EPA (Expanding Polytope Algorithm) which expands the final simplex to find the normal and depth needed to push the objects apart.

<p></p>

Then assuming the two objects stay in contact (resting contact), the magnitude of the normal force $N = m|\vec{g} \cdot \hat{n}|$, and the friction force is $f = \mu_k N$ opposing the tangential velocity.

Force-based friction is only used for resting/sliding contact. During the moment of a collision, friction is handled with impulses instead (see below).

### Applying Forces Over Time

<p></p>

Forces are not applied to an object right away, they are collected in accumulators during a step and then integrated. For each object, every force $\vec{F}_j$ applied at point $\vec{p}_j$ adds to the net force and net torque:

$$ \vec{F} _{net} = \sum_j \vec{F}_j + \vec{F} _{drag} \qquad \vec{\tau} _{net} = \sum_j \vec{r}_j \times \vec{F}_j + \vec{\tau} _{drag}, \quad \vec{r}_j = \vec{p}_j - \vec{x} _{COM} $$

where $\vec{F} _{drag}$ and $\vec{\tau} _{drag}$ are the linear and angular drag defined in the damping part below.

A force applied through the center of mass has $\vec{r}_j = \vec{0}$ and produces no torque, and gravity always acts through the center of mass.

<p></p>

The accelerations are then:

$$ \vec{a} = m^{-1}\vec{F} _{net} \qquad \vec{\alpha} = I^{-1}\left(\vec{\tau} _{net} - \vec{\omega} \times (I\vec{\omega})\right) $$

where $I^{-1}$ is the world space inverse inertia tensor, so a static object ($m^{-1} = 0$ and $I^{-1} = 0$) has no acceleration.

<p></p>

We integrate with semi-implicit Euler, meaning the position uses the new velocity, which is more stable than plain Euler:

$$ \vec{v} _{i+1} = \vec{v}_i + \vec{a}\Delta t \qquad \vec{x} _{i+1} = \vec{x}_i + \vec{v} _{i+1}\Delta t $$

$$ \vec{\omega} _{i+1} = \vec{\omega}_i + \vec{\alpha}\Delta t $$

(this is the undamped form, drag is included in $\vec{a}$ and $\vec{\alpha}$ and the stable implicit version of it is given in the damping part below), and the orientation is updated with the quaternion step from the Torque section. After the step the accumulators are reset to $\vec{0}$.

<p></p>

#### Connection to kinematics

<p></p>

Integrating a constant acceleration twice gives the classic kinematics equations:

$$ \vec{v}(t) = \vec{v}_0 + \vec{a}t \qquad \vec{x}(t) = \vec{x}_0 + \vec{v}_0 t + \frac{1}{2}\vec{a}t^2 $$

Applied over one step, the exact update would be:

$$ \vec{x} _{i+1} = \vec{x}_i + \vec{v}_i\Delta t + \frac{1}{2}\vec{a}\Delta t^2 $$

The integrator we use is a variation of this. Substituting $\vec{v} _{i+1} = \vec{v}_i + \vec{a}\Delta t$ into the semi-implicit position update gives:

$$ \vec{x} _{i+1} = \vec{x}_i + \vec{v}_i\Delta t + \vec{a}\Delta t^2 $$

so the acceleration term is $\vec{a}\Delta t^2$ instead of $\frac{1}{2}\vec{a}\Delta t^2$. This error is small for a small $\Delta t$ and the method is stable, which is why it is preferred over explicit Euler (which drops the acceleration term completely).

If exact results for constant acceleration are needed, velocity Verlet can be used instead:

$$ \vec{x} _{i+1} = \vec{x}_i + \vec{v}_i\Delta t + \frac{1}{2}\vec{a}_i\Delta t^2 \qquad \vec{v} _{i+1} = \vec{v}_i + \frac{1}{2}(\vec{a}_i + \vec{a} _{i+1})\Delta t $$

The rotational version is the same: $\theta(t) = \theta_0 + \omega_0 t + \frac{1}{2}\alpha t^2$ for a constant angular acceleration.

<p></p>

Gravity is $\vec{F} = m\vec{g}$, which gives $\vec{a} = \vec{g}$ no matter the mass.

<p></p>

Damping stops objects from moving forever. Drag opposes the velocity, with a linear term and a stronger quadratic term:

$$ \vec{F} _{drag} = -k_1\vec{v} - k_2\lVert\vec{v}\rVert\vec{v} $$

Rotating objects need damping too, otherwise a spinning object in free space would spin forever. Angular drag is a torque opposing the angular velocity, with the same two terms:

$$ \vec{\tau} _{drag} = -\kappa_1\vec{\omega} - \kappa_2\lVert\vec{\omega}\rVert\vec{\omega} $$

Since both are part of $\vec{F} _{net}$ and $\vec{\tau} _{net}$, they enter the accelerations directly. Written out in full:

$$ \vec{a} = m^{-1}\left(\sum_j \vec{F}_j - k_1\vec{v} - k_2\lVert\vec{v}\rVert\vec{v}\right) $$

$$ \vec{\alpha} = I^{-1}\left(\sum_j \vec{r}_j \times \vec{F}_j - \kappa_1\vec{\omega} - \kappa_2\lVert\vec{\omega}\rVert\vec{\omega} - \vec{\omega} \times (I\vec{\omega})\right) $$

<p></p>

For a large $\Delta t$ or large coefficients the explicit linear terms can overshoot and reverse the velocity. To keep it stable the linear terms are applied implicitly instead: leave $k_1$ and $\kappa_1$ out of $\vec{a}$ and $\vec{\alpha}$ (call these $\vec{a}'$ and $\vec{\alpha}'$), then divide after the velocity step:

$$ \vec{v} _{i+1} = \frac{\vec{v}_i + \vec{a}'\Delta t}{1 + c_1\Delta t} \qquad \vec{\omega} _{i+1} = \frac{\vec{\omega}_i + \vec{\alpha}'\Delta t}{1 + c_a\Delta t} $$

Here $c_1$ and $c_a$ are damping rates in $1/s$, related to the coefficients by $c_1 = k_1/m$ and $c_a \approx \kappa_1/I$ (for a small $c\Delta t$ this matches the explicit version). With a rate every object slows at the same pace, while with a torque coefficient a heavier object (larger $I$) would slow down less, and a rate per object is easier to expose in the Qt widget. The quadratic terms stay explicit.

The damped $\vec{\omega} _{i+1}$ is the one used to update the orientation. Optionally, if $\lVert\vec{\omega}\rVert$ falls below a small threshold, set it to $\vec{0}$ so objects come fully to rest.

<p></p>

Damping from the air is not the only thing that slows a rotating object. When it touches the ground, the friction impulse removes spin around the contact tangent (so a rolling or sliding object slows down), but spin around the contact normal (like a coin spinning flat on a table) and rolling resistance are not covered by the Coulomb friction above. The simple fix is to rely on the angular damping rate for those cases.

<p></p>

#### How forces relate to impulses

A force held constant for one step is the same as an impulse:

$$ \vec{J} = \vec{F}\Delta t $$

This connects the two halves of the engine. When an object rests on a floor, gravity adds $\vec{g}\Delta t$ to its velocity every step, and the collision solver responds with a normal impulse that cancels it. That impulse is $J_n \approx m|\vec{g} \cdot \hat{n}|\Delta t$, so dividing by $\Delta t$ recovers the normal force $N = m|\vec{g} \cdot \hat{n}|$ from earlier. The friction limit $\mu|J_n|$ in the impulse section is then just $\mu N \Delta t$, the same Coulomb law written for a single step.

So the force version of friction ($f = \mu_k N$) describes a continuous contact (like an object sliding down an incline), while the impulse version is what is actually applied by the solver each step.

### Impulse and Angular Velocity

<p></p>

<p>Impulse occurs when objects collide and is generally calculated as follows:</p>

$$ J = \int Fdt $$

Though in this simulation a simpler approach is to treat impulse as the change in momentum: $\vec{J} = \Delta \vec{p} = m(\vec{v}_f - \vec{v}_i)$ (generally because we do not have an accumulation of force). The final velocity $\vec{v}_f$ is not known in advance, it is determined by the restitution condition below, which is what gives us $J_n$.

<p></p>

We first need the kinetic and static friction coefficients. And since every object can have their own friction, the coefficients are calculated as a geometric mean $\mu_k = \sqrt{\mu _{k,A}\mu _{k,B}}$ and $\mu_s = \sqrt{\mu _{s,A}\mu _{s,B}}$. The restitution of the pair is combined as $e = \max(e_A, e_B)$.

<p></p>

When each object collides, they produce angular velocities. For each object, the direction from the center of mass to the contact point is calculated:

$$ \vec{r} = \vec{p} - \vec{x} _{COM} $$

Where $\vec{p}$ is the contact point and $\vec{x} _{COM}$ is the center of mass.

We similarly have a relative and normal velocity. The objects may already be spinning at the moment before the collision, so $\vec{\omega}_A$ and $\vec{\omega}_B$ are whatever they were from the previous step.

*When the collision happens, the angular velocities are changed so the next frame/step, $\vec{\omega}_A$ and $\vec{\omega}_B$ would have been modified.

$$ \vec{v} _{rel} = (\vec{v}_A + \vec{\omega}_A \times \vec{r}_A) - (\vec{v}_B + \vec{\omega}_B \times \vec{r}_B) $$

and

$$ v_n = \vec{v} _{rel} \cdot \hat{n} $$

<p></p>

If $v_n \geq 0$ the objects are already separating and no impulse is applied. Also, if $|v_n|$ is below a small threshold then $e = 0$ so resting contacts do not bounce forever.

The impulse is derived from the restitution condition $v_n' = -e v_n$ (the normal velocity after the collision). When both objects contact it is:

$$ J_n = -\frac{(1 + e)v_n}{m_A^{-1} + m_B^{-1} + (\vec{r}_A \times \hat{n}) \cdot\bf I^{-1}_A \it(\vec{r}_A \times \hat{n}) + (\vec{r}_B \times \hat{n}) \cdot \bf I^{-1}_B \it(\vec{r}_B \times \hat{n})} $$

Where $\bf I$ is the inertia tensor (or 3x3 matrix) which describes how the object should rotate and $e$ is the restitution bound within $e \in [0, 1]$.

The inertia tensor is defined in the object's local (body) space, so every step it has to be rotated into world space using the object's rotation matrix $R$:

$$ \bf I \it ^{-1} _{world} = R \, \bf I \it ^{-1} _{body} R^T $$

<p></p>

$\bf I$ can be represented as (for a cuboid with side lengths $a,b,c$):

$$ 
\bf I \it =  \begin{bmatrix}
    \frac{m}{12}(b^2 + c^2) & 0 & 0 \\
    0 & \frac{m}{12}(a^2 + c^2) & 0 \\
    0 & 0 & \frac{m}{12}(a^2 + b^2)
\end{bmatrix}
$$

Otherwise for any other object the generalized way to compute the inertia matrix is:

$$ 
\bf I \it = \begin{bmatrix}
    \int \rho (y^2 + z^2)dV & -\int \rho xy \ dV & -\int \rho xz \ dV \\
    -\int \rho xy \ dV & \int \rho (x^2 + z^2) \ dV & -\int \rho yz \ dV \\
    -\int \rho xz \ dV & -\int \rho yz \ dV & \int \rho (x^2 + y^2)dV
\end{bmatrix}
$$

### Mass Properties of a Loaded Mesh

<p></p>

Instead of computing integrals by hand all the time, we can simplify the process by enclosing parts of the objects into tetrahedrons from some reference point and each triangle. This requires the mesh to be closed, with consistent outward-facing triangle winding, and a uniform density $\rho$.

For this section the reference point is the origin of the mesh, and each triangle with vertices $\vec{a}, \vec{b}, \vec{c}$ forms one tetrahedron $(\vec{0}, \vec{a}, \vec{b}, \vec{c})$.

<p></p>

Each tetrahedron has its own signed volume:

$$ V_i = \frac{\vec{a} \cdot (\vec{b} \times \vec{c})}{6} $$

The sign is important: triangles facing away from the reference point give a positive volume and triangles facing it give a negative volume, so the extra volume cancels out and the reference point does not even need to be inside the mesh.

<p></p>

Each tetrahedron has its own mass $m_i = \rho V_i$ and its own center of mass $\vec{x}_i = \frac{\vec{a} + \vec{b} + \vec{c}}{4}$ (the fourth vertex is the origin).

Therefore $M$ represents the sum of all masses:

$$ M = \sum_i m_i $$

and

$$ \vec{x} _{COM} = \frac{\sum_i m_i\vec{x}_i}{M} $$

<p></p>

For the inertia tensor, we use the second moment matrix (covariance) $\bf C$ of each tetrahedron, which has a closed form. With $\vec{s} = \vec{a} + \vec{b} + \vec{c}$:

$$ \bf C \it _i = \int \vec{x}\vec{x}^T dV = \frac{V_i}{20}\left(\vec{a}\vec{a}^T + \vec{b}\vec{b}^T + \vec{c}\vec{c}^T + \vec{s}\vec{s}^T\right) $$

Summing over every triangle gives the inertia tensor about the mesh origin, because the integrals in the generalized matrix above are exactly $\rho\left(\text{tr}(\bf C \it)\bf 1 - \bf C \it\right)$:

$$ \bf I \it _{origin} = \rho\left(\text{tr}(\bf C \it)\bf 1 - \bf C \it\right), \quad \bf C \it = \sum_i \bf C \it _i $$

<p></p>

Finally the tensor has to be moved to the center of mass using the parallel axis theorem (with $\vec{d} = \vec{x} _{COM}$ since $\bf I \it _{origin}$ is measured from the origin):

$$ \bf I \it _{COM} = \bf I \it _{origin} - M\left(\lVert \vec{x} _{COM}\rVert^2 \bf 1 - \vec{x} _{COM}\vec{x} _{COM}^T\right) $$

This $\bf I \it _{COM}$ is the body space tensor used by the impulse equations, and the mesh vertices should be translated by $-\vec{x} _{COM}$ so that the center of mass sits at the object's origin.

<p></p>

### Friction Impulse

<p></p>

After calculating the impulse due to collision, we also compute the friction impulse where we introduce the tangential velocity:

$$ \vec{v}_t = \vec{v} _{rel} - v_n\hat{n} $$

Where

$$ \lVert\vec{v}_t\rVert > 0 $$

thus,

$$ \hat{t} = \frac{\vec{v}_t}{\lVert\vec{v}_t\rVert} $$

otherwise, no friction is produced and $\hat{t} = \vec{0}$

<p></p>

The impulse needed to completely stop the sliding uses the same effective mass as $J_n$ but along $\hat{t}$:

$$ J _{stop} = \frac{\lVert\vec{v}_t\rVert}{m_A^{-1} + m_B^{-1} + (\vec{r}_A \times \hat{t}) \cdot\bf I^{-1}_A \it(\vec{r}_A \times \hat{t}) + (\vec{r}_B \times \hat{t}) \cdot \bf I^{-1}_B \it(\vec{r}_B \times \hat{t})} $$

Friction can never apply more than this (otherwise it would reverse the sliding direction and the object would jitter), and it can never exceed the Coulomb limit. So the friction impulse magnitude is:

If $J _{stop} \leq \mu_s |J_n|$ then friction is static (the contact stops sliding completely):

$$ J_f = J _{stop} $$

Otherwise friction is kinetic:

$$ J_f = \mu_k |J_n| $$

The friction impulse is applied against the tangential velocity: $\vec{J}_f = -J_f\hat{t}$.

<p></p>

The total impulse due to friction and the collision is: $\vec{J} = \vec{J}_f + J_n\hat{n}$

Using the total impulse, we can calculate the angular velocity for each object:

$$ \vec{\Delta L}_A = \vec{r}_A \times \vec{J}, \quad \vec{\Delta L}_B = \vec{r}_B \times \vec{J} $$

$$ \vec{\Delta \omega}_A = I^{-1}_A\vec{\Delta L}_A, \quad \vec{\Delta \omega}_B = I^{-1}_B\vec{\Delta L}_B $$

<p></p>

Then for each object, we can alter the velocity of each object by applying the total impulse: 

$$ \vec{v} _{A,i+1} = \vec{v} _{A,i} + \frac{\vec{J}}{m_A} $$

$$ \vec{v} _{B,i+1} = \vec{v} _{B,i} - \frac{\vec{J}}{m_B} $$

<p></p>

Similarly the angular velocities can be calculated:

$$ \vec{\omega} _{A,i+1} = \vec{\omega} _{A,i} + \vec{\Delta \omega}_A $$

$$ \vec{\omega} _{B,i+1} = \vec{\omega} _{B,i} - \vec{\Delta \omega}_B $$

($i$ infers the previous frame/step and $i+1$ infers the current frame/step).

<p></p>

### Torque

<p></p>

A force applied at a point $\vec{p}$ away from the center of mass produces a torque:

$$ \vec{\tau} = \vec{r} \times \vec{F} $$

and the angular acceleration (using the world space inertia tensor) is:

$$ \vec{\alpha} = I^{-1}\left(\vec{\tau} _{net} - \vec{\omega} \times (I\vec{\omega})\right), \quad \vec{\tau} _{net} = \sum_j \vec{r}_j \times \vec{F}_j + \vec{\tau} _{drag} $$

where the angular drag is $\vec{\tau} _{drag} = -\kappa_1\vec{\omega} - \kappa_2\lVert\vec{\omega}\rVert\vec{\omega}$ (from the damping part of the force section), so a spinning object loses angular velocity over time even when nothing touches it.

Then the angular velocity is integrated like linear velocity. With the linear angular drag applied implicitly for stability (so $\vec{\alpha}'$ excludes the $\kappa_1$ term):

$$ \vec{\omega} _{i+1} = \frac{\vec{\omega}_i + \vec{\alpha}'\Delta t}{1 + c_a\Delta t} $$

<p></p>

The orientation is stored as a unit quaternion $q$ and updated each step with:

$$ q _{i+1} = \frac{q_i + \frac{\Delta t}{2}\,(0, \vec{\omega})\, q_i}{\lVert q_i + \frac{\Delta t}{2}\,(0, \vec{\omega})\, q_i \rVert} $$

The rotation matrix $R$ from $q$ is what gives $\bf I \it ^{-1} _{world}$ above.

<p></p>

### Position Correction

<p></p>

Impulses only fix velocities, so objects can slowly sink into each other. After resolving the impulse, the penetration depth $d$ from EPA is used to push the objects apart along the normal:

$$ \vec{x} _{A,i+1} = \vec{x} _{A,i} + \frac{m_A^{-1}}{m_A^{-1} + m_B^{-1}} \beta d \hat{n} $$
 
$$ \vec{x} _{B,i+1} = \vec{x} _{B,i} - \frac{m_B^{-1}}{m_A^{-1} + m_B^{-1}} \beta d \hat{n} $$

where $\beta \approx 0.2$ to $0.8$ is how much of the penetration is corrected each step.

<p></p>

(contact manifolds and iterative solving to come later)
