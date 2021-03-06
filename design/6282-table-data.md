# Proposal: Design proposal for 'Tables', multi-dimensional slices

Author(s): Brendan Tracey, with input from the gonum team

Last updated: November 11th, 2015

Discussion at https://golang.org/issue/6282.

## Note
Note 2015/11/10: There are likely necessary updates based on discussion since the
original reply by Nigel Tao.
I felt it was best for the original commit to have the proposal as it was.
Changes will be made as part of the proposal process and thus have a commit
history.
A further note on this topic will be included in the 'open issues' section.

## Document Motivation:
Issue 6282 was created in August of 2013, and despite the initial flurry of
discussion, little has happened since.
Norman Yarvin’s proposal is a lengthy one, and although complete, some of the
resistance to its implementation comes from its complexity.
This document proposes a smaller set of features while leaving room for future
extension.
These features solve the majority of the current problems using multi-
dimensional data in Go, and leave some of the more advanced features for the
future.
This is thus a smaller change to the Go specification that can be grown over
time if demand allows.

## Abstract

This document proposes the addition of two-dimensional slices to Go.
2-d slices, "tables", are accessed with two indices (rather than one), with each
dimension bounds-checked for safety.
This proposal defines syntax for accessing and slicing, provides definitions for
`make`, `len`, `cap`, `copy` and `range`, and discusses some additions to package
reflect.

## Background

Go presently lacks multi-dimensional slices.
Multi-dimensional arrays can be constructed, but they have fixed dimensions: a
function that takes a multidimensional array of size 3x3 is unable to handle an
array of size 4x4.
Go provides slices to allow code to be written for lists of unknown length, but
a similar functionality does not exist for multiple dimensions; there is no built
-in "table" type.

One very important concept with this layout is a Matrix.
Matrices are hugely important in many sections of computing.
Several popular languages have been designed with the goal of making matrices
easy (MATLAB, Julia, and to some extent Fortran) and significant effort has been
spent in other languages to make matrix operations fast and easy (Lapack, Intel
MLK, ATLAS, Eigpack, numpy).
Go was designed with speed and concurrency in mind, and so Go should be a great
language for computation, and indeed, scientific programmers are using Go
despite the lack of support from the standard library for scientific computing.
While the gonum project has a matrix library that provides a significant amount
of functionality, the results are problematic for reasons discussed below.
As both a developer and a user of the gonum matrix library, I can confidently
say that not only would implementation and maintenance be much easier with a
table type, but also that using matrices would change from being somewhat of a
pain to being enjoyable to use.

The desire for good matrix support is the motivation for this proposal, but
matrices are not synonymous with tables. A matrix is composed of real or complex
numbers and has well-defined operations (multiply, determinant, cholesky
decomposition).
Tables, on the other hand, are merely a rectangular data container. A table can
hold any data type and have none of the additional semantics of a matrix.
Building a matrix on top of a table can (and should be) the job of a package. 

A rectangular data container can find use throughout the Go ecosystem.
A partial list is 

1. Image processing: An image canvas can be represented as a retangle of colors.
Here the ability to efficiently slice in multiple dimensions is important.  
2. Machine learning: Typically feature vectors are represented as a row of a
matrix. Each feature vector has the same length, and so the additional safety of
a full rectangular data structure is useful.
Additionally, many fitting algorithms (such as linear regression) give this
rectangular data the additional semantics of a matrix, so easy interoperability
is very useful.
3. 2D game development: Go is becoming increasingly popular for the development
of games.
Tile-based games are especially well represented as a slice, not only for
depiction of the field of action, but the copy semantics are especially useful
for dealing with sprites.

Go is a great general-purpose language, and allowing users to slice a
multi-dimensional array will increase the sphere of projects for which Go is ideal.
In the end, tables are the pragmatic choice for supporting rectangular data.

### Language Workarounds
There are several possible ways to emulate a rectangular data structure, each
with its own downsides.

#### 1. Slice of slices
Perhaps the most natural way to express a table in Go is to use a slice of
slices (for example `[][]float64`).
This construction allows convenient accessing and assignment using the
traditional slice access

	v := s[i][j]
	s[i][j] = v

