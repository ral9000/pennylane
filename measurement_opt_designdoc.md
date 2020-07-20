# Pennylane measurement optimization utility design document

### **Overview:**

In this design document we discuss possible implementations of the qubit-wise commuting (QWC) and full-commuting measurement optimization schemes in PennyLane. The possibility of a more general implementation (not restricted to VQE) is explored, since the measurement optimization schemes can be utilized in any algorithm/experiment where many observables are measured with respect to a prepared state (for which VQE calculations are a proper subset).


### **Option 1: Contained within [VQECost](https://pennylane.readthedocs.io/en/stable/code/api/pennylane.VQECost.html) **

The QWC and fully-commuting measurement optimization procedures can be implemented directly within the __init__ and __call__ methods of the VQECost class. This seems to be the most self-contained and the most restrictive direction. The user would not interface directly with any of the measurement optimization utilities, only specify their application through optional keywords when initializing a VQECost instance. Note that, for convenience, self.cost_fn in the modified VQECost is now a sum over a list of separate cost functions, each initiated by qml.dot. In the case where optimize_measurements=None, self.cost_fn is a list of a single element, the normal cost function initialized in the standard manner. If e.g. optimize_measurements=‘qwc’, then self.cost_fn is a list of length M where M is the number of simultaneously measurable grouping. This is done so all simultaneously measurable terms are contained within their own QNodeCollection, which with the appropriate changes to QNodeCollection class, should enable their simultaneous measurement. 
 
`class VQECost:`
`def __init __(self, ansatz, hamiltonian, device, interface, diff_method, optimize_measurements=None, **kwargs ):
`
`coeffs, observables = hamiltonian.terms `
`self.hamiltonian = hamiltonian `

`if optimize_measurements == ‘qwc’:`
`
Perform MCC to find QWC groupings
Obtain the appropriate post-rotations for each grouping, obtain each QWC grouping in diagonal form
Define the post-rotated circuit ansatzes for each grouping 
For each grouping, obtain QNodeCollection via qml.map using the post-rotated ansatz circuit and the diagonal QWC grouping
Append each QNodeCollection dotted with its respective coefficient list to self.cost_fn `

`if optimize_measurements == ‘commuting’:`
`
Perform MCC to find fully commuting groupings
Obtain the coefficient lists for each grouping
Obtain the appropriate post-rotations for each grouping, obtain each fully commuting grouping in diagonal form
Define the post-rotated circuit ansatzes for each grouping 
For each grouping, obtain QNodeCollection via qml.map using the post-rotated ansatz circuit and the diagonal fully commuting grouping
Append each QNodeCollection dotted with its respective coefficient list to self.cost_fn `
`
else:`
`Perform normal VQECost initialization `


`def __call__(self, *args, **kwargs):`
`
return Sum over evaluated functions in self.cost_fn `

Note that VQECost has attribute metric_tensor, which evaluates the value of the metric tensor. This method would need slight modification to avoid breaking it when using measurement optimization.

### Option 2: Standalone function with output as transformed groupings and post-rotation circuit templates 

The measurement optimization schemes may be applied to any experiment where many Pauli observables must be measured with respect to a prepared state. The output of the measurement optimization (in the QWC and fully-commuting approaches) is the observable groupings, along with the corresponding necessary circuit post-rotations, and the only  problem-specific input is the list of Pauli observables to be measured. This motivates a standalone function, referred to here as optimize_measurements, which can in principle be used for optimizing the number of measurements for any experiment of this form, including VQE calculations. The output list of observable groupings would be a nested list of observables, where all observables are diagonal in the computational basis. The output list of post-rotations would be the corresponding circuit templates that must be appended to the circuit such that the expectation values are unchanged when evaluating the expectation values of the transformed groupings. 

`def optimize_measurements(observables, grouping=‘qwc’, mcc_method=‘rlf’):`

`if grouping == ‘qwc’:`
`
Perform MCC to find fully commuting groupings
Obtain the coefficient lists for each grouping if coefficients are given
Obtain the appropriate post-rotations for each grouping, obtain each fully commuting grouping in diagonal form

`
`if grouping == ‘commuting’:`
`
Perform MCC to find fully commuting groupings
Obtain the coefficient lists for each grouping if coefficients are given
Obtain the appropriate post-rotations for each grouping, obtain each fully commuting grouping in diagonal form

`
`return nested list of diagonalized observables (outer list defines a grouping), and the corresponding list of post-rotations (post-rotations are in the form qml.template)`

This approach allows for flexible usage of the QWC and fully-commuting measurement optimization schemes. The returned diagonal observable groupings and post-rotations can be used within VQECost to define the simultaneously measurable QNodeCollections produced within the measurement optimization blocks in VQECost of Option 1. It also allows for application of the measurement optimization scheme outside the context of variational algorithms, e.g. state tomography. Keeping the output as a list of the grouped observables and a list of post-rotation qml.templates seems to be adequately general, it allows straightforward application internally within VQECost, while also being a useful utility function employable by the user for applications outside of VQE.

### **Option 3: Standalone function with output as a list of [QNodeCollections](https://pennylane.readthedocs.io/en/stable/code/api/pennylane.QNodeCollection.html) **

Alternatively to Option 2, optimize_measurements could output QNodeCollections initiated using the post-rotated circuit ansatze and the diagonalized observable groupings. In this case, the input to optimize_measurements would be all specifications needed to construct the final QNodeCollections via qml.map, i.e. the template (circuit ansatz), list of observables, device, diff_method, etc. The output to optimize_measurements would be a list of QNodeCollections. The number of QNodeCollections equals the number of simultaneously measurable groups, and each QNodeCollection possesses QNodes with strictly diagonal Pauli terms in the computational basis.  
 
