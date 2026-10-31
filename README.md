# Jane Street ASIC Puzzle 2026

**Jacob Babula** · **Selected as a favorite solution by Jane Street**

Jane Street's ASIC puzzle started with a chip layout and a question: what does the circuit do? I reconstructed the circuit from the layout, recovered the puzzle hidden inside, and used the circuit itself to find and verify the answer. This is my writeup of how I got there.

## Answer

`(* TWO STARS *)`

The chip turned out to be an 11×11 two-star Star Battle checker. It reads one bit on each enabled clock, for 121 bits total. A valid input has exactly two stars in every row, column, and irregular region, with no stars touching in any direction, including diagonally. This particular board has one solution.

The part I liked most was that I could recover both the rules and the region map from the circuit itself. I didn't have to guess the puzzle from the layout image and then make a separate Star Battle solver. That meant the final input came directly from the recovered circuit instead of from a hand-built copy of the puzzle.

## 1. Getting a netlist from the layout

I started by opening the GDS with KLayout and gdstk and comparing the placed cells with the SKY130 standard-cell library. To recover connectivity, I flattened the top-level routing, merged shapes on each conductor layer, and connected neighboring metal layers only where the correct via cut was present. I kept the original cell placements separately so I could map the connected geometry back to gates afterward.

My first extraction was wrong, although it looked reasonable at first. I was probing the full LEF pin rectangles, so a wire passing over a cell could be mistaken for a real pin connection. The netlist was dense and even simulated without immediately failing, which made the problem harder to notice.

I fixed it by using the pin-marker geometry from each SKY130 cell's GDS. I matched the marker centers to their logical LEF pins, transformed the points by the placed cell's orientation, and probed the flattened layout only at those exact locations. I also required the local interconnect and routed metal components to agree. That removed the false crossover connections.

The corrected netlist had 728 functional standard cells and 92 flip-flops. It also recovered the input, clock, enable, active-low reset, success, and eight output bits, with no multiply-driven nets.

I then assigned Boolean behavior to the recovered SKY130 cells and made a small clocked simulator in Python. Before trying to solve the chip, I replayed the first supplied VCD attempt. My simulator returned `TRY AGAIN`, matching the reference trace. That gave me confidence that the extraction was right because the test exercised the routing, gate models, sequential timing, and output generator—not only the logic around success.

## 2. Figuring out what the circuit checked

The physical placement already hinted at repetition. Similar groups of registers and logic appeared in bands across the layout. Instead of trying to name every block by eye, I measured which input positions could affect each state bit.

I simulated all 121 one-hot input histories at the same time. Each bit of a Python integer represented one experiment, so normal bitwise operations could propagate 121 runs through every recovered gate in parallel. The influence masks fell into four patterns: eleven masks corresponding to the columns, eleven disjoint masks corresponding to the irregular regions, an 11-bit register holding the previous row, and a 2-bit counter for the current row. This also accounted for all 57 state bits in the main success logic: 22 bits for the column counters, 22 for the region counters, 11 for the previous row, and 2 for the current-row counter.

That was the point where the circuit clicked for me. The previous-row register handles vertical and diagonal touching, while horizontal touching can be checked as the current row streams in. The counters enforce two stars per row, column, and region. Together, those pieces describe a two-star Star Battle checker.

The same influence masks recovered the region map:

```text
A A A A A B B C D D E
A A F A A B C C D D E
A A F B B B B C C D E
A A F B G G G E C C E
F A F B G E E E E E E
F F F B G G G E H H H
B B B B B B G E H I I
B J J J G G G E H I I
B J J K E E E E H I I
B B J K K E E E H H H
B J J K E E E E E E E
```

## 3. Solving it

For the final step, I translated the recovered gates into Z3 Boolean expressions and unrolled the actual clocked circuit. The trace included three reset clocks with `rst_n` low, reset release, and 121 enabled input clocks.

I initially checked success immediately after the last input and got nowhere. The final comparison feeds a flip-flop, so it needs one more rising edge before success changes. After adding that capture edge, I made all 121 input bits symbolic and constrained success to be high. Z3 found a solution. I then blocked that complete 121-bit model and solved again; the second query was unsatisfiable, showing that the solution was unique.

Here is the recovered board. Each cell includes its region letter, with `*` for a star and `.` for an empty cell.

```text
A. A. A. A. A. B. B. C* D. D* E.
A* A. F. A. A. B* C. C. D. D. E.
A. A. F. B. B. B. B. C* C. D* E.
A* A. F* B. G. G. G. E. C. C. E.
F. A. F. B. G* E. E* E. E. E. E.
F. F. F* B. G. G. G. E. H* H. H.
B. B. B. B. B* B. G. E. H. I. I*
B. J* J. J. G. G. G* E. H. I. I.
B. J. J. K* E. E. E. E. H. I. I*
B. B. J. K. K. E* E. E. H* H. H.
B. J* J. K* E. E. E. E. E. E. E.
```

Every row, column, and recovered region has two stars, and no pair touches. Replaying the solution through the complete extracted circuit raises success and changes the output to:

```text
(* TWO STARS *)\0
```

## Easter eggs

The VCD had several clues hidden in it. Its `$version` string says:

> Leave no stone unturned! But for this file, consider looking at it in a  
> waveform viewer instead.

Its `$date` is `Sat Dec 31 23:59:60 2016`. The unusual 60th second is the real leap second inserted at the end of 2016.

The two failed 121-bit attempts also contain a message. Splitting each attempt into eleven 11-bit rows and reading the first seven bits of each row as little-endian 7-bit ASCII gives “The night s” and “ky awaits ”. Together they say: **“The night sky awaits”**.

The output generator had two more possible Easter eggs. A normal failure produces `TRY AGAIN`. The successful string, `(* TWO STARS *)`, contains two asterisks. The warm-up circuit also compares its sum against 496. That number is both triangular and perfect: its proper divisors add back up to 496. I'm not entirely sure if that was intended, though; maybe it’s just me thinking about the SAT too much.

## Tools used

- **KLayout** for GDS geometry and connectivity
- **gdstk** for hierarchy, placements, labels, and cell geometry
- **SKY130 GDS and LEF data** for standard-cell pin recovery
- **Python** for netlist construction, simulation, and influence analysis
- **Z3** for the symbolic solve and uniqueness check
