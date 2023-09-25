---
author: SoniaLopezBravo
description: In this tutorial, you implement two quantum adders and then inspect them using the profiling feature in the Azure Quantum Resource Estimator.
ms.author: sonialopez
ms.date: 08/20/2023
ms.service: azure-quantum
ms.subservice: computing
ms.topic: tutorial
no-loc: [target, targets]
title: 'Tutorial: Profiling of a quantum adder'
uid: microsoft.quantum.tutorial.resource-estimator.profiling
---

# Tutorial: Estimate the resources of a quantum adder using the profiling feature

In this tutorial, you will implement two quantum adders and then inspect them using the [profiling feature](xref:microsoft.quantum.work-with-resource-estimator#use-profiling-to-analyze-the-structure-of-your-program) in the [Azure Quantum Resource Estimator](xref:microsoft.quantum.overview.intro-resource-estimator). The profiling feature allows you to analyze how subroutine operations in the quantum algorithm impact the overall resources.

In this tutorial, you'll learn how to:

> [!div class="checklist"]
> * Connect to the Azure Quantum service.
> * Implement two quantum adders.
> * Inspect the quantum adders using the profiling feature in the Azure Quantum Resource Estimator.
> * Analyze the profiling results and detect the impact of subroutine operations.

## Prerequisites

* An Azure account with an active subscription. If you don’t have an Azure account, register for free and sign up for a [pay-as-you-go subscription](https://azure.microsoft.com/pricing/purchase-options/pay-as-you-go/).
* An Azure Quantum workspace. For more information, see [Create an Azure Quantum workspace](xref:microsoft.quantum.how-to.workspace).
* The **Microsoft Quantum Computing** provider added to your workspace. For more information, see [Enabling the Resource Estimator target](xref:microsoft.quantum.quickstarts.computing.resources-estimator#enable-the-resources-estimator-in-your-workspace).

## Get started

Using the Resource Estimator is the same as submitting a job against other software and hardware provider targets in Azure Quantum - define your program, set a target, and submit your job for computation.

### Load the required imports

1. First, you need to import some Python classes and functions from `azure.quantum`. Click **+ Code** to add a new cell, then add and run the following code:

    ```python
    from azure.quantum import Workspace
    from azure.quantum.target.microsoft import MicrosoftEstimator
    import pandas
    import qsharp
    import re
    ```

### Connect to the Azure Quantum service

To connect to the Azure Quantum service, your program need the resource ID and the location of your workspace, which you can obtain from the **Overview** page in your Azure Quantum workspace in the [Azure portal](https://portal.azure.com).

:::image type="content" source="media/azure-quantum-resource-id.png" alt-text="Screenshot showing how to retrieve the resource ID and location from an Azure Quantum workspace":::

1. Paste the values into the following `Workspace` constructor to create a `workspace` object that connects to your Azure Quantum workspace.

    ```python
    workspace = Workspace(
      resource_id="",
      location=""
    )
    ```

2. This workspace is then used to create an instance to the Resource Estimator.

    ```python
    estimator = MicrosoftEstimator(workspace)
    ```

## Implement the quantum adders

In quantum computing, the ability to perform addition of qubit registers is crucial for a wide range of applications. Given two qubit $n$-bit registers $\ket{x} = \ket{x_{n-1}\dots x_1 x_0}$ and $\ket{y} = \ket{y_{n-1}\dots y_1 y_0}$, the goal is compute an $n+1$ bit register $\ket{z}=\ket{x+y}$.

In this tutorial, you implement two different quantum adders that can be used to perform addition in quantum computing: the *ripple-carry adder* and the *carry-lookahead adder*. The quantum adders are implemented as Q# operations `RippleCarryAdd` and `LookAheadAdd` together with estimation entry points `EstimateRippleCarryAdd` and `EstimateLookAheadAdd`, respectively. Each entry point can be provided a bit width.

Both ripple-carry and carry-lookahead adders support the overflowing variant, in which the output register has $n$ bits and $\ket{z}= \ket{(x+y) \mod 2^n}$.

### Ripple-carry adder

For the ripple-carry adder, you use the _out-of-place_ variant, in which the sum of input registers $\ket{x}$ and $\ket{y}$ is computed into a $\ket{0}$-initialized output register. A ripple-carry adder adds the bits of $\ket{x}$ and $\ket{y}$ one by one, starting from the rightmost bit, and propagates the carry generated by the addition to the next bit to the left. This process is repeated until all the bits have been added, resulting in a final sum and carry-out.

The core component of the ripple-carry adder is the **full adder**, which is a one-bit adder that takes a carry input bit, and returns a carry output bit, in addition to the sum bit.

For more information, see the ripple-carry adder described in [Halving the cost of quantum addition](https://arxiv.org/abs/1709.06648).

1. First, you create an internal operation `FullAdder`. The full adder is implemented using `CNOT` gates and the `ApplyAnd` operation, which applies an AND gate to two qubits and stores the result in a third qubit. Note that the `FullAdder` operation is marked as `Adj` to indicate that it's its own adjoint. This is necessary for the ripple-carry adder to be reversible.

    ```python
    %%qsharp
    internal operation FullAdder(carryIn : Qubit, x : Qubit, y : Qubit, carryOut : Qubit) : Unit is Adj {
        CNOT(x, y);
        CNOT(x, carryIn);
        ApplyAnd(y, carryIn, carryOut);
        CNOT(x, y);
        CNOT(x, carryOut);
        CNOT(y, carryIn);
    }
    ```

2. Next, with the `FullAdder` operation, implementing the ripple-carry adder is as simple as carrying the carry bit through successive full adder invocations. The two `CNOT` operations at the end handle the case in which $\ket{z}$ has $n$ bits, in which the sum for the most significant bit is not computed by a full adder (since the corresponding carry out bit doesn't exist).

    ```python
    %%qsharp
    open Microsoft.Quantum.Arrays;
    open Microsoft.Quantum.Diagnostics;
    
    operation RippleCarryAdd(xs : Qubit[], ys : Qubit[], zs : Qubit[]) : Unit is Adj {
        let n = Length(xs);
        let m = Length(zs);
    
        Fact(Length(ys) == n, "Registers xs and ys must be of same length");
        Fact(m == n or m == n + 1, "Register zs must be same length as xs or one bit larger");
    
        for k in 0..m - 2 {
            FullAdder(zs[k], xs[k], ys[k], zs[k + 1]);
        }
    
        if n > 0 and n == Length(zs) {
            CNOT(Tail(xs), Tail(zs));
            CNOT(Tail(ys), Tail(zs));
        }
    }
    ```

3. Finally, you create an entry point for resource estimation. The `EstimateRippleCarryAdd` operation calls the ripple-carry adder for a given bit width, which can be provided as an input argument.

    ```python
    %%qsharp
    operation EstimateRippleCarryAdd(bitwidth : Int) : Unit {
        use xs = Qubit[bitwidth];
        use ys = Qubit[bitwidth];
        use zs = Qubit[bitwidth + 1];
    
        RippleCarryAdd(xs, ys, zs);
    }
    ```

> [!NOTE]
> The sample of this tutorial considers an output register of $n+1$ bits.

### Carry-lookahead adder

The idea of carry-lookahead adder is to compute all carry bits $c_i$ based on propagate bits $p_i = x_i \oplus y_i$ and generate bits $g_i= x_i \land y_i$, without requiring other carry bits except for the carry-in $c_0$.

For example, the first carry bit $c_1$ can be computed as $c_1 =g_0 \oplus (p_0 \land c_0)$. 

* If both $x_0$ and $y_0$ are 1, then $g_0 =1$, and therefore $c_1 =1$.
* If either $x_0$ or $y_0$ is 1, then $p_0 =1$, and therefore carry bit $c_0$ is propagated. 

More significant carry bits are computed in a similar way, for example $c_3 =g_2 \oplus (g_1 \land p_2) \oplus (g_0 \land p_1 \land p_2) \oplus (c_0 \land p_0 \land p_1 \land p_2)$. That is, $c_3$ is either generated from bits at index 2, or generated from bits at index 1 _and_ propagated from bits at index 2, and so on.

For more information, see the carry-lookahead adder described in [A logarithmic-depth quantum carry-lookahead adder](https://arxiv.org/abs/quant-ph/0406142).

In order to minimize the number of`AND` gates, these intermediate products can be computed in a clever way, as well as in logarithmic depth.

1. First, you create a helper function to compute the number of 1-bits in an integer, also called _Hamming weight_, using a compact implementation based on a sequence of bitwise manipulations. For more information, see [Hamming weight](https://en.wikipedia.org/wiki/Hamming_weight#Efficient_implementation).

    ```python
    %%qsharp
    function HammingWeight(n : Int) : Int {
        mutable i = n - ((n >>> 1) &&& 0x5555555555555555);
        set i = (i &&& 0x3333333333333333) + ((i >>> 2) &&& 0x3333333333333333);
        return (((i + (i >>> 4)) &&& 0xF0F0F0F0F0F0F0F) * 0x101010101010101) >>> 56;
    }
    ```

2. Next, you create internal routines that compute the generalized propagate bits, `PRounds`,  generalized generate, `GRounds`, and the carry bits from them, `CRounds`. For more information, see the description of these implementations in [A logarithmic-depth quantum carry-lookahead adder](https://arxiv.org/abs/quant-ph/0406142).

    ```python
    %%qsharp
    open Microsoft.Quantum.Arrays;
    open Microsoft.Quantum.Convert;
    open Microsoft.Quantum.Math;
    
    internal operation PRounds(pWorkspace : Qubit[][]) : Unit is Adj {
        for ws in Windows(2, pWorkspace) {
            let (current, next) = (Rest(ws[0]), ws[1]);
    
            for (m, target) in Enumerated(next) {
                ApplyAnd(current[2 * m], current[2 * m + 1], target);
            }
        }
    }
    
    internal operation GRounds(pWorkspace : Qubit[][], gs : Qubit[]) : Unit is Adj {
        let numRounds = Length(pWorkspace);
        let n = Length(gs);
    
        for t in 1..numRounds {
            let length = Floor(IntAsDouble(n) / IntAsDouble(2^t)) - 1;
            let ps = pWorkspace[t - 1][0..2...];
    
            for m in 0..length {
                CCNOT(gs[2^t * m + 2^(t - 1) - 1], ps[m], gs[2^t * m + 2^t - 1]);
            }
        }
    }
    
    internal operation CRounds(pWorkspace : Qubit[][], gs : Qubit[]) : Unit is Adj {
        let n = Length(gs);
    
        let start = Floor(Lg(IntAsDouble(2 * n) / 3.0));
        for t in start..-1..1 {
            let length = Floor(IntAsDouble(n - 2^(t - 1)) / IntAsDouble(2^t));
            let ps = pWorkspace[t - 1][1..2...];
    
            for m in 1..length {
                CCNOT(gs[2^t * m - 1], ps[m - 1], gs[2^t * m + 2^(t - 1) - 1]);
            }
        }
    }
    ```

3. With the subroutine operations, you can build an operation that computes the carry bits from the initial propagate and generate bits. Note that the generalized propagate bits are computed out-of-place into some helper qubits `qs`, whereas the generalized generate and carry bits are computed in-place into the register `gs`, which contains the initial generate bits.

    ```python
    %%qsharp
    internal operation ComputeCarries(ps : Qubit[], gs : Qubit[]) : Unit is Adj {
        let n = Length(gs);
    
        let numRounds = Floor(Lg(IntAsDouble(n)));
        use qs = Qubit[n - HammingWeight(n) - numRounds];
    
        let registerPartition = MappedOverRange(t -> Floor(IntAsDouble(n) / IntAsDouble(2^t)) - 1, 1..numRounds - 1);
        let pWorkspace = [ps] + Partitioned(registerPartition, qs);
    
        within {
            PRounds(pWorkspace);
        } apply {
            GRounds(pWorkspace, gs);
            CRounds(pWorkspace, gs);
        }
    }
    ```

4. Now, you can implement the carry-lookahead adder. First, computing the initial propagate and generate bits, then computing the carry bits, and finally computing the output bits using the sums (initial propagate bits) together with the carry bits. Note that this implementation supports both an variant where the output register is 1 bit larger and does not overflow, as well as a variant in which the sum is computed modulo $2^n$. The latter uses the former by using a special treatment of the most-significant bits of the input registers.

    ```python
    %%qsharp
    open Microsoft.Quantum.Arrays;
    open Microsoft.Quantum.Canon;
    open Microsoft.Quantum.Diagnostics;
    
    operation LookAheadAdd(xs : Qubit[], ys : Qubit[], zs : Qubit[]) : Unit is Adj {
        let n = Length(xs);
        let m = Length(zs);
    
        Fact(Length(ys) == n, "Registers xs and ys must be of same length");
        Fact(m == n or m == n + 1, "Register zs must be same length as xs or one bit larger");
    
        if m == n + 1 { // with carry-out
            // compute initial generate values
            for k in 0..n - 1 {
                ApplyAnd(xs[k], ys[k], zs[k + 1]);
            }
    
            within {
                // compute initial propagate values
                ApplyToEachA(CNOT, Zipped(xs, ys));
            } apply {
                if n > 1 {
                    ComputeCarries(Rest(ys), Rest(zs));
                }
    
                // compute sum into carries
                for k in 0..n - 1 {
                    CNOT(ys[k], zs[k]);
                }
            }
        } else { // without carry-out
            LookAheadAdd(Most(xs), Most(ys), zs);
            CNOT(Tail(xs), Tail(zs));
            CNOT(Tail(ys), Tail(zs));
        }
    }
    ```

5. Finally, you create an entry point for resource estimation. The `EstimateLookAheadAdd` operation calls the carry-lookahead adder for a given bit width, which can be provided as an input argument.

    ```python
    %%qsharp
    operation EstimateLookAheadAdd(bitwidth : Int) : Unit {
        use xs = Qubit[bitwidth];
        use ys = Qubit[bitwidth];
        use zs = Qubit[bitwidth + 1];
    
        LookAheadAdd(xs, ys, zs);
    }
    ```

## Profiling with the Resource Estimator

The profiling feature in [Azure Quantum Resource Estimator](xref:microsoft.quantum.overview.intro-resource-estimator) creates a resource estimation profile that shows how the subroutine operations in the program, for example the `FullAdder` inside `RippleCarryAdd`, contribute to the overall costs.

1. You can enable profiling by setting the `call_stack_depth` variable in the `profiling` group. The `call_stack_depth` variable is a number that indicates the maximum level up to which subroutine operations are tracked. The entry point operation is at level 0. Thus, any operation called from the entry point is at level 1, any operation therein at 2, and so on. The call stack depth is setting a maximum value to an operation's level in the call stack for tracking resources in the profile.

    ```python
    params = estimator.make_params()
    params.arguments["bitwidth"] = 32
    params.profiling.call_stack_depth = 10 # entrypoint + 10 sub operations
    ```

2. Next, you submit a resource estimation job for the ripple-carry adder by providing the Q# operation `EstimateRippleCarryAdd` and the job parameter object. You store there the results of the job in the variable `result_rca`, where RCA is an abbreviation for ripple-carry adder.

    ```python
    job = estimator.submit(EstimateRippleCarryAdd, input_params=params)
    result_rca = job.get_results()
    ```

    There are two important concepts to review to best understand the outputs of the profiling feature, a call graph and a call tree.
    
    * **Call graph**: The call graph is a static representation of the quantum program which informs which operations call which other operations. For example, `RippleCarryAdder` calls `FullAdder`, but both of these operations call `CNOT`. The call graph contains a node for each operation and a directed edge for each calling relation. The call graph may contain cycles, for example, in the case of recursive operations.
    
    * **Profile**: The profile is a dynamic tree representation of the program execution in which there are no cycles and for each node there is a clear path from the root node. For example, distinguishes the calls to `CCNOT` from `GRounds` and `CRounds` within the `ComputeCarries` operation in the carry-lookahead adder.


3. You can inspect the call graph by calling the `call_graph` property on the result object. It displays the call graph with the node corresponding to the entry point operation at the top and aligns other operations top-down according to their level.

    ```python
    result_rca.call_graph
    ```

    As described above, you can see that `RippleCarryAdd` calls both the `FullAdder` and `CNOT`, and that the `FullAdder` also calls `CNOT` and `ApplyAnd`. In fact, the `CNOT` operation is called from three other operations.

4. Next, you also generate resource estimates for the carry-lookahead implementation.

    ```python
    job = estimator.submit(EstimateLookAheadAdd, input_params=params)
    result_cla = job.get_results()
    ```

5. Again, you can look at its call graph.

    ```python
    result_cla.call_graph
    ```

    This call graph is much larger since much more operations are involved. We can see the `PRounds`, `GRounds`, and `CRounds` operations, and see that `GRounds` and `CRounds` call `CCNOT`, but `PRounds` calls `ApplyAnd`. See also that the adjoint variant of `PRounds` calls the adjoint variant of `ApplyAnd`.

    > [!NOTE]
    > You can obtain a more compact call graph (and resource estimation profile) by inlining some functions. For example, you can use the `inline_functions` parameter for those that just call a different function and have no other logic inside. This will inline the `ApplyAnd` operation into `PRounds` and `FullAdder` into `RippleCarryAdd`.
    >
    >   ```python
    >    params.profiling.inline_functions = True
    >    job = estimator.submit(EstimateLookAheadAdd, input_params=params)
    >    result_cla_inline = job.get_results()
    >    result_cla_inline.call_graph
    >    ```
    >
    > Although the call graph is now smaller, certain details, such as the rounds in `ComputeCarries`, have been inlined and are no longer available for analysis. Thus, the value for the inline parameter is typically chosen based on the desired type of analysis. In this sample, you will be considering profiles for the adder operations without inlining.

6. Next, you inspect the resource estimation profiles of both results. These provide the subroutine operations' contributions to runtime, logical depth, physical qubits for the algorithm, and logical qubits for the algorithm. Let's first have an overview of the total counts for each result:

    ```python
    breakdown_rca = result_rca['physicalCounts']['breakdown']
    breakdown_cla = result_cla['physicalCounts']['breakdown']
    
    pandas.DataFrame(data = {
        "Runtime": [f"{result_rca['physicalCounts']['runtime']} ns", f"{result_cla['physicalCounts']['runtime']} ns"],
        "Logical depth": [breakdown_rca['logicalDepth'], breakdown_cla['logicalDepth']],
        "Physical qubits": [breakdown_rca['physicalQubitsForAlgorithm'], breakdown_cla['physicalQubitsForAlgorithm']],
        "Logical qubits": [breakdown_rca['algorithmicLogicalQubits'], breakdown_cla['algorithmicLogicalQubits']],
    }, index = ["Ripple-carry adder", "Carry-lookahead adder"])
    ```

7. The resource estimation profile is generated as a JSON file that you can read using the [speedscope](https://www.speedscope.app/) interactive online profile viewer. In order to generate and download the file, you can call the `profile` property on a result object.

    ```python
    result_rca.profile
    ```

8. After downloading and opening the profile, you see first the runtime profile. For this ripple-carry adder, you can readily see that all the runtime cost is caused by the `AND` operation inside the full adder. No other operation contributes to the overall runtime. The top most box is the entry point operation and the root note of the call tree. In the next layer below are all operations that are called from the root node and that contribute to the overall runtime.

    :::image type="content" source="media/profile_rca_runtime.png" alt-text="Diagram of the algorithm runtime showing the result of a resource estimation job of a ripple-carry adder using the profiling feature.":::

9. On the top center is a button that displays _Runtime (1/4)_, which we can press to look at profiles for the other metrics. If you click on _physical algorithmic qubits_, you get this profile:

    :::image type="content" source="media/profile_rca_qubits.png" alt-text="Diagram of the number of qubits showing the result of the resource estimation job of a ripple-carry adder using the profiling feature.":::

    For the number of qubits, you account how many new qubits are allocated by some operation and track the maximum number of allocated qubits. In this sample, `EstimateRippleCarryAdd` allocates all qubits for the adder and there are no additional helper qubits. The entry point operation accounts for the additional qubits that are required for the padding in the 2D layout on the surface code.

10. Let's also generate the profile for the carry-lookahead adder.

    ```python
    result_cla.profile
    ```

11. The runtime profile for the carry-lookahead adder contains more operations:

    :::image type="content" source="media/profile_cla_runtime.png" alt-text="Diagram of the algorithm runtime showing the result of a resource estimation job of a carry-lookahead adder using the profiling feature.":::

    You can see that the `AND` gates in the `LookAheadAdd` operation and the `ComputeCarries` operation contribute to the overall runtime. Inside the `ComputeCarries` operation, you can analyze the contribution of the subroutine operations for the different rounds. For example, see that the adjoint variant for the `PRounds` takes the fewest time, whereas all other three rounds are of similar complexity.

12. In the qubit profile, you see that some additional helper qubits are allocated in the `ComputeCarries` operation to hold the auxiliary bits, that are computed out-of-place and required in the computation of the `GRounds` and `CRounds`.

    :::image type="content" source="media/profile_cla_qubits.png" alt-text="Diagram of the number of qubits showing the result of the resource estimation job of a carry-lookahead adder using the profiling feature":::

## Advance analysis of profiling results

Finally, you can access the profiling data and examine the impact of `PRounds`, `GRounds`, and `CRounds`. For simplicity, let's do it only on the carry-lookahead adder's runtime for increasing bit widths.

1. First, you establish parameters for a [batching job](xref:microsoft.quantum.work-with-resource-estimator#batching-with-q) with bit widths ranging from $2^1 =2$ to $2^{10} = 1024$, with power-of-2 steps. You also set the call stack depth to 10.

    ```python
    bitwidths = [2**k for k in range(1, 11)]
    params = estimator.make_params(num_items=len(bitwidths))
    
    params.profiling.call_stack_depth = 10
    for i, bitwidth in enumerate(bitwidths):
        params.items[i].arguments['bitwidth'] = bitwidth
    ```

    > [!IMPORTANT]
    > While the call stack depth parameter applies to all items in the batching parameters, the bit width must be specified for each item individually.

2. Next, you submit the job and store its results in `results_all`.

    ```python
    job = estimator.submit(EstimateLookAheadAdd, input_params=params)
    results_all = job.get_results()
    ```

3. The raw profiling data can be accessed in JSON format via the 'profile' key in a resource estimation result. For example, to access the profiling data for the first item in `results_all`, you use:

    ```python
    results_all[0]['profile']
    ```

3. The profiling data is organized as a tree, with each node corresponding to a subroutine operation. Each node in the tree is assigned a frame with an index, and the profile contains samples organized by calling order, with each sample assigned a weight (for example, runtime). To locate the frame associated with a given round name, for example `PRounds`, the following Python function can be used: it finds the frame, then identifies all samples that contain it, and sums up the corresponding weights.

    ```python
    def rounds_runtime(profile, round_name):
        # Find the frame index which name contains `round_name` and ends in "body".
        frame_index = [i for (i, frame) in enumerate(profile['shared']['frames']) if re.match(f'.*{round_name}.+body', frame['name'])][0]
    
        # The runtime profile is the first profile
        runtime_profile = profile['profiles'][0]
    
        # Get variables to the samples and weights field of the runtime profile
        samples = runtime_profile['samples']
        weights = runtime_profile['weights']
    
        # Sum up all the weights that correspond to samples that contain the operation,
        # i.e., that contain the frame_index
        return sum(weight for (sample, weight) in zip(samples, weights) if frame_index in sample)
    ```

4. Finally, you extract the profile for each job result item and use the `rounds_runtime` function to obtain the runtime for each round, add it to a data frame together with the total runtime and return a plot.

    ```python
    entries = []
    for idx in range(len(results_all)):
        bitwidth = bitwidths[idx]
        result = results_all[idx]
        profile = result['profile']
    
        total_runtime = result['physicalCounts']['runtime']
        entries.append([
            rounds_runtime(profile, "PRounds"),
            rounds_runtime(profile, "GRounds"),
            rounds_runtime(profile, "CRounds"),
            total_runtime
        ])
    
    df = pandas.DataFrame(data=entries, index=bitwidths, columns=["PRounds", "GRounds", "CRounds", "Total"])
    ax = df.plot(logx=True, xticks=bitwidths, xlabel="Bitwidth", ylabel="Runtime (ns)")
    # show all xticks
    from matplotlib.text import Text
    ax.set_xticklabels([Text(b, 0, b) for b in bitwidths]);
    ```

Note how the total runtime grows much faster compared to the runtime of the rounds. The reason is that we need $\mathcal{O}(n)$ `AND` gates in the preparation part of `LookAheadAdd` but only $\mathcal{O}(\log n)$ `AND` and `CCNOT` gates in the `ComputeCarries` operation.

Further note that logical depth of a the carry-lookahead adder is also logarithmic in $n$, since on the logical level, all `AND` and `CCNOT` gates, in both the preparation parts and in the rounds can be applied in parallel. However, when mapping to surface code operations using Parallel Synthesis Sequential Pauli Computation (PSSPC), these operations are sequentialized. For more information, see Appendix D in [Assessing requirements to scale to practical quantum advantage](https://arxiv.org/pdf/2211.07629.pdf).

## Next steps

You now know how to use the profiling feature of the Azure Quantum Resource Estimator to analyze how different parts of your program contribute to overall resource estimates. If you want to further explore this feature, you can try out the following ideas:

* Experiment with changing the `call_stack_depth` parameter.
* Investigate call graphs and profiles of recursive programs.
* Generate profiles from programs in other notebooks.
* Perform an advanced profile analysis to compare the number of helper qubits to the number of input and output qubits in the carry-lookahead implementation.