This representation has two major problems.
First, a slice of slices, on its own, has no guarantees about the size of the
slices in the minor dimension.
Routines must either check that the lengths of the inner slices are all equal,
or assume that the dimensions are equal (and accept possible bounds errors).
This approach is error-prone for the user and unnecessarily burdensome for the
implementer.
In short, a slice of slices represents exactly that; a slice of arbitrary length
slices.
It does not represent a table where all of the minor dimension slices are of
equal length
Secondly, a slice of slices has a significant amount of computational overhead.
Many programs in numerical computing are dominated by the cost of matrix
operations (linear solve, singular value decomposition), and optimizing these
operations is the best way to improve performance.
Likewise, any unnecessary cost is a direct unnecessary slowdown.
On modern machines, pointer-chasing is one of the slowest operations.
At best, the pointer might be in the L1 cache, and retrieval from the L1 cache
has a latency similar to that of the multiplication that address arithmetic
requires.
Even so, keeping that pointer in the cache increases L1 cache pressure, slowing
down other code.
If the pointer is not in the L1 cache, its retrieval is considerably slower than
address arithmetic; at worst, it might be in main memory, which has a latency on
the order of a hundred times slower than address arithmetic.
Additionally, what would be redundant bounds checks in a true table are
necessary in a slice of slice as each slice could have a different length, and
some common operations like taking a subtable (the 2-d equivalent of slicing)
are expensive on a slice of slice but are cheap in other representations.

#### 2. Single array
A second representation option is to contain the data in a single slice, and
maintain auxiliary variables for the size of the table.
The main benefit of this approach is speed.
A single array avoids the cache and bounds concerns listed above.
However, this approach has several major downfalls.
The auxiliary size variables must be managed by hand and passed between
different routines.
Every access requires hand-writing the data access multiplication as well as hand
-written bounds checking (Go ensures that data is not accessed beyond the array,
but not that the row and column bounds are respected).
Not only is hand-written access error prone, but the integer multiply-add is
much slower than compiler support for access.
Furthermore, it is not clear from the data representation whether if the table
is to be accessed in "row major" or "column major" format

	v := a[i*stride + j]     // Row major a[i,j]
	v := a[i + j*stride]     // Column major a[i,j]

In order to correctly and safely represent a slice-backed table, one needs four
auxiliary variables: the number of rows, number of columns, the stride, and also
the ordering of the data since there is currently no "standard" choice for data
ordering.
A community accepted ordering for this data structure would significantly ease
package writing and improve package inter-operation, but relying on library
writers to follow unenforced convention is ripe for confusion and incorrect code.

#### 3. Struct type
A third approach is to create a struct data type containing a data slice and all
of the data access information.
The data is then accessed through method calls.
This is the approach used by go.matrix and gonum/matrix.

	type RawMatrix struct {
		Order
		Rows, Cols  int
		Stride      int
		Data        []float64
	}

	type Dense struct {
		mat RawMatrix
	}

	func (m *Dense) At(r, c int) float64{
		if r < 0 || r >= m.mat.Rows{
			panic("rows out of bounds")
		}
		if c < 0 || c >= m.mat.Cols{
			panic("cols out of bounds")
		}
		return m.mat.Data[r*m.mat.Stride + c]
	}

	func (m *Dense) Set(r, c int, v float64) {
		if r < 0 || r >= m.mat.Rows{
			panic("rows out of bounds")
		}
		if c < 0 || c >= m.mat.Cols{
			panic("cols out of bounds")
		}
		m.mat.Data[r*m.mat.Stride+c] = v
	}

From the user’s perspective:

	v := m.At(i, j)
	m.Set(i, j, v)

The major benefits to this approach are that the data is encapsulated correctly
-- the structure is presented as a table, and panics occur when either dimension
is accessed out of bounds -- and that speed is preserved when doing commons
matrix operations (such as multiplication), as they can operate on the slice
directly.

