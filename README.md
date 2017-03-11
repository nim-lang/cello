# Briefly

Briefly is a library of succinct data structures, oriented in particular
for string searching and string operations.

## Operations

The most common operations that we implement on various kind of sequence data
are `rank` and `select`.

For bit sequences, `rank(i)` counts the number of 1 bits in the first `i`
places. The number of 0 bits can easily be obtained as `i - rank(i)`. Viceversa,
`select(i)` finds the position of the `i`-th 1 bit in the sequence. In this
case, there is not an obvious relation to the position of the `i`-th 0 bit,
so we provide a similar operation `select0(i)`.

To ensure that `rank(select(i)) == i`, we define `select(i)` to be 1-based,
that is, we count bits starting from 1.

As a reference, we implement `rank` and `select` on Nim built-in sets, so
that for instance the following is valid:

```nim
let x = { 13..27, 35..80 }

echo x.rank(16)  # 3
echo x.select(3) # 16
```

More generally, one can define 'rank' and `select` for sequence of symbols
taken from a finite alphabet, relative to a certain symbol. Here, `rank(c, i)`
is the number of symbols equal to `c` among the first `i` symbols, and
`select(c, i)` is the position of the `i`-th symbol `c` in the sequence.

Again, we give a reference implementation for strings, so that the following
is valid:

```nim
let x = "ABRACADABRA"

echo x.rank('A', 8)   # 4
echo x.select('A', 4) # 8
```

Notice that in both cases, the implementation of `rank` and `select` is a
naive implementation which takes `O(i)` operations. More sophisticated data
structures allow to perform similar operations in constant (for rank) or
logarithmic (for select) time, by using indices. *Succinct* data structures
allow to do this using indices that take at most `o(n)` space in addition
to the sequence data itself, where `n` is the sequence length.

## Data structures

We now describe the succinct data structures that will generalize the bitset
and the string examples above. In doing so, we also need a few intermediate
data structures that may be of independent interest.

### Bit arrays

Bit arrays are a generalization of Nim default `set` collections. They can
be seen as an ordered sequence of `bool`, which are actually backed by a
`seq[int]`. We implement random access - both read and write - as well as
naive `rank` and `select`. An example follows:

```nim
var x = bits(13..27, 35..80)

echo x[12]   # false
echo x[13]   # true
x[12] = true # or incl(x, 12)
echo x[12]   # true
x[12] = false

echo x.rank(16)    # 3
echo x.select(3)   # 16
echo x.select0(30) # 90
```

### Int arrays

Int arrays are just integer sequences of fixed length. What distinguishes
them by the various types `seq[int64]`, `seq[int32]`, `seq[int16]`, `seq[int8]`
is that the integers can have any length, such as 23.

They are backed by a bit array, and can be used to store many integer numbers
of which an upper bound is known without wasting space. For instance, a sequence
of positive numbers less that 512 can be backed by an int array where each
number has size 9. Using a `seq[int16]` would almost double the space
consumption.

Most sequence operations are available, but they cannot go after the initial
capacity. Here is an example:

```nim
var x = ints(200, 13) # 200 ints at most 2^13 - 1

x.add(123)
x.add(218)
x.add(651)
echo x[2]   # 651
x[12] = 1234
echo x[12]   # 1234

echo x.len       # 13
echo x.capacity  # 200
```

### RRR

