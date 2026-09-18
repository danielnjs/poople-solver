# Poople Offline Solver

A self-contained webpage that finds the fewest one-letter changes needed to turn a four-letter word into **POOP**.

Open [poople-solver.html](poople-solver.html) in a browser with JavaScript enabled. No installation, server, account, or internet connection is required. All styling, code, and dictionary data are embedded in the HTML file.

## How to use it

1. Enter the game's four-letter starting word.
2. Select **Find shortest path**, or press Enter in the starting-word field.
3. Follow the displayed ladder. Green highlights identify the letter changed at each move.
4. Use the arrow buttons to browse other equally short routes, when available.
5. Select **Copy this path** to copy the route. If the browser blocks clipboard access, a selectable text field appears for manual copying.

The initial example is:

```text
BARN → BORN → BOON → BOOP → POOP
```

This takes **4 moves**. The starting word is step 0; five words in a ladder mean four moves.

### Adjusting accepted words

Expand **Adjust the word list** to add accepted words or exclude words your version of the game rejects. Separate entries with spaces, newlines, commas, or semicolons. Every entry must contain exactly four letters A–Z; lowercase input is converted to uppercase.

The selected dictionary is calculated as:

```text
(embedded dictionary ∪ additional accepted words) − words to avoid
```

Duplicate entries are removed automatically. Exclusions take priority over additions. Select **Find shortest path** again to apply edits. **Restore original word list** clears both fields and solves again.

Edits are held in memory only and disappear when the page closes or reloads. The solver does not silently add an unknown starting word; add it explicitly if the game accepts it.

## Game rules and graph model

Each word is a vertex in an unweighted, undirected graph. Two vertices share an edge only when their words differ in exactly one character position.

```text
BARN → BORN    valid: A changes to O
BARN → BOON    invalid: two positions change
BARN → BARN    invalid: no position changes
```

All words, including the start and target, must belong to the selected dictionary. Letters cannot be inserted, deleted, or rearranged in a single move. Every valid edge costs one move, so solving the game means finding a shortest path from the starting vertex to POOP.

## Solving workflow

```text
Starting word + dictionary edits
              ↓
Normalize and validate inputs
              ↓
Build the selected dictionary as a Set
              ↓
Breadth-first search outward from POOP
              ↓
Record minimum distance to POOP for every reachable word
              ↓
Walk from the starting word through decreasing distances
              ↓
Display up to 100 shortest routes and highlight each change
```

### 1. Generate valid neighbors

`neighbors(word, words)` examines each of the four positions. At each position it substitutes every letter from A to Z except the existing letter, then checks whether the candidate belongs to the dictionary `Set`.

This considers exactly **4 × 25 = 100 candidates** per word. It generates edges as needed instead of building and storing every pairwise connection in advance. Set membership checks are typically constant-time.

### 2. Compute minimum distances with breadth-first search

`solveLadder(start, words)` starts a breadth-first search (BFS) at **POOP**, assigning it distance 0. A queue processes words in increasing distance order. Each previously unseen neighbor receives its parent's distance plus one.

```text
distance[POOP] = 0
queue = [POOP]

for each word in queue, in insertion order:
    for each valid neighbor of word:
        if neighbor has no recorded distance:
            distance[neighbor] = distance[word] + 1
            append neighbor to queue
```

The queue uses a moving index rather than repeatedly removing its first element. A distance map doubles as the visited set, preventing repeated visits and cycles.

**Why this is shortest:** BFS explores all words one move away before exploring words two moves away, and so on. Because every edge has the same cost, the first distance assigned to a word is its minimum distance from POOP. Edges are undirected, so that is also the minimum distance from the word to POOP.

The implementation searches the entire component reachable from POOP on each solve. It does not use a heuristic such as choosing the word with the most matching target letters; a shortest route can require changing a letter that already matches the target.

### 3. Enumerate equally short routes

After BFS, the solver starts at the requested word and follows only neighbors satisfying:

```text
distance[next] = distance[current] − 1
```

Every move reduces the remaining distance by exactly one. Consequently, every completed route is shortest, and the traversal cannot cycle.

Routes are enumerated with depth-first traversal using an explicit stack. Each stack frame tracks the current word, its eligible next words, and which choice to visit next. Backtracking explores alternative choices. An explicit stack avoids depending on the browser's recursive call-stack limit.

The interface displays at most **100 routes**. The solver looks for a 101st route to determine whether to show “more exist,” then discards that extra route. It does not calculate the total number of routes beyond the cap. Routes are ordered by character position and replacement letter, not by word familiarity or frequency.

### 4. Render the result

`renderRoute()` displays the selected route with numbered steps, the minimum move count, and changed-letter highlights. Previous/next controls select among the computed routes without rerunning BFS. DOM elements are created with text content, rather than interpreting user entries as HTML.

## Errors and edge cases

| Situation | Behavior |
| --- | --- |
| Starting input is not four letters A–Z | Requests a valid starting word. |
| Starting word is absent from the selected dictionary | Requests a spelling correction or an explicit dictionary addition. |
| A dictionary edit contains an invalid entry | Rejects the edit and explains the four-letter requirement. |
| POOP has been excluded | Requests that it be removed from the exclusion list. |
| Start is valid but disconnected from POOP | Reports that no route exists in the selected dictionary. |
| Start is POOP and POOP is included | Displays POOP with zero moves. |
| An error occurs after an earlier solution | Clears the old solution so it cannot be mistaken for a current result. |

## Performance

Let **V** be the number of words in the selected dictionary, **R** the number reachable from POOP, **D** the shortest move count, and **C** the number of routes explored before the display cap stops enumeration.

- Dictionary construction takes time and space proportional to the supplied word entries.
- BFS considers 100 candidates per reachable word. With word length fixed at four, its running time is proportional to R, and its queue and distance map use O(R) space.
- Route enumeration depends on the number and length of shortest routes. It is bounded here by exploring at most 101 completed routes; with the fixed neighbor-generation cost, traversal work is O(C × D).
- Stored route data uses O(C × (D + 1)) space. The traversal stack grows with route length.

An uncapped graph can have exponentially many shortest routes. The cap limits enumeration and output without changing the minimum move count or the validity of any displayed route.

## Dictionary and offline behavior

The embedded dictionary contains **2,398 unique four-letter words**, extracted from the publicly served [poople.io](https://poople.io/) game bundle on **September 18, 2026**. The build embeds word strings; the solver computes distances itself at runtime rather than looking up prewritten answers.

Shortest-path guarantees apply to the **selected dictionary**. The live game may update its accepted words, and other Poople versions may use different lists. This is an independent solver, not an official Poople product.

The page loads no external scripts, fonts, images, or APIs. Its Content Security Policy includes `connect-src 'none'`. The optional dictionary-source hyperlink requires internet only if you choose to open it. Clipboard access is requested only when you click the copy button. No browser storage is required.

In browsers that support `document.modelContext.registerTool`, the page also registers an optional `solve_poople` tool. It validates a four-letter starting word, invokes the same visible solving workflow, and returns routes, move count, and any error. Unsupported browsers skip this integration; normal use does not depend on it.

## Implementation and development workflow

The delivered application is one file: `poople-solver.html`. Its main functions are:

| Function | Responsibility |
| --- | --- |
| `parseWords()` | Normalize and validate dictionary edits. |
| `neighbors()` | Generate dictionary words exactly one substitution away. |
| `solveLadder()` | Run reverse BFS and enumerate shortest routes. |
| `runSolve()` | Validate the form, assemble the dictionary, and coordinate solving. |
| `renderRoute()` | Display the selected path and route controls. |
| `showError()` | Report a problem and remove stale results. |
