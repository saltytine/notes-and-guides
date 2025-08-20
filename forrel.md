ds^2 = -alpha^2 dt^2 + gamma_ij (dx^i + beta^i dt)(dx^j + beta^j dt)

evolve:
∂_t gamma_ij = -2 alpha K_ij + L_beta gamma_ij
∂_t K_ij = -D_i D_j alpha + alpha (R_ij + K K_ij - 2 K_ik K^k_j) + L_beta K_ij

constraints:
H = R + K^2 - K_ij K^{ij} ≈ 0
M^i = D_j (K^{ij} - gamma^{ij} K) ≈ 0

BSSN vars:
bar{gamma}_ij = e^{-4 phi} gamma_ij
phi = (1/12) ln det(gamma_ij)
bar{A}_ij = e^{-4 phi} (K_ij - (1/3) gamma_ij K)
K = gamma^{ij} K_ij
bar{Gamma}^i = - ∂_j bar{gamma}^{ij}

central difference for derivatives:
∂f/∂x ≈ (f_{i+1} - f_{i-1}) / (2 Δx)
second derivative:
∂²f/∂x² ≈ (f_{i+1} - 2 f_i + f_{i-1}) / (Δx)^2

for boundary calculations:
periodic: copy values from opposite edges
outflow: copy interior to edge
dirichlet: set edge to fixed value
neumann: set edge to interior + dx * derivative

lapse:
∂_t alpha = -2 alpha K + beta^i ∂_i alpha

gamma driver:
∂_t beta^i = (3/4) B^i
∂_t B^i = ∂_t Gamma^i - eta B^i

TODO
clamp phi to avoid metric singularities, like phi in [-20, 20]
to make sure bar{A}_ij is trace-free: subtract tr/3 from diagonal
check for NaNs in constraints - NOTE: i dont really need this anymore, cause we are NaN free
print max/avg constraint violations after each step
add explanations to output for non experts (aka, retards)
make boundary conditions runtime configurable with some flag

something that looks like:
“hamiltonian constraint: max=1e-5, avg=2e-6”
“momentum constraint: max=3e-6, avg=1e-6”

ricci
R_ij = ∂_k Gamma^k_ij - ∂_j Gamma^k_ik + Gamma^k_kl Gamma^l_ij - Gamma^k_jl Gamma^l_ik

invert 3x3 matrix for raising indices
and use central differences for all spatial derivatives

for scalar field matter:
T_{mu nu} = ∂_mu phi ∂*nu phi - (1/2) g*{mu nu} (∂^alpha phi ∂_alpha phi + V(phi))

ADD SOURCE TERMS TO EVOLUTION EQUATIONS

i gotta implement some way to enforce symmetry and positive-definiteness of metric
also, i want to use saturating arithmetic for indices at boundaries, but ill implement that later
add “verbose” mode for extra explanation (prolly the most important)
