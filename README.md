# FlyWire-Summer-Internship-2026-Submission

import csv
import random
import sys
import time
from collections import defaultdict

random.seed(31415)
P = lambda *a, **kw: (print(*a, **kw), sys.stdout.flush())

MAX_CANDS = 1500


def load_graph(filepath):
    out_adj = defaultdict(set)
    in_adj = defaultdict(set)
    with open(filepath, 'r') as f:
        reader = csv.reader(f)
        next(reader)
        for row in reader:
            s, t = int(row[0]), int(row[1])
            if s != t:
                out_adj[s].add(t)
                in_adj[t].add(s)
    all_nodes = set(out_adj.keys()) | set(in_adj.keys())
    for n in all_nodes:
        if n not in out_adj: out_adj[n] = set()
        if n not in in_adj: in_adj[n] = set()
    return dict(out_adj), dict(in_adj), list(all_nodes)


def compute_sig_groups(out_adj, in_adj, ordered, cand_set, min_e=1):
    """Build all connection signatures via inverted index — O(N × degree)."""
    sig_bits = defaultdict(set)
    for i, s in enumerate(ordered):
        for t in out_adj.get(s, set()):
            if t in cand_set:
                sig_bits[t].add(2 * i + 1)
        for t in in_adj.get(s, set()):
            if t in cand_set:
                sig_bits[t].add(2 * i)
    groups = defaultdict(list)
    for c, bits in sig_bits.items():
        if len(bits) >= min_e:
            groups[frozenset(bits)].append(c)
    return dict(groups)


def greedy_extend(out_adjs, in_adjs, seeds, max_steps=2500):
    nd = len(out_adjs)
    current = [list(s) for s in seeds]
    cur_sets = [set(s) for s in seeds]

    for step in range(max_steps):
        sig_maps = []
        for d in range(nd):
            cands = set()
            for node in current[d]:
                cands.update(out_adjs[d].get(node, set()))
                cands.update(in_adjs[d].get(node, set()))
            cands -= cur_sets[d]
            if len(cands) > MAX_CANDS:
                cands = set(random.sample(list(cands), MAX_CANDS))

            smap = compute_sig_groups(
                out_adjs[d], in_adjs[d], current[d], cands)
            sig_maps.append(smap)

        common = set(sig_maps[0].keys())
        for d in range(1, nd):
            common &= set(sig_maps[d].keys())
        if not common:
            break

        best = max(common, key=lambda p: len(p))
        for d in range(nd):
            node = sig_maps[d][best][0]
            current[d].append(node)
            cur_sets[d].add(node)

        if (step + 1) % 200 == 0:
            P(f"    step {step+1}: N={len(current[0])}, profiles={len(common)}")

    return current


def sample_seeds(out_adjs, node_lists, nd=3):
    for _ in range(2000):
        edges = []
        for d in range(nd):
            for _ in range(100):
                u = random.choice(node_lists[d])
                out_u = out_adjs[d].get(u, set())
                if out_u:
                    v = random.choice(list(out_u))
                    edges.append((u, v))
                    break
        if len(edges) < nd:
            continue
        rev = [edges[d][0] in out_adjs[d].get(edges[d][1], set()) for d in range(nd)]
        if all(r == rev[0] for r in rev):
            return [[edges[d][0], edges[d][1]] for d in range(nd)]
    return None


def verify(out_adjs, result):
    N = len(result[0])
    mats = []
    for d in range(len(out_adjs)):
        mat = tuple(
            tuple(1 if result[d][j] in out_adjs[d].get(result[d][i], set()) else 0
                  if i != j else 0 for j in range(N))
            for i in range(N))
        mats.append(mat)
    valid = all(m == mats[0] for m in mats)
    edges = sum(sum(row) for row in mats[0])
    return valid, edges


