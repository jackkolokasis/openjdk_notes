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

First we introduce the ConstantPool class


