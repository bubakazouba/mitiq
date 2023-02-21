---
jupytext:
  text_representation:
    extension: .myst
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.11.1
kernelspec:
  display_name: Python 3
  language: python
  name: python3
---

# What happens when I use DDD?

The workflow of Digital Dynamical Decoupling (DDD) in Mitiq is represented in the figure below.

```{figure} ../img/ddd_workflow.svg
---
width: 700px
name: ddd-workflow
---
Workflow of the DDD technique in Mitiq.
```

- The user provides a `QPROGRAM`, (i.e. a quantum circuit defined via any of the supported [frontends](frontends-backends.md)).
- Mitiq modifies the input circuit with the insertion of DDD gate sequences in idle windows.
- The modified circuit is executed via a user-defined [Executor](executors.md).
- The error mitigated expectation value is returned to the user.

With respect to the workflows of other error-mitigation techniques (e.g. [ZNE](zne-4-low-level.md) or [PEC](pec-4-low-level.md)),
DDD involves the generation and the execution of a _single_ modified circuit.
For this reason, there is no need to combine the results of multiple circuits and the final inference step which is necessary for other 
techniques is instead trivial for DDD.

```{note}
When setting the `num_trials` option to a value larger than one, multiple circuits are actually generated by Mitiq and 
the associated results are averaged to obtain the final expectation value. This more general case is not shown in the figure since
it can be considered as an average of independent single-circuit workflows.
```

As shown in [How do I use DDD?](ddd-1-intro.md), the function {func}`.execute_with_ddd()` applies DDD behind the scenes 
and directly returns the error-mitigated expectation value.
In the next sections instead, we show how one can apply DDD at a lower level, i.e., by:

- Characterizing all the slack windows in a circuit;
- Inserting DDD sequences in all slack windows;
- Executing the modified circuit.

## Analysis of idle windows (optional step)

In this section we show how one can determine all the idle windows (often called slack windows) in a circuit and how long they are,
i.e. how many single-qubit gates can fit each window. 
This is an optional step, since it is actually not necessary to apply DDD with Mitiq. 
Nonetheless, it provides additional information on the gate structure of the circuit that can be useful, especially for research purposes. 

### The circuit mask
A quantum circuit can be visualized as a 2D grid where the horizontal axis represents discrete
time steps (often called moments) and the vertical axis represents the qubits of the circuit. Each gate occupies one or more grid cells,
depending on the number of qubits it acts on. 

This 2D grid is essentially what we get each time we print a circuit out.

```{code-cell} ipython3
from cirq import Circuit, X, SWAP, LineQubit

qreg = LineQubit.range(8)
x_layer = Circuit(X.on_each(qreg))
cnots_layer = Circuit(SWAP.on(q, q + 1) for q in qreg[:-1])
circuit = x_layer + cnots_layer + x_layer
circuit
```

The grid structure of a circuit can be expressed as a _mask matrix_ with $1$ entries in cells that
are occupied by gates and $0$ entries in empty cells. So, for example, the mask matrix of the above circuit is the following:

```{code-cell} ipython3
from mitiq import ddd

mask_matrix = ddd.insertion._get_circuit_mask(circuit)
mask_matrix
```

A slack window is an horizontal and contiguous sequence of zeros in the mask matrix, corresponding to a qubit which is
idling for a finite amount of time.

### The slack matrix

To analyze the structure of idle windows, it is more convenient to define a _slack matrix_, i.e.,
a matrix of positive integers that are placed at the beginning of each slack window and whose value represent the
time length of that window. For example, in our simple example, we get:

```{code-cell} ipython3
slack_matrix = ddd.get_slack_matrix_from_circuit_mask(mask_matrix)
slack_matrix
```

This matrix contains the same amount of information as `mask_matrix`, but it is more convenient for the analysis
of idle windows as shown in the next code cell.

```{code-cell} ipython3
import numpy as np 

print(f"The circuit contains {np.count_nonzero(slack_matrix)} slack windows.")
print(f"The maximum slack length is {np.max(slack_matrix)}.")
lengths, counts = np.unique(slack_matrix, return_counts=True)
length_distribution = dict(zip(lengths[1:], counts[1:]))
print(f"The full distribution of slack lengths is {length_distribution}.")
print(f"On average, each qubit spends {np.mean(slack_matrix) :.0%} of time in idle mode.")
```

## Inserting DDD sequences

+++

The DDD error mitigation technique consists of filling the slack windows of a circuit with DDD gate sequences.
This can be directly achieved via the function {func}`.insert_ddd_sequences()` function.

```{code-cell} ipython3
xyxy_rule = ddd.rules.xyxy
circuit_with_ddd = ddd.insert_ddd_sequences(circuit, rule=xyxy_rule)
circuit_with_ddd
```

```{note}
In principle, the function {func}`.insert_ddd_sequences()` is all one needs to apply DDD.
Indeed, since in DDD there is not a final post-processing step, one can simply insert DDD sequences before running the circuit on a noisy backend.
As shown in the next section, this approach is exactly equivalent to the application of the standard Mitiq function {func}`.execute_with_ddd()`. 
```

+++

## Executing the modified circuit

```{code-cell} ipython3
from cirq import DensityMatrixSimulator, amplitude_damp

def execute(circuit, noise_level=0.003):
    """Returns Tr[ρ |00..⟩⟨00..|] where ρ is the state prepared by the circuit
    executed with depolarizing noise.
    """
    noisy_circuit = circuit.with_noise(amplitude_damp(noise_level))
    rho = DensityMatrixSimulator().simulate(noisy_circuit).final_density_matrix
    return rho[0, 0].real
```

If executed on a noiseless backend, `circuit_with_ddd` and `circuit` are equivalent.
On a real backend, they have a different sensitivity to noise. The core idea of the DDD technique is that,
`circuit_with_ddd` is (hopefully) less sensitive to noise thanks to the particular structure of DDD sequences.

```{code-cell} ipython3
# Ideal result
execute(circuit, noise_level=0)
```

```{code-cell} ipython3
# Unmitigated result
execute(circuit)
```

```{code-cell} ipython3
# Error-mitigated result (with DDD)
execute(circuit_with_ddd)
```

As a final remark, we stress that the low-level procedure that we have shown is exactly what {func}`.execute_with_ddd()` does behind the scenes.
Let's verify this fact: 

```{code-cell} ipython3
np.isclose(
  ddd.execute_with_ddd(circuit, execute, rule=xyxy_rule),
  execute(circuit_with_ddd),
)
```