def main():
    P("Fast Solver (inverted-index signatures)")
    P("=" * 60)

    combo = ('BANC', 'FAFB', 'MCNS')
    paths = {
        'BANC': 'D:/FlywireCodex/banc_626_edge_list.csv',
        'FAFB': 'D:/FlywireCodex/fafb_783_edge_list.csv',
        'MCNS': 'D:/FlywireCodex/mcns_0.9_edge_list.csv',
    }

    P("Loading graphs...")
    out_adjs, in_adjs, node_lists = [], [], []
    for name in combo:
        t0 = time.time()
        o, i, n = load_graph(paths[name])
        out_adjs.append(o)
        in_adjs.append(i)
        node_lists.append(n)
        P(f"  {name}: {len(n):,} nodes ({time.time()-t0:.1f}s)")

    best_N = 0
    best_result = None

    def save_intermediate(result, N):
        path = 'D:/FlywireCodex/solution_fast.csv'
        with open(path, 'w', newline='') as f:
            writer = csv.writer(f)
            writer.writerow(list(combo))
            for i in range(N):
                writer.writerow([result[d][i] for d in range(3)])
        if N > 829:
            with open('D:/FlywireCodex/solution.csv', 'w', newline='') as f:
                writer = csv.writer(f)
                writer.writerow(list(combo))
                for i in range(N):
                    writer.writerow([result[d][i] for d in range(3)])

    # Phase 1: Many random-seeded trials
    P(f"\nPhase 1: Random-seeded greedy (40 trials)")
    for trial in range(40):
        seeds = sample_seeds(out_adjs, node_lists)
        if not seeds:
            continue
        t0 = time.time()
        result = greedy_extend(out_adjs, in_adjs, seeds)
        N = len(result[0])
        elapsed = time.time() - t0
        if N > best_N:
            best_N = N
            best_result = result
            save_intermediate(result, N)
            P(f"  Trial {trial}: N={N} ({elapsed:.1f}s) *** New best (saved)")
        elif (trial + 1) % 10 == 0:
            P(f"  Trial {trial}: N={N} ({elapsed:.1f}s) (best={best_N})")

    # Phase 2: Perturbation from best
    if best_result and best_N > 100:
        P(f"\nPhase 2: Perturbation (30 trials, starting from N={best_N})")
        for trial in range(30):
            n_remove = random.randint(20, min(400, best_N // 3))
            base_N = best_N - n_remove
            trimmed = [best_result[d][:base_N] for d in range(3)]
            result = greedy_extend(out_adjs, in_adjs, trimmed)
            N = len(result[0])
            if N > best_N:
                best_N = N
                best_result = result
                save_intermediate(result, N)
                P(f"  Perturb {trial} (rm={n_remove}): N={N} *** New best! (saved)")
            elif (trial + 1) % 10 == 0:
                P(f"  Perturb {trial}: N={N} (best={best_N})")

    # Phase 3: More perturbation with different removal sizes
    if best_result and best_N > 200:
        P(f"\nPhase 3: Deep perturbation (20 trials, best={best_N})")
        for trial in range(20):
            n_remove = random.randint(best_N // 2, best_N - 10)
            base_N = best_N - n_remove
            trimmed = [best_result[d][:base_N] for d in range(3)]
            result = greedy_extend(out_adjs, in_adjs, trimmed)
            N = len(result[0])
            if N > best_N:
                best_N = N
                best_result = result
                save_intermediate(result, N)
                P(f"  Deep {trial} (rm={n_remove}): N={N} *** New best! (saved)")
            elif (trial + 1) % 5 == 0:
                P(f"  Deep {trial}: N={N} (best={best_N})")

    # Verify and save
    P(f"\n{'='*60}")
    P(f"FINAL BEST: N={best_N}")

    if best_result:
        valid, edges = verify(out_adjs, best_result)
        density = edges / (best_N * (best_N - 1)) * 100 if best_N > 1 else 0
        P(f"Verified: {valid}, edges={edges}, density={density:.2f}%")

        path = 'D:/FlywireCodex/solution_fast.csv'
        with open(path, 'w', newline='') as f:
            writer = csv.writer(f)
            writer.writerow(list(combo))
            for i in range(best_N):
                writer.writerow([best_result[d][i] for d in range(3)])
        P(f"Saved {path}")

        if best_N > 829:
            path2 = 'D:/FlywireCodex/solution.csv'
            with open(path2, 'w', newline='') as f:
                writer = csv.writer(f)
                writer.writerow(list(combo))
                for i in range(best_N):
                    writer.writerow([best_result[d][i] for d in range(3)])
            P(f"Saved as solution.csv (N={best_N})")


if __name__ == '__main__':
    main()
