ndArray (beta)
========

ndArray is a single-header template C++11 library that defines a **generic n-dimensional array class** capable of handling both Matlab's `mxArray`s (mutable or not) and standard C++ allocations (both static and dynamic). It is designed to provide minimal (but useful) functionality and can be easily extended/built-upon for any kind of application.

---

## Documentation

The library implements a template n-dimensional array class `ndArray`, focusing mainly on **four core features**:

- Flexible memory management, either assuming ownership (ie dealing with deallocation) or acting as a passive handle on a given allocation;
- Handle shallow and deep copies;
- Provide natural (and efficient) multi-dimensional accessors to the contents of the array;
- Provide detailed access to the dimensions of the array.

In the following, we describe the interface provided by this class and give simple usage examples. Note that the file `test_ndArray` contains additional examples of usage.

### The `ndArray` template class

The `ndArray` template class requires **two** template parameters:

1. `typename T`: the value-type (eg. `double`, `char`, or any user-defined/STL class). This type can be `const`-qualified to prevent underlying elements from being mutated (useful with Matlab inputs for instance, cf examples).
2. `dimen_t N`: the number of dimensions of the array, any number greater than 1.

> NOTE:
>
> The type `dimen_t` is currently set to `uint8_t`, do the number of dimensions should be less than 128.
> On the other hand, the index type `index_t` is either set to match Matlab's `mwIndex` type, or `uint64_t` if `ND_ARRAY_USING_MATLAB` is not set.
>
> Both can easily be changed manually in the header `ndArray.h`.


