# Compressed sensing reconstruction with a subspace constrained 

## Background

The main idea is to solve the following problem :

```math
\min_\alpha \frac{1}{2}||y - E \Phi_K \alpha||^2_2 + \lambda||W(\alpha)||_1
```

where **y** is the k-space data, **E** is the encoding operator regrouping the FFT operator and
the coil sensitvity operator. **$\Phi_K$** is the subspace cropped to the
first K temporal dimensions, **$\alpha$** is the subspace coefficient maps and **W** corresponds to the Wavelet Operator which is applied on **$\alpha$**.

## Julia implementation

The subspace reconstruction was implemented as part of [MRIReco.jl](https://github.com/MagneticResonanceImaging/MRIReco.jl) package. An example in the MRIReco.jl package can be found [here](https://magneticresonanceimaging.github.io/MRIReco.jl/latest/generated/examples/03-subspaceReconstruction/).

The reconstruction works on an `AcquisitionData` object (previously converted from the Bruker raw dataset). Because it is a combined parallel imaging and compressed sensing approach, we have to estimate the coil sensitivity map.

```julia
using Subspace_MESE
using Subspace_MESE.MRIFiles
using Subspace_MESE.MRIReco
using Subspace_MESE.MRICoilSensitivities

b=BrukerFile("path/to/dataset/")
raw = RawAcquisitionData_MESE(b)
acq = AcquisitionData(raw,OffsetBruker = true);

coilsens = espirit(acq,eigThresh_2=0.0);
basis,dico = basis_calibration(6,acq,(15,15,15))
```

From that point, we need to define a Dict structure (here named `params`) to store and pass the MRIReco reconstruction parameters.

```julia
params = Dict{Symbol,Any}()
params[:reconSize] = acq.encodingSize

# Subspace 
params[:reco] = "multiCoilMultiEchoSubspace"
params[:senseMaps] = coilsens
params[:basis] = basis

# Compressed sensing transform
params[:regularization] = "L1"
params[:sparseTrafo] = "Wavelet" #sparse trafo
params[:λ] = Float32(0.01)

# Algorithm parameters
params[:solver] = "fista"
params[:iterations] = 30
params[:normalizeReg] = true
```

`params[:reco] = "multiCoilMultiEchoSubspace"` means that it is a subspace reconstruction with parallel imaging and needs the fields `params[:senseMaps]` and `params[:basis]`

The remaining parameters are linked to the algorithm
used, here a wavelet transform + L1 regularization with a FISTA algorithm and 30 iterations.

To perform the reconstruction we can now call the function :

```julia
alpha = reconstruction(acq, params);
```

The results of the reconstruction is not directly the echo images but the subspace coefficient maps. In order to get the virtual echo images, the subspace needs to be applied on this coefficient maps.

```julia
im_TE = abs.(applySubspace(α, basis))
```

!!! warning     
    The reconstruction part might evolve due to changes in the **MRIReco** package and its dependencies, particularly **RegularizedLeastSquares.jl**

## Comparison with BART

All the figures (except some parts of figure 8) from the publication were generated using a reconstruction from BART. For comparison purpose, the reconstruction is also available in this package and detailed below.

You need to compile the BART toolbox : https://mrirecon.github.io/bart/
and pass as an argument the path to the executable BART library in the `subspace_bart_reconstruction` function.

This package uses the wrapper `BartIO.jl` to send the data back and forth between JULIA and BART.

```julia
using Subspace_MESE.BartIO

params = Dict{Symbol,Any}()
params[:iterations] = 60
params[:λ] = 0.01
params[:basis]=basis

im_sub_bart,im_TE_bart = subspace_bart_reconstruction(acq,params,"/home/CODE/bart/bart")
```

If you need more control over the reconstruction parameters for BART, take a look at the function (only 20 lines)

!!! note     
    The regularization parameters between MRIReco.jl and BART are different and need to be adjusted manually. This can be explained by some low-level differences in the FISTA algorithm / Wavelet transform and also due to scaling of the data prior to the algorithm.