If the set of observables to be measured have weights associated to them, such as when obtaining the expectation value of an arbitrary Hermitian observable (linear combination of Pauli terms with real weights), the list of weights can be specified by using the optional coeffs keyword argument in optimize_measurements. Typically, correspondence between a Pauli term and its weight is made through index in their respective lists, i.e. coeffs[i] is the weight of Pauli term observables[i]. When observables are re-organized into simultaneously measurable groupings, the canonical order is lost, and hence optimize_measurements will need to restructure coeffs in a similar manner to the observables. With weights being an optional input, optimize_measurements is suitable for use within VQECost, while also being applicable as a separate utility altogether for use in other variational algorithms.
 
`def optimize_measurements(ansatz, device, observables, coefficients=None, interface=‘autograd’, diff_method=‘best’, grouping=‘qwc’, mcc_method=‘rlf’, **kwargs):`

`if grouping == ‘qwc’:`
`
Perform MCC to find fully commuting groupings
Obtain the coefficient lists for each grouping if coefficients are given
Obtain the appropriate post-rotations for each grouping, obtain each fully commuting grouping in diagonal form
Define the post-rotated circuit ansatzes for each grouping 
For each grouping, obtain QNodeCollection via qml.map using the post-rotated ansatz circuit and the diagonal fully commuting grouping
Append each QNodeCollection to output list`
`
if grouping == ‘commuting’:`
`
Perform MCC to find fully commuting groupings
Obtain the coefficient lists for each grouping if coefficients are given
Obtain the appropriate post-rotations for each grouping, obtain each fully commuting grouping in diagonal form
Define the post-rotated circuit ansatzes for each grouping 
For each grouping, obtain QNodeCollection via qml.map using the post-rotated ansatz circuit and the diagonal fully commuting grouping
Append each QNodeCollection to output list`

`return output list of QNodeCollections (one QNodeCollection for each simultaneously measurable grouping with all observables diagonalized)
(If coefficients are given, also return the grouped coefficient lists corresponding to each QNodeCollection.)`

With standalone utility function optimize_measurements, it can be simply called within VQECost to implement Option 1. Then VQECost with measurement optimization implement would look like:

`class VQECost:`
`def __init __(self, ansatz, hamiltonian, device, interface, diff_method, optimize_measurements=None, **kwargs ):
`
`coeffs, observables = hamiltonian.terms `
`self.hamiltonian = hamiltonian `

`if optimize_measurements == ‘qwc’:`
`
simultaneously_measurable_qnodecollections, grouped_coefficients = optimize_measurements(ansatz, device, observables, coefficients = coeffs, interface=‘autograd’, diff_method=‘best’, grouping=‘qwc’, **kwargs)

Append each QNodeCollection in simultaneously_measurable_qnodecollections dotted with its respective coefficient list in group_coefficients to self.cost_fn
`

`if optimize_measurements == ‘commuting’:`
`
simultaneously_measurable_qnodecollections, grouped_coefficients = optimize_measurements(ansatz, device, observables, coefficients = coeffs, interface=‘autograd’, diff_method=‘best’, grouping=‘commuting’, **kwargs)

Append each QNodeCollection in simultaneously_measurable_qnodecollections dotted with its respective coefficient list in group_coefficients to self.cost_fn
`
`
else:`
`Perform normal VQECost initialization `


`def __call__(self, *args, **kwargs):`
`
return Sum over evaluated functions in self.cost_fn `

Further it can be called independently outside of a VQECost initialization. However, since qml.map requires specifications relating to optimization (interface, diff_method), this implementation seems limited to application in variational optimization algorithms.


### Option 4: Standalone function with alternative output structure

 This option is a point of discussion for what output data structure make the most sense for optimize_measurements. Currently, Option 2 uses a list of QNodeCollections as the output format, where each QNodeCollection corresponds to a simultaneously measurable grouping that is diagonal in the computational basis. The design philosophy behind this is that QNodeCollections can/will be modified such that when all QNodes within the QNodeCollections have strictly diagonal observables in computational basis, one does not need to initiate a new circuit preparation for each measurement. Rather, one just needs to take the union of the wires being acted on non-trivially by all observables in the QNodeCollection, and measure each non-trivial wire. This enables a single sampling of the expectation value for this grouping with a single circuit execution.

If there is a more natural way to perform the simultaneous measurements, this should play a role in determining the output format of optimize_measurements. For instance, can it be a single QNodeCollection rather than a list of individual QNodeCollections? This will depend on how simultaneously measurable operators are flagged.

### Option 5: Implementation within [qml.map](https://pennylane.readthedocs.io/en/stable/code/api/pennylane.map.html)  

Can optimize_measurements live in qml.map? As mentioned, one can perform a measurement optimization for any setting where there is a ansatz/state which is to be measured with respect to a set of Pauli observables. This coincides with the acceptable input to a qml.map calling, hence a possibly natural place for the optimize_measurement utility is within qml.map internally. Note that the current implementation of optimize_measurement uses qml.map for each grouping, hence this would need to be replaced with a function build out of primitive functions if optimize_measurements is to be called within qml.map itself. Implementing optimize_measurements within qml.map also has implications for the ideal output data structure of optimize_measurements, as the output data structure of qml.map should remain consistent whether measurements are optimized or not. Currently the output data structure of qml.map is a single QNodeCollection (as opposed to a list of QNodeCollections).