Once the dimensionality `N` of an array is set, it cannot be modified, so you'll have to declare several arrays to deal with 2D and 3D matrices for example. The members of `ndArray<T,N>` are:
- `index_t m_numel`: the number of elements stored in the array;
- `index_t m_size[N]`: the dimensions of the array (similar to Matlab's `size()` function);
- `index_t m_strides[N]`: the index-offsets in each dimension, typcially `[1 cumprod(size(1:N-1))]`;
- `std::shared_ptr<T>`: the actual data, stored as a plain 1D array of length `m_numel`.

Note that the indexing convention is similar to that of Matlab: the second element in `m_data` corresponds to the coordinates `(2,1,...)` (ie **row-first/column-wise**).

#### Deactivating Matlab support

Simply comment the definition of the flag `ND_ARRAY_USING_MATLAB`.

#### Handling memory

The [STL implementation](http://www.cplusplus.com/reference/memory/shared_ptr/) of shared pointers allows to specify a deleter. This deleter is used when the last referring instance gets destroyed in order to release the memory allocation. We provide a dummy class `no_delete<T>` that acts as a "fake-deleter"; the allocation will not be modified upon destruction of the last shared pointer, which allows to handle static allocations or externally managed memory (like Matlab's) for example.

There are **four possible ways** to assign the contents of a `ndArray`:

1. `assign( const mxArray* )`: with either a Matlab input or output. The memory allocation corresponding to a Matlab matrix **will not  be managed** by the shared pointer. Note that if the `mxArray` has the wrong number of dimensions or wrong type, an exception will be raised.
2. `assign( pointer, const index_t*, bool )`: with a pointer `T*`, a size array and a boolean flag that specifies whether or not the corresponding memory should be managed. If true, the default deleter `std::default_delete` will be passed to the shared pointer (this releases memory allocated using `new` **only**, do NOT use `malloc` variants). If false, `no_delete` will be passed instead, which will prevent from deallocating static or externally managed memory.
3. `operator= ( const ndArray<T,N>& )`: the assignment operator copies the contents of another instance of same type and with same number of dimensions. Only a reference to the data is copied, not the data itself (**shallow copy**). The copy constructor uses this implementation, so you can safely return matrices after instanciating them inside a function without triggering a deep-copy of the underlying data (although it is always preferable to use input references in my opinion).
4. `copy( const ndArray<U,N>& )`: this performs a **deep copy** of an array with same dimensionality and possibly different type. A new memory allocation is requested (and therefore managed) to store the copies if the number of elements in the current instance (`m_numel`) is different from that of the input to be copied. Note that deep copies are _not possible_ if the value-type `T` is const-qualified, an exception will be raised in this case.

Additionally, two arrays _of same type_ can be swapped without copy, using the method `swap( ndArray<T,N>& other )`.

#### Accessing the data

The actual data can be accessed in **four possible ways** again:

1. `operator[] (index_t)`: access the matrix as a 1D array, with column-wise convention;
2. `operator() ( const index_t* )`: access the matrix with N-dimensional coordinates stored in an array (ie `nd::index_t size[5] = {1,4,2,3,3};` for a 5D array);
3. `operator() ( std::initializer_list<index_t> )`: idem, but with an initializer list (ie passing directly `{1,4,2,3,3}` as an input);
4. `operator() ( index_t, ... )`: access with N coordinate inputs (eg `M(1,4,2,3,3)`).

Note that these methods are _unsafe_, and will segfault if:

- The underlying array has not been assigned;
- The coordinates in methods 2 or 3 are not of length `N` (the third method will throw an exception in safe mode);
- Not enough inputs are given in method 4;
- Indexes are out of bounds in unsafe mode.

The flag `ND_ARRAY_SAFE_ACCESS` can be commented in the sources to increase speed slightly (although this probably won't be the bottleneck of your program).

#### Iterating over the data

The underlying data can be iterated using the methods `data`, `begin` and `end`. Specifically:

- `data()` returns a pointer to the first element of the array;
- `begin()` as well, it is provided to match the standard in terms of iteration (random access iterator);
- `end()` similarly returns a pointer to the element _past-the-end_ of the array.

If the value-type is const qualified, then all the above methods return pointers to constant values. Alternatively for non-const arrays, explicitly constant variants are implemented as well (`cdata()`, `cbegin()` and `cend()`).

#### Accessing dimensions

There are several methods to access the dimensions of the array:

- `dimen_t ndims()`: get the number of dimensions;
- `index_t numel()`: get the number of elements in the array;
- `const index_t* size()`: access the size array;
- `index_t size( dimen_t k )`: access the k-th dimension;
- similar methods to `size` for `strides` (index-offsets for each dimension).

### Other useful implementations

The library comes with two other implementations that might be useful to you, even though they have nothing to do with N-dimensional arrays.

#### `mx_type`

`mx_type` is a template class that associates numeric C types with the corresponding Matlab name and class ID. A typical usage is as follows: `mx_type<float>::name` will give you "single" and `mx_type<float>::id` will give you `mxSINGLE_CLASS`. This is used for example to make sure that the type of an input `mxArray` corresponds to the value-type of the `ndArray` (cf examples).

#### `singleton`

A very simple singleton construct is provided, simply use `singleton<T>::instance` at any time in your code.

---

## Examples

Two main examples are provided (both can be compiled and run independently in the Matlab console), in addition to the test file. As the library is quite small, and the code entirely commented, these examples are only meant to give you a feel at what your program should look like if you should decide to use this library in your application.

### Handling an existing allocation

```c++
#include "mexArray.h"
#define MASSERT( cdt, msg ) { if(!cdt) {mexErrMsgTxt(msg);return;} }

using namespace nd;

void mexFunction(	int nargout, mxArray *out[],
					int nargin, const mxArray *in[] )
{

	//////////////////////////////////////////////
	// Get a double matrix of size 3x4 in input //
	//////////////////////////////////////////////

	// Instanciate a const double matrix
	// NOTE: how we use a const type with the Matlab input
	ndArray<const double,2> A;

	// Get Matlab's first input
	// NOTE: an exception will be raised if the input is not two-dimensional (matrix)
	//       or if the value type is not double.
	MASSERT( nargin > 0, "One input expected." );
	A.assign( in[0] );

	// Make sure the dimensions are correct
	MASSERT( A.size(0) == 3 && A.size(1) == 4, "Input required to be a 3x4 matrix." );

	// ------------------------------------------------------------------------

	/////////////////////////////////////////////////
	// Handle a 5x20x3 matrix statically allocated //
	/////////////////////////////////////////////////

	// Create static allocation
	unsigned static_alloc[ 5*20*3 ];

	// Create ndArray handler
	const index_t size[3] = { 5, 20, 3 };
	ndArray<unsigned,3> B( static_alloc, size, false );
		// false because we don't want to manage static memory;
		// any request to deallocate static_alloc would fail.

	// Test an assignment
	// NOTE: we're using the 4th way to access data here (cf documentation)
	B( 3,11,0 ) = 42;
}
```

### Creating a new array

```c++
#include "mexArray.h"

using namespace nd;

void mexFunction(	int nargout, mxArray *out[],
					int nargin, const mxArray *in[] )
{
	///////////////////////////////////////////////////
	// Create a 4D matrix using a dynamic allocation //
	///////////////////////////////////////////////////

	// Allocate new 4D matrix
	const index_t size[4] = {3,4,5,6};
	const index_t numel   = 3*4*5*6;
	ndArray<short,4> A( new short[numel], size, true );
		// true because this is a dynamic allocation, so it needs
		// to be deleted at some point. The last refering instance
		// will take care of deleting this memory automatically.

	// Do something with it
	for ( index_t i1 = 0; i1 < size[0]; ++i1 )
	for ( index_t i2 = 0; i2 < size[1]; ++i2 )
	for ( index_t i3 = 0; i3 < size[2]; ++i3 )
	for ( index_t i4 = 0; i4 < size[3]; ++i4 )
	{
		A( {i1,i2,i3,i4} ) = static_cast<short>( (i1 + 2*i2 + 3*i3) % (i4 + 1) );
	}

	// ------------------------------------------------------------------------

	//////////////////////////////////////////////////////////////
	// Create a Matlab output and copy the previous array to it //
	//////////////////////////////////////////////////////////////

	/*

	IMPORTANT NOTE: this is just an example to illustrate the fact that deep-copies on
	arrays of SAME SIZE do not require a new allocation. We could perform the exact same
	treatment than the previous for-loops directly on the following Matlab output...
		... and it would be much more efficient to do so!!

	*/

	// Crate a new 4-dimensional Matlab output
	// NOTE: look how mx_type can be used here
	out[0] = mxCreateNumericArray( 4, size, mx_type<short>::id, mxREAL );

	// Assign output to new ndArray
	// NOTE: if we want to assign the values of the output, the value type should NOT be const!
	ndArray<short,4> B( out[0] );

	// Copy the values of A
	// NOTE: because B was created with the same number of elements,
	// this will not incur any memory reallocation
	B.copy( A );
}
```