The problems with this approach are convenience of use and speed of execution
for uncommon operations.
The At and Set methods when used in simple expressions are not too bad; they are
a couple of extra characters, but the behavior is still clear.
Legibility starts to erode, however, when used in more complicated expressions

	// Set the third column of a matrix to have a uniform random value
	for i := 0; i < nCols; i++ {
		m.Set(i, 2, (bounds[1] - bounds[0])*rand.Float64() + bounds[0])
	}
	// Perform a matrix  add-multiply, c += a .* b  (.* representing element-
	// wise multiplication)
	for i := 0; i < nRows; i++ {
		for j := 0; j < nCols; j++{
			c.Set(i,j, c.At(i,j) + a.At(i,j) * b.At(i,j))
		}
	}

The above code segments are much clearer when written as an expression and
assignment

	// Set the third column of a matrix to have a uniform random value
	for i := 0; i < nRows; i++ {
		m[i,2] = (bounds[1] - bounds[0]) * rand.Float64() + bounds[0]
	}
	// Perform a matrix element-wise add-multiply, c += a .* b
	for i := 0; i < nRows; i++ {
		for j := 0; j < nCols; j++{
			c[i,j] += a[i,j] * b[i,j]
		}
	}

In addition, going through three data structures for accessing and assignment is
slow (public Dense struct, then private RawMatrix struct, then RawMatrix.Data
slice).
This is a problem when performing matrix manipulation not provided by the matrix
library (and it is impossible to provide support for everything one might wish
to do with a matrix).
The user is faced with the choice of either accepting the performance penalty (
up to 10x), or extracting the underlying RawMatrix, and indexing the underlying
slice directly (thus removing the safety provided by the Dense type).
The decision to not break type safety can come with significant full-program
cost; there are many codes where matrix operations dominate computational cost (
consider a 10x slowdown when accessing millions of matrix elements).

