# Constant Pool Analysis

When calling the ClassFileParser::parseClassFile() method to interpret
the class file, it will call the
ClassFileParser::parse_constant_pool() method to interpret the
constant pool. The calling statement is as follow:

```
constantPoolHandle cp = parse_constant_pool(CHECK_(nullHandle))

```

The method parse_constant_pool() is implemented as follow:

```
constantPoolHandle ClassFileParser::parse_constant_pool(TRAPS) {
	ClassFileStream* cfs = stream();
	constantPoolHandle nullHandle;

	u2 length = cfs->get_u2_fast();
	ConstantPool* constant_pool = ConstantPool::allocate(_loader_data, length,
			CHECK_(nullHandle));
	_cp = constant_pool; // save in case of errors
	constantPoolHandle cp (THREAD, constant_pool);
	// ...
	// parsing constant pool entries
	parse_constant_pool_entries(length, CHECK_(nullHandle));
	return cp;
}
```

ConstantPool::allocate() is called to create a ConstantPool object,
and then parse_constant_pool_entries() is called to parse the
items in the constant pool and save these items to the
ConstantPool object.

First we introduce the ConstantPool class, the specific constant pool
of the object of this class, which holds the constant pool meta
information.

## 1. ConstantPool class
The definition of the class is as follows:

```
class ConstantPool: public Metadata {
	private:
		Array<u1>*			_tags;			// The tag array describing the constant pool's contents
		ConstantPoolCache*	_cache; 		// The cache holding interpreter runtime information
		InstanceKlass*		_pool_holder	// The corresponding class
		Array<u2>*			_operands		// For variabled-sized (InvokeDynamic ) nodes - usually is empty

		// Array of resolved objects from the constant pool and map
		// from resolved object index to original constant pool index
		jobject				_resolved_references; // jobject is a pointer type
		Array<u2>*			_reference_map;

		int					_flags;			// Old fashioned bit twiddling
		int                 _length;        // number of elements in the array
 
		union {
			// set for CDS to restore resolved references
			int             _resolved_reference_length;
			// keeps version number for redefined classes (used in backtrace)
			int             _version;
		} _saved;

		Monitor*            _lock;
		...
}
```

The class represents constant pool metadata information, so for this
reason inherits the class _Metadata. 
_tags represent the contents in the constant pool. 
_length represents the total number of items in the
constant pool. The length of the _tags array is also the the _length.
cache assists the interpretation operation to save some information,
which will be introduced when introducing the interpretation
operation. 
The other attributes will not be introduced too much for the time
being.


## 2. Create ConstantPool Instance
In the method ClassFileParser::parse_constant_pool() that parses the
constant poil, the mehtod ConstantPool::allocate() is first called to
create a ConstantPool instance. The method is implemented as follow:

```
ConstantPool* ConstantPool::allocate(ClassLoaderData* loader_data, 
									 int length, TRAPS) {
	// Tags are RW but comment below applies to tags also.

	Array<u1>* tags = MetadataFactory::new_writable_array<u1>(loader_data, lenght, 0, CHECK_NULL);

	int size = ConstantPool::size(lenght);
	
	// CDS considerations:
	// Allocate read-write but may be able to move to read-only at dumping time
	// if all the klasses are resolved.  The only other field that is writable is
	// the resolved_references array, which is recreated at startup time.
	// But that could be moved to InstanceKlass (although a pain to access from
	// assembly code).  Maybe it could be moved to the cpCache which is RW.

	return new (loader_data, size, false, MetaspaceObj::ConstantPoolType, THREAD) 
		ConstantPool(tags);
}
```

The parameter length represents the number of constant pool items.  To
create a ContantPool object, we call the ConstantPool::size() to
calculate the size of the allocated memory.  The implementation of the
size() method is as follow:

```
static int size(int length) {
	int s = header_size();
	return align_object_size(s + length);
}

// Header size is in words
static int header_size() {
	int num = sizeof(ConstatnPool);
	return num/HeapWordSize;
}
```

According to the method implementation, it is the memory size occupied
by the ConstantPool instance itself plus length pointer length.
The final memory layout of the ConstantPool object is shown below:

```
+---------------------+
| vfptr               |
+---------------------+		 Metadata
| _valid			  |
+---------------------+      ----------------------
| _tags               |
+---------------------+
| _cache              |
+---------------------+
| _pool_holder        |
+---------------------+
| _operands           |
+---------------------+
| _resolved_references|		 Constant Pool
+---------------------+
| _reference_map      |
+---------------------+
| _flags              |
+---------------------+
| _length             |
+---------------------+
| _saved              |
+---------------------+
| _lock               |
+---------------------+      -----------------------

```
