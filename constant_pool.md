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
		Array<u1>*		_tags;		// The tag array describing the constant pool's contents
		ConstantPoolCache*	_cache; 	// The cache holding interpreter runtime information
		InstanceKlass*		_pool_holder	// The corresponding class
		Array<u2>*		_operands	// For variabled-sized (InvokeDynamic ) nodes - usually is empty

		// Array of resolved objects from the constant pool and map
		// from resolved object index to original constant pool index
		jobject			_resolved_references; // jobject is a pointer type
		Array<u2>*		_reference_map;

		int			_flags;		// Old fashioned bit twiddling
		int                     _length;        // number of elements in the array
 
		union {
			// set for CDS to restore resolved references
			int             _resolved_reference_length;
			// keeps version number for redefined classes (used in backtrace)
			int             _version;
		} _saved;

		Monitor*            	_lock;
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

The constant pool contains the following information:
  
```
                                                                   +-------------+
                                                               +---| Text String |               
                                                               |   +-------------+
                                                               |   +---------------------------------+
                                                               |---| A constant declared as final    |
                                 +-------------------+         |   +---------------------------------+
                             +---| Font amount       |---------+   +---------------------------------+
+--------------------+       |   +-------------------+         |---| The value of the basic data type|
| What a constant    |       |                                 |   +---------------------------------+
| pool item          |-------+                                 |   +------------------------------+
| contains           |       |                                 +---| Other                        |
|                    |       |                                     +------------------------------+
|                    |       |                                     +------------------------------------------------+
+--------------------+       |   +-------------------+         +---| Fully qualified names for classes and stuctures|                
                             +---| Symbol References |---------|   +------------------------------------------------+
							     +-------------------+         |   +----------------------------+
                                                               |---| Field names and descriptors|
                                                               |   +----------------------------+
                                                               +   +-----------------------------+
                                                               |---| Method names and descriptors|
                                                                   +-----------------------------+
```

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
+---------------------+      Metadata
| _valid              |
+---------------------+      ----------------------
| _tags               |
+---------------------+
| _cache              |
+---------------------+
| _pool_holder        |
+---------------------+
| _operands           |
+---------------------+
| _resolved_references|	      Constant Pool
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
The _valid is an int type defind in Metadata and it is only available in
debug version.  In case that we run a product version then there is no
such attribute and the metadata only occupies 8 bytes.

Heap memory allocation is performed by overloading new operator. The
overloading of the new operator is defined in the MetaspaceOjb
(ConstantPool indirectly inherits this class) class as follow:

```

void* MetaspaceObj::operator new(size_t size, ClassLoaderData* loader_data, 
		                         size_t word_size, 
								 bool read_only, MetaspaceObj::Type type, TRAPS) throw() {

	// Klass has it's own operator new
	return Metaspace::allocate(loader_data, word_size, read_only, type, CHECK_NULL);
}

```
The called method Metaspace::allocate() memory in the heap. This
method will be described in detail when introducing garbage
collection. Here you only need to know that this method will allocate
size memory in the heap and clear the memory.

Call the ConstantPool constructor to initialize some properties as
follow:

```

ConstantPool::ConstantPool(Array<u1>* tags) {
	set_length(tags->length());
	set_tags(NULL);
	set_cache(NULL);
	set_reference_map(NULL);
	set_resolved_references(NULL);
	set_operands(NULL);
	set_pool_holder(NULL);
	set_flags(0);

	// only set to non-zero if constant pool is merged by RedefineClasses
	set_version(0);
	set_lock(new Monitor(Monitor::nonleaf + 2, "A constant pool lock"));

	// initialize tag array
	int length = tags->length();
	for (int index = 0; index < length; index++) {
		tags->at_put(index, JVM_CONSTANT_Invalid);
	}
	set_tags(tags);
}

```

You can see the initialization of attributes such as tags, _length,
and _lock. The JVM_CONSTANT_Invalid value is stored in the tags
array, which will be updated to the value defined in the following
enumeration class when analyzing the specific constant pool item:

```
enum {
    JVM_CONSTANT_Utf8 = 1,      // 1
    JVM_CONSTANT_Unicode,       // 2      /* unused */
    JVM_CONSTANT_Integer,       // 3
    JVM_CONSTANT_Float,         // 4
    JVM_CONSTANT_Long,          // 5
    JVM_CONSTANT_Double,        // 6
    JVM_CONSTANT_Class,         // 7
    JVM_CONSTANT_String,        // 8
    JVM_CONSTANT_Fieldref,      // 9
    JVM_CONSTANT_Methodref,     // 10
    JVM_CONSTANT_InterfaceMethodref,   // 11
    JVM_CONSTANT_NameAndType,          // 12
    JVM_CONSTANT_MethodHandle           = 15,  // JSR 292
    JVM_CONSTANT_MethodType             = 16,  // JSR 292
    //JVM_CONSTANT_(unused)             = 17,  // JSR 292 early drafts only
    JVM_CONSTANT_InvokeDynamic          = 18,  // JSR 292
    JVM_CONSTANT_ExternalMax            = 18   // Last tag found in classfiles
};

```
This is the tag value in the constant pool item, but the first item in
the constant pool is still JVM_CONSTANT_Invalid.

The following describes the format specified by the virtual machine
specification:

```
CONSTANT_Utf8_info {
	u1 tag;
	u2 length;
	u1 bytes[length];
}

CONSTANT_Integer_info {
	u1 tag;
	u4 bytes;
}

CONSTANT_Float_info {
	u1 tag;
	u4 bytes;
}

CONSTANT_Long_info {
	u1 tag;
	u4 high_bytes;
	u4 low_bytes;
}

CONSTANT_Double_info {
	u1 tag;
	u4 high_bytes;
	u4 low_bytes;
}

CONSTANT_Class_info {
	u1 tag;
	u2 name_index;
}


CONSTANT_String_info {
	u1 tag;
	u2 string_index;
}

CONSTANT_Fieldref_info {
	u1 tag;
	u2 class_index;
	u2 name_and_type_index;
}

CONSTANT_Methodref_info {
	u1 tag;
	u2 class_index;
	u2 name_and_type_index;
}

CONSTANT_InterfaceMethodref_info {
	u1 tag;
	u2 class_index;
	u2 name_and_type_index;
}

CONSTANT_NameAndType_info {
	u1 tag;
	u2 name_index;
	u2 descriptor_index;
}


CONSTANT_MethodHandle_info {
	u1 tag;
	u1 reference_kind;
	u2 reference_index;
}

CONSTANT_MethodType_info {
	u1 tag;
	u2 descriptor_index;
}

CONSTANT_InvokeDynamic_info {
	u1 tag;
	u2 bootstrap_method_attr_index;
	u2 name_and_type_index;
}
```

In the constant pool parsing process, after the constant pool item is
determined by the index, the tag will be placed in the _tag array in
the ConstantPool class.