The [RRR](http://alexbowe.com/rrr/) bit vector is the first of our collections
that is actually succinct. It consists of a bit arrays, plus two int arrays
that stores `rank(i)` values for various `i`, at different scales.

It can be created after a bit array, and allows constant time `rank` and
logarithmic time `select` and `select0`.

```nim
let b: BitArray = ...
let r = rrr(b)

echo r.rank(123456)
echo r.select(123456)
echo r.select0(123456)
```

To convince oneself that the structure really is succinct, `stats(rrr)` returns
a data structures that shows the space taken (in bits) by the bit array, as
well as the two auxiliary indices.

### Wavelet tree

The [wavelet tree](http://alexbowe.com/wavelet-trees/) is a tree constructed
in the following way. An input string over a finite alphabet is given. The
alphabet is split in two parts - the left and the right one, call them L and R.

For each character of the string, we use a 1 bit to denote that the character
belongs to R and a 0 bit to denote that it belongs to L. In this way, we
obtain a bit sequence. The node stores the bit sequence as an RRR structures,
and has two children: the one to the left is the wavelet tree associated to
the substring composed by the characters in L, taken in order, and similarly
for the right child.

This structure allows to compute `rank(c, i)`, where `c` is a character in the
alphabet, in time `O(log(l))`, and `select(c, i)` in time `O(log(l)log(n))`
where `l` is the size of the alphabet and `n` is the size of the string.
It also allows `O(log(l))` random access to read elements of the string.

It can be used as follows:

```nim
let
  x = "ACGGTACTACGAGAGTAGCAGTTTAGCGTAGCATGCTAGCG"
  w = waveletTree(x)

echo x.rank('A', 20)   # 7
echo x.select('A', 7)  # 20
echo x[12]             # 'G'
```

### Rotated strings

The next ingredient that we need it the Burrows-Wheeler transform of a string.
It can be implemented using string rotations, so that's what we implement
first. It turns out that this implementation is too slow for our purposes,
but rotated strings may be useful anyway, so we left them in.

A rotated strings is just a view over a string, rotated by a certain amount
and wrapping around the end of the string. If the underlying string is a `var`,
our implementation reuses that memory (which is then shared) to avoid the
copy of the string. We just implement random access and printing:

```nim
var
  s = "The quick brown fox jumps around the lazy dog"
  t = s.rotate(20)

echo t[10] # n
echo t[20] # u

t[18] = e

echo s # The quick brown fox jumps around the lezy dog
echo t # jumps around the lezy dogThe quick brown fox
```

### Suffix array

The suffix array of a string is a permutation of the numbers from 0 up to the
string length excluded. The permutation is obtained by considering, for each
`i`, the string rotated by `i`, and sorting these strings in lexicographical
order. The resulting order is the suffix array.

Here the suffix array is represented as an IntArray. It can be obtained as
follows:

```nim
let
  x = "this is a test."
  y = suffixArray(x)

echo y # @[7, 4, 9, 14, 8, 11, 1, 5, 2, 6, 3, 12, 13, 10, 0]
```

### Burrows-Wheeler transform

The [Burrows-Wheeler transform](http://michael.dipperstein.com/bwt/) of a string
is a string having the same length, together with a distinguished index.
Once one has a suffix array `sa` for the string `s`, the Burrows-Wheeler
transform is the string which at the index `i` has the last character of the
rotation of `s` by `sa[i]`. The distinguished index if the permutation of `0`.

We recall the following two facts:

* the Burrows-Wheeler transform can be inverted - the exact algorithm is
  outside the purposes of this documentation
* whenever a character is a good predictor for the next one (in the original
  string), the string in the Burrows-Wheeler transform tends to have many
  repeated characters, which allows to compress it by run-length encoding.

An example of usage is this:

```nim
let
  s = "The quick brown fox jumps around the lazy dog"
  (t, i) = burrowsWheeler(s)
  u = inverseBurrowsWheeler(t, i)

echo t # skynxeedg l in hh otTu c uwudrrfm abp qjoooza
echo u # The quick brown fox jumps around the lazy dog
```

### FM indices

An [FM index](http://alexbowe.com/fm-index/) for a string puts together
essentially all the pieces that we have described so far. The index itself
holds a walevet tree for the Burrows-Wheeler transform of the string, together
with a small auxiliary table having the size of the string alphabet.

It can be used for various purposes, but the simplest one is backward search.
Given a pattern `p` (a small string) and possibly long string `s`, there is a
way to search all occurrences of `p` in time `O(L)`, where `L` is the length
of `p` - the time is independent of `s` - using an FM index for `s`.

Every occurrence of `p` appears as the prefix of some rotation of `s` - hence
all such occurrences correspond to consecutive positions into the suffix
array for `s`. The first and last such positions can be found as follows:

```nim
let
  x = "mississippi"
  pattern = "iss"
  fm = fmIndex(x)
  positions = fm.search(pattern)

echo positions.first # 2
echo positions.last  # 3

for j in positions.first .. positions.last:
  let i = sa[j]
  echo x.rotate(i)

# issippimiss
# ississippim

```