### Benchmarks
Two benchmarks were created to compare the representations: 1) a traditional
matrix multiply, and 2) summing elements of the matrix that meet certain
qualifications. The code can be found [here](http://play.golang.org/p/VsL5HGNYT4).
Six benchmarks show the speed of multiplying a 200x300 times a 300x400 matrix,
and the summation is of the 200x300 matrix.
The first three benchmarks present the simple implementation of the algorithm
given the representation.
The final three benchmarks are provided for comparison, and show optimizations
to the code at the sacrifice of some code simplicity and legibility.

Benchmark times for Partial Sum and Matrix Multiply.
Benchmarking performed on OSX 2.7 GHz Intel Core i7 using Go 1.5.1.
Times are scaled so that the single slice representation has a value of 1.

| Representation         | Partial sum | Matrix Multiply |
| ---------------------  | :---------: | :-------------: |
| Single slice           | 1.00        | 1.00            |
| Slice of slice         | 1.10        | 1.51            |
| Struct                 | 2.33        | 8.32            |
| Struct no bounds       | 1.08        | 1.96            |
| Struct no bounds no ptr| 1.12        | 4.47            |
| Single slice resliced  | 0.80        | 0.76            |
| Slice of slice cached  | 0.82        | 0.77            |
| Slice of slice range   | 0.80        | 0.74            |

And with -gcflags = -B

| Representation         | Partial sum | Matrix Multiply |
| ---------------------  | :---------: | :-------------: |
| Single slice           | 0.95        | 0.95            |
| Slice of slice         | 1.10        | 1.33            |
| Struct                 | 2.35        | 8.20            |
| Struct no bounds       | 1.13        | 1.81            |
| Struct no bounds no ptr| 1.11        | 4.48            |
| Single slice resliced  | 0.83        | 0.59            |
| Slice of slice cached  | 0.83        | 0.58            |
| Slice of slice range   | 0.77        | 0.56            |

For the simple implementations, the single slice representation is the fastest
by a significant margin.
Notably, the struct representation -- the only representation that presents a
correct model of the data and the one which is least error-prone -- has a
significant speed penalty, being 8 (!) times slower for the matrix multiply.
The additional benchmarks show that significant speed increases through
optimization of array indexing and by avoiding some unnecessary bounds checks (
issue 5364).
A native table implementation would allow even more efficient data access by
allowing accessing via increment rather than integer multiplication.
This would improve upon even the optimized benchmarks.
In the future, Go compilers will add vectorization to provide huge speed
improvements to numerical code.
Vectorization opportunities are much more easily recognized with a table type
where the data is controlled by the implementation than in any of the
alternate representation choices.
For the single slice representation, the compiler must actually analyze the
indexing integer multiply to confirm there is no overflow and that the
elements are accessed in order.
In the slice of slice representation, the compiler must confirm that all slice
lengths are equal, which may require non-local code analysis, and
vectorization will be more difficult with non-contiguous data.
In the struct representation, the actual slice index is behind a function call
and a private field, so not only would the compiler need all of the same
analysis for the single-slice case, but now must also inline a function call
that contains out-of-package private data.
A native table type provides immediate speed improvements and opens to the
door for further easily-recognizable optimization opportunities.

### Recap
The following table summarizes the current state of affairs with tables in go

|                | Correct Representation | Access/Assignment Convenience | Speed |
| -------------: | :--------------------: | :---------------------------: | :---: |
| Slice of slice | X                      | ✓                             | X     |
| Single slice   | X                      | X                             | ✓     |
| Struct type    | ✓                      | X                             | X     |

In general, we would like our codes to be

1. Easy to use
2. Not error-prone
3. Performant

At present, an author of numerical code must choose one.
The relative importance of these priorities will be application-specific, which
will make it hard to establish one common representation.
This lack of consistency will make it hard for packages to interoperate.
The addition of a language built-in allows all three goals to be met
simultaneously which eliminates this tradeoff, and allows gophers to write
simple, fast, and correct numerical and graphics code.

## Proposal

The proposal is to add a new built-in generic type, a "table" into the language.
It is a two-dimensional analog to a slice.
The term "table" is chosen because the proposed new type is just a data
container.
The term "matrix" implies the ability to do other mathematical operations (which
will not be implemented at the language level).
One may multiply a matrix; one may not multiply a table.
Just as []T is shorthand for a slice, [,]T is shorthand for a table (as will be
clear later).

### Syntax (spec level specification)

### Allocation:
A new table may be constructed either using the make built-in or as a table
literal.
The elements are guaranteed to be stored in a continuous array, and are
guaranteed to be stored in "row-major" order, which matches the existing layout
of two-dimensional arrays in Go.
Specifically, in a table t with m rows and n columns, the elements are laid out
as

	[t11, t12, … t1n, t21, t22, … t2n … , tm1, … tmn]

Guaranteeing a specific order allows code authors to reason about data layout
for optimal performance.
Row major is the only acceptable layout as tables can be considered as multi-
dimensional arrays which have been sliced, and the spec guarantees that multi-
dimensional arrays are in row-major order.

#### Using make:
A new table (of generic type) may be allocated by using the make command with
two mandatory length arguments followed by two optional capacity arguments.
If either capacity is present both must be.
If no capacity arguments are present, they are defaulted to the length
arguments.
The first length and capacity argument are for the first dimension, and the
second length and capacity are for the second dimension.
These act like the length and capacity for slices.
The table will be filled with the zero value of the type

	s := make([,]T, r, c, maxr, maxc)
	t := make([,]T, r, c)

Calling make with a zero length or capacity is allowed, and is equivalent to
creating an equivalently sized multi-dimensional array and slicing it.
In the following code

	u := make([,]float32, 0, 6)
	v := [0][6]float32{}
	w := v[0:0, 0:6]

u and w have the same behavior.
Specifically, the length and capacities for both are 0 and 6 in the two
dimensions, and the underlying data array contains 0 elements.

#### Table literals
A table literal can be constructed using nested braces

	u := [,]T{{x, y, z}, {a, b, c}}

The allocated table will have a number of rows equal to the number of sets of
braces, and will have a number of columns equal to the number of elements within
each set of braces.
For example, u has two rows and three columns.
It is a compile error if the inner braces do not all contain the same number of
elements.

### Access/ Assignment
An element of a table can be accessed with [row,column] syntax, and can be
assigned to similarly.

	var v T
	v = t[r,c]
	t[r,c] = v

If r is negative or if r >= the length in the first dimension, a runtime panic
occurs, and likewise with c and the second dimension.
Other combination operators are valid (assuming the table is of correct type)

	t := make([,]float64, 2,3)
	t[1,2] = 6
	t[1,2] *= 2   // Now contains 12
	t[3,3] = 4    // Runtime panic, out of bounds (possible compile-time with
	              // constants)

### Slicing
A table can be sliced using the normal 2 or 3 index slicing rules on each side
of the comma i:j or i:j:k.
The same panic rules as slices apply (0 <= i <= j <= k, must be less than the
capacity).
Like slices, this would update the length and capacity in the respective
dimensions

	a := make([,]int, 10, 2, 10, 15)
	b := a[1:3, 3:5:6]

Also in-line with slices, a multi-dimensional array may be sliced to create a
table.
In

	array := [10][5]int
	t := array[2:6, 3:5]

t is a table with lengths 4 and 2, capacities 5 and 10, and a stride of 5.

### Length / Capacity
Like slices, the len and cap built-ins can be used on a table.
Len and cap take in a table and return a [2]int representing the lengths/
capacities in the dimensions of the table.

	lengths := len(t)    // lengths is a [2]int
	nRows := len(t)[0]
	nCols := len(t)[1]
	maxElems := cap(t)[0] * cap(t)[1]

#### Discussion:
This behavior keeps the natural definitions of len and cap.
There are three possible syntax choices

	lengths := len(t)     // returns [2]int
	length := len(t, 0)   // returns the length of the table along the first
	                      // dimension
	rows, cols := len(t)

The first behavior is preferable to the other two.
In the first syntax, it is easy to get any particular dimension (access the
array) and if the array index is a constant, it is verifiable and optimizable at
compile-time.
Second, it is easy to compare the lengths and capacities of the array with

	len(x) == len(y)

Third, this behavior naturally extends to higher dimensional tables, as length
can be extended to return an [n]int.

The second representation seems strictly worse than the first representation.
While it is easy to obtain a specific dimension length of the table, one cannot
compare the full table lengths directly.
One has to do
	len(x,0) == len(y, 0) && len(x,1) == len(y,1)
Additionally, now the call to length requires a check that the second argument
is 0 or 1, and may panic if that check fails.
There doesn’t seem to be any benefit gained by allowing this failure.

The third option will always succeed, but again, it’s hard to compare the full
lengths of the tables

	rx, cx := len(x)
	ry, cy := len(y)
	rx == ry && cx == cy

Additionally, it’s hard to get any one specific length.
Such an ability is useful in for loops, for example

	for i := 0; i < len(t)[0]; i++ {
	}

Lastly, this behavior does not extend well to higher dimensional tables.
For a 5-dimensional table,

	r1, r2, r3, r4, r5 := len([,,,,]int{})

is pretty silly. It would be much better to return a [5]int

### Copy
There are two cases in copy for tables.
The first is a table-table copy in which copy takes in two tables and returns a
[2]int specifying the number of elements that were copied in each dimension.

	n := copy(dst, src)   // n is a [2]int

Copy will copy all of the elements in the subtable from the first row to
min(len(dst)[0], len(src)[0]) and the first column to
min(len(dst)[1], len(src)[1]).

	dst := make([,]int, 6, 8)
	src := make([,]int, 5, 10)
	n := copy(dst, src) // n == [2]int{5, 8}
	fmt.Println("All destination elements were overwritten:", n == len(dst))

The second case is a copy between a single dimension of a table and either a
slice or another table single dimension.
The dimension of the table to copy will be specified with a single integer in
one dimension, and a two-index slice formation in the other dimension.
In this case, the number of elements copied is to the minimum length along the
copying dimension.
Copy will return an integer stating this number.
In the case of slice-table copy, the element-type must be the same.

	slice := []int{0, 0, 0, 0, 0}
	table := [,]int{{1,2,3} , {4,5,6}, {7,8,9}, {10,11,12}}
	copy(slice, table[1,:])    // Copies all the whole second row of the table
	fmt.Println(slice)  // prints [4 5 6 0 0]
	copy(table[1:, 2], slice[1:])  // copies elements 1:4 of the slice into the
	                               // 3rd table column
	fmt.Println(table)  // prints  [[1 2 3] [4 5 5] [7 8 6] [10 11 0]]
	copy(table[2,:], table[1,:]) // copies the second row into the third row
	fmt.Println(table)  // prints  [[1 2 3] [4 5 5] [4 5 5] [10 11 0]]

#### Discussion
The table-table copy seems like the only reasonable extension to copy for tables.

The single dimension copy is required because it is very common to want to
extract data from a specific row or column of a table.
It does increase the complexity of copy, as there needs to be an implementation
of runtime·copyn that handles strided copying (more than just memmove).
However, this implementation is already required for a table-table copy, and the
special case greatly improves the compatibility of slices and tables thus
meriting its inclusion.

### Range
Range allows for efficient iteration along a fixed location in a single
dimension of a table, for example the (elements of the) third column.
The expression list of the range statement may have one or two items that
represent the index and the table value along the fixed location respectively.
This fixed dimension is specified similarly to copy above -- one dimension of
the table has a single integer specifying the row or column to loop over, and
the other dimension has a two-index slice syntax.
Optionally, if there is only one element in the expression list, the fixed
integer may be omitted.
It is a compile-time error if the expression list has two elements but the fixed
integer is omitted.
It is also a compile-time error if the single integer is a negative constant,
and a runtime panic will occur if the fixed integer is out of bounds of the
table in that dimension.
To help with legibility, gofmt will format such that there is a space between
the comma and the bracket when the fixed index is omitted, i.e. [:, ] and [ ,:],
not [:,] and [,:].

#### Examples

	table := [,]int{{1,2,3} , {4,5,6}, {7,8,9}, {10,11,12}}
	for i, v := range table[2,:]{
		fmt.Println(i, v)  // i ranges from 0 to 2, v will be 7,8,9
		                   // (values of 3rd row)
	}
	for i, v := range table[:,0]{
		fmt.Println(i, v)  // i ranges from 0 to 3, v will be 1,4,7,10
		                   // (values of 1st column)
	}
	for i := range table[:, ]{
		fmt.Println(i) // i ranges from 0 to 3
	}
	for i, v := range table[:, ]{ // compile time error, no column specified
		fmt.Println(i)
	}
	// Sum the rows of the table
	rowsum := make([]int, len(table,1))
	for i = range table[:, ]{
		for _, v = range table[i,:]{
			rowsum[i] += v
		}
	}
	// Matrix-matrix multiply (given existing tables a and b)
	c := make([,]float64, len(a)[0], len(b)[1])
	for i := range a[:, ]{
		for k, va := range a[i,:] {
			for j, vb := range b[k,:] {
				c[i,j] += va * vb
			}
		}
	}

#### Discussion
This description of range mimics that of slices.
It permits a range clause with one or two values, where the type of the second
value is the same as that of the elements in the table.
This is far superior to ranging over the rows or columns themselves (rather than
the elements in a single row or column).
Such a proposal (where the value is []T instead of just T) would have O(n) run
time in the length of the table dimension and could create significant extra
garbage.

The option to omit a specific row or column is necessary to allow for nice
behavior when the length of the table is zero in the relevant dimension.
One may sum the elements in a slice as follows:

	sum := 0
	for _, v := range s {
		sum += v
	}

This works for any slice including nil and those with length 0.
With the ability to omit, similar code also works for nil tables and tables with
zero length in any or all of the dimensions:

	sum := 0
	for i := range t[:, ]{
		for j, v := range t[i,:]{
			sum += v
		}
	}

Were it mandatory to specify a particular column, one would have to replace
t[:,] with t[:,0] in the first range clause.
If len(t,0) == 0, this would panic.
It would thus be necessary to add in an extra length check before the range
statements to avoid such a panic.

There is an argument about the legibility of the syntax, however this should not
be a problem for most programmers.
There are four ways one may range over a table:

1. `for i := range t[:, ]`
2. `for i := range t[ ,:]`
3. `for i, v := range t[x,:]`
4. `for i, v := range t[:,x]`

These all seem reasonably distinct from one another.
With the gofmt space enforcement, It’s clear which side of the comma the colon
is on, and it’s clear whether or not a value is present.
Furthermore, anyone attempting to read code involving tables will need to know
which dimension is being looped over, and anyone debugging such code will
immediately check that the the correct dimension has been looped over.

Lastly, this range syntax is very robust to user error.
All of the following are compile errors:

	for i := range t            // Error: expected "[" when ranging over table
	for i, v := range t[:, ]    // Error: column unspecified when using two-
	                            // element syntax
	for i := range t[ , ]       // Error: no ranging dimension specified

Note in particular that while the omission is technically optional, an extra
omission is a compile-time error, and not omitting when one could will have no
effect on program behavior as long as the specified index is in-bounds.

Instead of the optional omission, using an underscore was also considered, for
example,

	for i := range t[:,_]

This has the benefit that the programmer must specify something on both sides of
the comma, and this usage matches the spirit of underscore in that "the specific
column doesn’t matter".
This was rejected for fear of overloading underscore, but remains a possibility
if such overloading is acceptable.
Similarly, "-" could be used, but is overloaded with subtraction.

A third option considered was to make the rule that there is only a runtime
panic when the table is accessed, even if the fixed integer is out of bounds.
This would avoid the zero length issue, `for i := range t[0,:]` never needs a
table element, and so would not panic.
However, this introduces its own issues. Does `for i, _ := range t[0,:]` panic?
The lack of consistency is very undesirable.

## Compatibility

This change is fully backward compatible with the Go1 spec.
It has also been designed to be compatible with a future extension to
n-dimensional tables.

## Implementation

A table can be implemented in Go with the following data structure

	type Table struct {
		Data       uintptr
		Stride     int
		Len        [2]int
		Cap        [2]int
	}

Access and assignment can be performed using the stride.
t[i,j] gets the element at i*stride + j in the array pointed to by the Data
uintptr.
When a new table is allocated, Stride is set to Cap[0] (for row major).

A table slice is as simple as updating the pointer, lengths, and capacities.

	t[a:b:c, d:e:f]

causes Data to update to a*Stride + d, Len[0] = b-a, Len[1] = e-d, Cap[0] = c-a,
Cap[1] = f-d, and Stride is unchanged.

### Reflect

Package reflect will add reflect.TableHeader (like reflect.SliceHeader and
reflect.StringHeader).

	type TableHeader struct {
		Data       uintptr
		Stride     int
		Len        [2]int
		Cap        [2]int
	}

The same caveats can be placed on TableHeader as the other Header types.
If it is possible to provide more guarantees, that would be great, as there
exists a large body of C libraries written for numerical work, with lots of time
spent in getting the codes to run efficiently.
Being able to call these codes directly is a huge benefit for doing numerical
work in Go (for example, the gonum and biogo matrix libraries have options for
calling a third party BLAS libraries for tuned matrix math implementations).
A new reflect.Kind will also need to be added, and many existing functions will
need to be modified to support the type.
Eventually, it will probably be desirable for reflect to add functions to
support the new type (MakeTable, TableOf, SliceTable, etc.).
The exact signatures of these methods can be decided upon at a later date.

Help is needed to determine the when and who for the implementation of this
proposal.
The gonum team would translate the code in gonum/matrix, gonum/blas,
and gonum/lapack to assist with testing the implementation.

### Non-goals

This proposal intentionally omits many behaviors put forth in Norman’s proposal.
This is not to say those proposals can’t ever be added (nor does it imply that
they will be added), but that they provide additional complications and, in my
opinion, do not pull their weight for an initial implementation.

### Full multi-dimensional slices
The most controversial part of this proposal is the decision to only add
two-dimensional tables to the language.
The reason for this is to minimize the immediate changes to the language and to
keep the initial implementation small.
While many of the language behaviors (allocation, slicing, length, copy, range,
etc.) extend quite nicely to higher dimensions, the implementation is quite a
bit more complex (Norman’s proposal used product notation for example), and the
language extensions needed in reflect are less straight-forward (the discussion
in Norman’s proposal about SliceHeader6D for example).
Two-dimensional tables are much more common in numerical work, and in my
experience, higher dimensional tables are mostly used for storing data rather
than manipulating it.
Accesses and assignments are not in the inner-loop of computation, thus nested
slices could suffice.

That said, this proposal was designed to allow extension to higher-dimensional
tables without backward-incompatible changes.
There are implementation choices which would need to be made, for example, would
there be one "multi-dimensional"  table type, or would each dimensional type
have its own internal representation (length represented as []int vs. [N]int for
example).
These choices would have to be made in a future proposal to extend tables to
higher dimensions.

### Down-slicing
Norman’s proposal suggested the allowance of "down-slicing", where one could
turn, say, a table into a slice with a single element slicing operation

	slice := table[1,2:10]   // creates a slice of length 8 using the elements
	                         // from row 1 of the table

In my opinion, this feature would be nice, but it either requires one of three
things

1. Asymmetry in spec (table[1,2:10] is allowed but not table[2:10,1])
2. Modification of the slice implementation to add a stride field
3. Have a "1-D table" type which is the same as a slice but has a stride

Option 3 is, in my opinion, harmful to the language.
It will cause a gap between the codes that support strided slices and the codes
that support non-strided slices, and may cause duplication of effort to support
both forms (2^N effort in the worst case).
On the whole, it will make the language less cohesive.
Option 1 is a possibility, as it reflects the underlying row-major
implementation of the data (required by the spec).
That said, it could be confusing for users of the language.
Option 2 would be a significant change to the language, adding memory and cost
to a fundamental data structure.
Such a change would require more commentary and analysis from the community at
large.
In any event, nothing in the proposal prevents any of these options, and it is
better to avoid making a change than to add something that must be kept forever.
This decision can be put off until a later discussion.

### Append
Like Norman’s proposal, this proposal does not allow append to be used with
tables. This is mainly a matter of syntax. Can you append a table to a table or
just add a row/column at a time. What would the syntax be? Again, better to leave
for later.

### Arithmetic Operators
Some have called for tables to support arithmetic operators (+, -, *) to also
work on [,]Numeric (int, float64, etc.), for example

	a := make([,], 1000, 3000)
	b := make([,], 3000, 2000)
	c := a*b

While operators can allow for very succinct code, they do not seem to fit in Go.
Go’s arithmetic operators only work on numeric types, they don’t work on slices.
Secondly, arithmetic operators in Go are all fast, whereas the operation above
is many more orders of magnitude more expensive than a floating point multiply.
Finally, multiplication could either mean element-wise multiplication, or
standard matrix multiplication.
Both operations are needed in numerical work, so such a proposal would require
additional operators to be added (such as .*).
Especially in terms of clock cycles per character, `c.Mul(a,b)` is not that bad.

## Conclusion
Matrices are widely used in numerical algorithms, and have been used in
computing almost as long as there have been computers.
With time and effort, Go could be a great language for numerical computing (for
all of the same reasons it is a great general-purpose language), but first it
needs a rectangular data structure, a "table", built into the language as a
foundation for more advanced libraries.
This proposal describes a behavior for tables which is a strict improvement over
the options currently available.
It will be faster than the single-slice representation (index optimization and
range), more convenient than the slice of slice representation (range, copy,
len), and will provide a correct representation of the data that is more compile-
time verifiable than the struct representation.
The desire for tables is not driven by syntax and ease-of-use, though that is a
huge benefit, but instead a request for safety and speed; the desire to build
"simple, reliable, and efficient software".

|                | Correct Representation | Access/Assignment Convenience | Speed |
| -------------: | :--------------------: | :---------------------------: | :---: |
| Slice of slice | X                      | ✓                             | X     |
| Single slice   | X                      | X                             | ✓     |
| Struct type    | ✓                      | X                             | X     |
| Built-in       | ✓                      | ✓                             | ✓     |

## Open issues
Since this was originally proposed, a few issues have come to light.

1. It seems Go would want to add tables of any size.
The proposal was designed with this as a future outcome and can easily be
modified.
There are no significant changes in the proposal in the higher-dimensional
extension.
2. Given the desire for 'all-at-once' changes rather than incremental changes
(which was the belief at the time of writing), it's worth thinking about
down-slicing.
This would likely be option 1 in the non-goals.
3. In the discussion, it was mentioned that adding a TableHeader is a bad idea.
This can be removed from the proposal, but some other mechanism should be added
that allows data in tables to be passed to C.
