# Notation
In this documentation we will use the following notation:

- $Latex$ is used for:
  - variables, with lowercase letters (e.g. a variable $v$)
  - simple objects, with uppercase letters (e.g. a committee $C$)
  - consensus parameters, state variables, and object fields, with whole words, (e.g. the $Epoch$ parameter)
- $\boldsymbol{Bold}$ is used for arrays (e.g. a bitset $\boldsymbol{bs}$)
- $\mathcal{Calligrafic}$ is used for actors (e.g. a provisioner $\mathcal{P}$)
- Markdown's `code` format is used for structure reference in texts (e.g., the `Block` structure), while $\mathsf{Sans Serif}$ is used for structures in pseudocode (e.g., a $\mathsf{Candidate}$ message)[^1]
- *Italic* is used for function names (e.g. the *Hash* function)

Moreover, we will use the following symbols:
- $pk$ and $sk$ are used to denote public and secret keys, respectively
- $\sigma$ characters are used for signatures (e.g. a message signature $\sigma_{M}$)
- $\eta$ characters are used for hash digests (e.g. a block hash $\eta_B$)
- $\tau$ characters are used for time variables (e.g. the Proposal timeout $\tau_{Proposal}$)
- $\mathcal{G}$ is used to the block generator
- $\mathsf{B}^c$ is used for the candidate block
- $\mathsf{B}^w$ is used for the winning block

While we can use the dot notation to specify a structure field (e.g. $\mathsf{B}.Header$), we will prefer the use of subscript when convenient (e.g. $\mathsf{H_B} = \mathsf{B}.Header$). Generally speaking, we use subscripts to specify the object or actor to which a variable belongs to (e.g. a block's hash $\eta_\mathsf{B}$), to indicate numeric order (e.g. the $i\text{th}$ block $`\mathsf{B}_i`$), or to specify a particular instantiation (e.g. *Hash*$`_{BLS}`$). In the case of vectors, we also use the standard notation $[i]$ to indicate the $i\text{th}$ element.

When using the dot notation, we will omit intermediate fields when possible. For instance, we will use $\mathsf{B}.Seed$ to denote $\mathsf{B}.Header.Seed$

We use $\leftarrow$ to assign object fields to separate variables. For instance, $\mathsf{H_B}, \boldsymbol{txs} \leftarrow \mathsf{B}$ will assign $\mathsf{B}.Header$ to $\mathsf{H_B}$, and $B.Transactions$ to $\boldsymbol{txs}$. The assignation follows the field order in the related structure definition. If a field is not needed in the algorithm, we ignore it by assigning it to a null variable $`\_`$.

## Procedure Control
Some procedures (typically containing a loop) are executed concurrently (e.g., as parallel threads). We have these procedures be controlled using the following commands:
- $\texttt{start}(P)$: initiates the procedure $P$;
- $\texttt{stop}(P)$: interrupts the procedure $P$;
- $\texttt{restart}(P)$: stops and restart the procedure $P$.

Additionally, we define the function $\texttt{running}(P)$ which outputs $true$ if $P$ is currently running, and $false$ otherwise.

When in a $\texttt{loop}$ we use $\texttt{break}$ to stop the execution and move on to the next iteration.

<!----------------------- FOOTNOTES ----------------------->

[^1]: This duality is mostly due to limitations in Markdown's formatting
