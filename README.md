# FlyWire-Summer-Internship-2026-Submission


**Approach**

Here, one of the largest directed induced subgraph isomorphic across three Drosophila connectomic datasets were identified. This algorithm uses connection profile matching with inverted-index acceleration: 

For each candidate neuron, a binary signature encodes all directed edges to/from existing subgraph members. Rather than computing signatures per-candidate:           

        (O(N×|cands|)) 

We iterate adjacency lists of existing members to build all signatures in one pass 

       (O(N×avg_degree))
       
This helps achieving **20-100×** speedup per step. Candidates sharing identical signatures across all datasets preserve isomorphism by construction. We run 40 randomseeded trials plus 50 perturbation trials (trimming 1-99% of the best solution and re-extending) to escape local optima.


**Technical strategy**

I modeled each Codex dataset as an unweighted directed graph and ignored the synapse count weights as required. The goal was to find three neuron lists of equal length such that the induced directed adjacency matrix is identical across the three selected datasets. In this case, datasets (BANC, FAFB, and MCNS) were selected. The reason I selected these datasets is because these three datasets provide a large-scale and broader coverage of the adult Drosophila brain and/or the central nervous system (CNS). On the other hand, the MAOL and MANC are region-specific datasets that cover only the optic lobe and the venral nerve cord respectively. This results in a limited anatomical coverage which makes them less suitable for finding a large sets of motif between different datasets.


**Main Methods**

The algorithm started with a small matching connection (a seed) across three datasets and gradually expanded it by adding one neuron at a time. A neuron was added only if its pattern of incoming and outgoing connections to the already-selected neurons matched the corresponding pattern in the other datasets. This ensured that the connectivity structure remained identical as the shared circuit grew.

In order to identify the largest N possible across datasets, open source data from Flywire Codex were used and final output CSV data was checked manually by looking into each of the random Root ID between all 3 datasets and and grouping based of their similarities betweeen:

       1. Cell Type
       2. Soma side
       3. Flow
       4. Hemilineage

A total amount of 10% output data was selected for this manual verification. This step is to ensure that the final analyzed data produced are isomorphic and is indeed a motif between all 3 different sets of connectomes.


**Graph-matching method**

For each candidate neuron, I encoded its relationship to the current subgraph as a binary signature:

       existing_i → candidate

       candidate → existing_i

for every existing matched neuron i.

Candidates with identical signatures across the three datasets can be added while preserving directed induced-subgraph isomorphism. The final solution was independently verified by reconstructing the full N × N adjacency matrices for all three datasets and checking exact equality.


**Acceleration**

Instead of computing every candidate signature directly, the code uses an inverted-index signature method. It scans the incoming and outgoing adjacency lists of current subgraph members and updates signatures for all neighboring candidates in one pass.

This reduces each extension step from roughly:

       O(N × number_of_candidates)

to:

       O(N × average_degree)

which produced the reported **20–100×** speedup.


**Heuristics**

Here, the code that I used does not provide an exact maximum isomorphism solver. It first randomly selects compatible edge seeds to start the matched subgraph. Then, it uses a greedy extension to choose the shared signature within the largest number of edges to the current circuit. In order to search for the best network motif in these datasets, I performed a candidate pruning which limits the candidate pools to a maximum of 1500 with the code **(MAX_CANDS = 1500)**. 
Then, by performing a random restarts, I was able to do a perturbation search and trim out the poorer candidates by re-evaluating the best solution and extending my motif with those that produce a higher matching score. 

Here, to improve the chances of finding a better motif, the search was repeated multiple times using different random starting points. Each dataset triplet undergoes a series of perturbation trials which includes 30 shallow perturbations (making a small changes to the current motif) and 20 deep perturbations (exploring more substantial changes). After each trial, weaker candidates were discarded, while higher-scoring motifs were retained and further expanded. This process allowed the algorithm to gradually refine the motif and identify a stronger and more conserved network structures.

To ensure that the results were not specific to a single set of datasets, the same procedure was repeated for every possible combination of three datasets selected (BANC, FAFB, MCNS) from the five available datasets, resulting in 10 dataset triplets **(C(5,3) = 10)**. This approach enabled a broad comparison of network motifs across multiple connectome datasets.


**Assumptions**

The connectome data were treated as simple directed graphs, where only the existence of a connection between two neurons was considered. Synapse counts (edge weights) and self-connections (self-loops) were ignored. In the output CSV file, neurons listed on the same row are considered matching neurons across the three datasets.

To find a large shared circuit, the algorithm used a randomized search strategy that repeatedly grew matching neuron groups, starting from different random seeds and refining the best solutions found. Because the datasets are very large, this approach does not guarantee the absolute best possible solution, but it efficiently finds very large conserved circuits.

Finally, every solution was verified by reconstructing the connectivity pattern of the selected neurons in each dataset and checking that all three graphs had exactly the same directed connections. This ensures that the reported circuit is a valid shared directed induced subgraph.


**Reproduction instructions**

1. Place the five Codex edge-list CSV files in the expected directory.

2. Update the paths dictionary to point to the local files.

3. Set the random seed:

       random.seed(31415)

4. Run the solver for each of the ten 3-dataset combinations.

For each triplet, run:

    40 random-seeded greedy trials
    30 shallow perturbation trials
    20 deep perturbation trials
   

5. Save the largest verified solution as solution.csv.

6. Confirm validity using **verify()**, which checks that the three induced adjacency matrices are identical.

By using this procedure, the largest solution found was **N = 2311** across BANC, FAFB, and MCNS, with 2451 directed edges.

**Primary submission**

Maximum N: An isomorphic subgraph of N = 2311 neurons across BANC (female brain + nerve cord, 112,885 neurons), FAFB (female adult brain, 138,584 neurons), and MCNS (male complete CNS, 165,820 neurons) were found. The shared adjacency matrix contains 2451 directed edges (0.05% density). Exhaustive scanning of all C(5,3)=10 dataset triplets confirms this combination yields the largest shared structure; the runner-up (BANC+MAOL+MCNS) reaches only N≈1259 under equivalent search effort. 

