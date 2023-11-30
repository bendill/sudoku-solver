## Solving Sudokus

Sudokus can be modelled as constraint satisfaction problems (CSP), where certain values must be assigned to variables to satisfy a set of constraints. The sudokus tackled by this program are 9x9 grids, where each cell (variable) must contain one and only one value between 1 and 9. They also have the following constraints:
- Each row must contain every number between 1 and 9
- Each column must contain every number between 1 and 9
- Each ninth (3x3 box) in the grid must contain every number between 1 and 9

There are many approaches to solving constraint satisfaction problems such as this. This program contains three different algorithms of varying sophistication, each of which I will explain in this document.

## Initial Solutions

#### Backtracking Depth-First Search with Constraint Propagation

My first implementation of a sudoku solver uses a backtracking depth-first search with constraint propagation. A simple backtracking depth-first search is a brute-force approach which tests the validity of each value assignment to each variable. This is a very inefficient approach, with an exponential time complexity. It can be improved using constraint propagation, where the constraints of each value assignment are propagated to other variables. For example, assigning the value 1 to row 1, column 1 of a sudoku results in the value 1 being removed from the domain of all other variables in row 1, column 1 and box 1.

My implementation of this algorithm uses a 2D array to store known final values for each variable, as well as a 2D array of lists to store the possible values (domain) for each variable. In each recursive step, a variable is chosen and the assignment of each possible value is tested for validity. When a value is set, the constraints that it causes are propagated to variables in its row, column and 3x3 box by removing the value from their domains. If any cells have only one possible value after this propagation, these values are then set using the same method. The base case of this recursion is when either a cell has no possible values, in which case 'None' is returned, or the sudoku is solved, in which case the board state is returned. My implementation of this algorithm is based on another implementation which solves the N-Queens Problem (University of Bath, n.d.).

To optimise this algorithm, I used the Minimum Remaining Values heuristic to choose the next variable to assign values to. This heuristic examines every variable and selects the one with the fewest possible values, reducing solving times dramatically. If I were to further improve my implementation, I would use a heuristic to arrange the order of a variable's value assignments so that values causing the fewest constraints are prioritised.

Overall, this algorithm is reasonably fast at solving 9x9 sudokus, and is guaranteed to find a solution if one exists. However its simple, almost brute-force approach is much more inefficient than that of other algorithms.

#### Minimum Conflicts

I then experimented with a hill climbing algorithm using a Minimum Conflicts heuristic. This is a type of local search, where all variables are assigned initial values which are then repeatedly changed in a way that reduces the number of conflicts. In the case of sudoku, the number of conflicts caused by a value can be measured as the number of times that value appears in its row, column and 3x3 box. Values are changed in this way until a maximum, local or global, is found.

This algorithm has the potential to be a lot faster than the backtracking search, especially with easier sudokus, and has seen use in the scheduling of the Hubble Space Telescope (Norvig, Russel, 2016). However, this algorithm has the big disadvantage of being unable to escape local maximums, meaning it may get to a non-goal state where no values can be changed to reduce the number of conflicts. This can be somewhat resolved by repeating the whole process a set number of times, starting with a random assignment of values each time. Either way, however, the algorithm isn't guaranteed to find a solution, and is better suited to tasks to which there is guaranteed a solution.

My implementation of this algorithm is a relatively naive one. It allocates initial values randomly, then repeatedly selects variables at random, choosing the value with the fewest number of conflicts. It is semi-decidable, meaning it stops when a solution is found, but requires a 'max_steps' parameter to prevent it from searching infinitely for a solution when there is none. If I were to improve this algorithm, I would have it return 'None' when a local maximum is reached, instead of continuing to search. It could further be improved by implementing a greedy algorithm to allocate initial values (Norvig, Russel, 2016), which could help to avoid local maxima.

## Exact Cover Solution

My chosen solution represents a sudoku as an exact cover problem, and uses Knuth's Algorithm X (Knuth, 2000) to solve it. Exact cover problems can be represented with a matrix, where columns represent constraints and rows represent all possible values for all variables. Each element in the matrix is a Boolean value which denotes whether or not its row (value assignment) satisfies its column (constraint). For example, an assignment of the value 1 in row 1, column 1 satisfies the following constraints:
- row 1, column 1 contains a value
- row 1 contains the value 1
- column 1 contains the value 1
- box 1 contains the value 1

Each possible value assignment will satisfy exactly four constraints similar to those above. To solve the exact cover problem, a set of rows must be chosen such that each constraint is satisfied once and only once. For example, the constraint 'row 1 contains the value 1' must be satisfied by exactly one value assignment, otherwise there wouldn't be exactly one value of 1 in row 1, resulting in an invalid sudoku. The solution to the problem is the set of rows that make up the exact cover.

Knuth's Algorithm X is a method to solve exact cover problems. It chooses the column with the fewest possibilities (rows), and attempts to select each possible row in that column. Rows which clash with the selected row (i.e. rows which share a satisfied constraint with the selected row) are removed from the matrix, followed by the now satisfied columns. This process is repeated with a backtracking depth-first search, until either the matrix is empty (meaning all constraints are satisfied, and a solution is found), or a column in the matrix is empty (meaning a constraint can't be satisfied, so it is no use searching deeper in that subtree).

Representing a sudoku as an exact cover problem breaks it down into its fundamental components by storing each possible value assignment and how it affects each constraint. This allows an algorithm to scan through constraints and test value assignments in a very efficient way, resulting in consistently fast solving times.

#### Implementation

Instead of representing the exact cover problem as a sparse 729x324 matrix, my implementation stores the state as a dictionary whose keys are column numbers, and values are lists containing the row numbers which satisfy that column (i.e. the elements with the value 'True' in the matrix). Not only is this representation more memory efficient, but it also allows for faster searching of the data structure, causing better performance and improved scalability. My implementation also stores a constant 2D numpy array to allow quick access to the column numbers which a given row satisfies. The row numbers chosen so far by the algorithm are stored in a list, which can be translated to value assignments in the form of the following tuple: (row, column, value).

The rows representing the initial assignments of a given sudoku are chosen when the state object is created. If any of these initial rows clash with each other (e.g. if two values in row 1 have the assignment of value 1), the sudoku is marked as unsolvable. Otherwise, the initial rows are chosen and removed in bulk, in a similar way to Algorithm X, before the algorithm begins searching.

My initial implementation used the function 'deepcopy' to create a new copy of the sudoku state before selecting rows and columns to remove in each iteration. This function creates a completely new state object, which is both memory inefficient and time consuming. Instead of using this function, my final implementation reverts back to its previous state each time it backtracks. This works alongside recursion with the following steps:
1. choose a possible row, and remove rows and columns accordingly
2. recurse
3. if solution is found, return state
4. revert previous changes, before choosing the next possible row

If a solution is ever found, it is recursively returned back up the tree before any of its changes are reverted. To keep track of the changes made in each step, a stack stores lists of the removed rows. Whenever the object is reverted to its previous state, the stack is popped, and the rows, along with the columns they satisfy, are added back to the model. The most recent row added to the 'final_rows' list is also removed, because it isn't part of the solution.

Knuth's suggested implementation of his algorithm uses a technique called dancing links (Knuth, 2000). This is a particularly efficient technique that uses circular doubly linked lists to quickly add and remove list nodes, and can be applied to his algorithm by representing the exact cover model using lists, in a similar way to my implementation. If I were to improve my program further, I would implement Knuth's Algorithm X using dancing links, possibly in a programming language that gives the programmer more control over pointers, such as C.

## References

Knuth, D.E., 2000. *Dancing links* [Online]. s.s. Available from: https://arxiv.org/abs/cs/0011047 [Accessed 9 March 2021].

Norvig, P., Russel, S., 2016. *Artificial intelligence: a modern approach* [Online]. Global Edition. s.l.: Pearson Education. Available from: https://ebookcentral.proquest.com/lib/bath/reader.action?docID=5495854 [Accessed 8 March 2021].

University of Bath, n.d. *02 eightqueens* [computer program]. Available from: https://moodle.bath.ac.uk/course/view.php?id=59592&section=6 [Accessed 10 February 2021].