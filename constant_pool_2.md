# Constant Pool Analysis (2)

The method parse_constant_pool() calls the
parse_constant_pool_entries() to parse each item in the constant pool.
The method is implemented as follow:


```
void ClassFileParser::parse_constant_pool_entries(int length, TRAPS) {
	// Use a local copy of ClassFileStream. It helps the C++ compiler to optimize
	// this function (_current can be allocated in a register, with scalar
	// replacement of aggregates). The _current pointer is copied back to
	// stream() when this function returns. DON'T call another method within
	// this method that uses stream().
	ClassFileStream*  cfs0 = stream();
	ClassFileStream   cfs1 = *cfs0;
	ClassFileStream*  cfs  = &cfs1;

	Handle class_loader(THREAD, _loader_data->class_loader());

	// Used for batching symbol allocations.
	const char*   names[SymbolTable::symbol_alloc_batch_size];
	int           lengths[SymbolTable::symbol_alloc_batch_size];
	int           indices[SymbolTable::symbol_alloc_batch_size];
	unsigned int  hashValues[SymbolTable::symbol_alloc_batch_size];
	int           names_count = 0;

	// parsing  Index 0 is unused
	for (int index = 1; index < length; index++) {
		// Each of the following case guarantees one more byte in the stream
		// for the following tag or the access_flags following constant pool,
		// so we don't need bounds-check for reading tag.
		u1 tag = cfs->get_u1_fast();
		switch (tag) {
			case JVM_CONSTANT_Class :
				{
					cfs->guarantee_more(3, CHECK);  // name_index, tag/access_flags
					u2 name_index = cfs->get_u2_fast();
					_cp->klass_index_at_put(index, name_index);
				}
				break;
			case JVM_CONSTANT_Fieldref :
				{
					cfs->guarantee_more(5, CHECK);  // class_index, name_and_type_index, tag/access_flags
					u2 class_index = cfs->get_u2_fast();
					u2 name_and_type_index = cfs->get_u2_fast();
					_cp->field_at_put(index, class_index, name_and_type_index);
				}
				break;
			case JVM_CONSTANT_Methodref :
				{
					cfs->guarantee_more(5, CHECK);  // class_index, name_and_type_index, tag/access_flags
					u2 class_index = cfs->get_u2_fast();
					u2 name_and_type_index = cfs->get_u2_fast();
					_cp->method_at_put(index, class_index, name_and_type_index);
				}
				break;
			case JVM_CONSTANT_InterfaceMethodref :
				{
					cfs->guarantee_more(5, CHECK);  // class_index, name_and_type_index, tag/access_flags
					u2 class_index = cfs->get_u2_fast();
					u2 name_and_type_index = cfs->get_u2_fast();
					_cp->interface_method_at_put(index, class_index, name_and_type_index);
				}
				break;
			case JVM_CONSTANT_String :
				{
					cfs->guarantee_more(3, CHECK);  // string_index, tag/access_flags
					u2 string_index = cfs->get_u2_fast();
					_cp->string_index_at_put(index, string_index);
				}
				break;
			case JVM_CONSTANT_MethodHandle :
			case JVM_CONSTANT_MethodType :
				if (tag == JVM_CONSTANT_MethodHandle) {
					cfs->guarantee_more(4, CHECK);  // ref_kind, method_index, tag/access_flags
					u1 ref_kind = cfs->get_u1_fast();
					u2 method_index = cfs->get_u2_fast();
					_cp->method_handle_index_at_put(index, ref_kind, method_index);
				} else if (tag == JVM_CONSTANT_MethodType) {
					cfs->guarantee_more(3, CHECK);  // signature_index, tag/access_flags
					u2 signature_index = cfs->get_u2_fast();
					_cp->method_type_index_at_put(index, signature_index);
				} else {
					ShouldNotReachHere();
				}
				break;
			case JVM_CONSTANT_InvokeDynamic :
				{
					cfs->guarantee_more(5, CHECK);  // bsm_index, nt, tag/access_flags
					u2 bootstrap_specifier_index = cfs->get_u2_fast();
					u2 name_and_type_index = cfs->get_u2_fast();
					if (_max_bootstrap_specifier_index < (int) bootstrap_specifier_index)
						_max_bootstrap_specifier_index = (int) bootstrap_specifier_index;  // collect for later
					_cp->invoke_dynamic_at_put(index, bootstrap_specifier_index, name_and_type_index);
				}
				break;
			case JVM_CONSTANT_Integer :
				{
					cfs->guarantee_more(5, CHECK);  // bytes, tag/access_flags
					u4 bytes = cfs->get_u4_fast();
					_cp->int_at_put(index, (jint) bytes);
				}
				break;
			case JVM_CONSTANT_Float :
				{
					cfs->guarantee_more(5, CHECK);  // bytes, tag/access_flags
					u4 bytes = cfs->get_u4_fast();
					_cp->float_at_put(index, *(jfloat*)&bytes);
				}
				break;
			case JVM_CONSTANT_Long :
				{
					cfs->guarantee_more(9, CHECK);  // bytes, tag/access_flags
					u8 bytes = cfs->get_u8_fast();
					_cp->long_at_put(index, bytes);
				}
				index++;   // Skip entry following eigth-byte constant, see JVM book p. 98
				break;
			case JVM_CONSTANT_Double :
				{
					cfs->guarantee_more(9, CHECK);  // bytes, tag/access_flags
					u8 bytes = cfs->get_u8_fast();
					_cp->double_at_put(index, *(jdouble*)&bytes);
				}
				index++;   // Skip entry following eigth-byte constant, see JVM book p. 98
				break;
			case JVM_CONSTANT_NameAndType :
				{
					cfs->guarantee_more(5, CHECK);  // name_index, signature_index, tag/access_flags
					u2 name_index = cfs->get_u2_fast();
					u2 signature_index = cfs->get_u2_fast();
					_cp->name_and_type_at_put(index, name_index, signature_index);
				}
				break;
			case JVM_CONSTANT_Utf8 :
				{
					cfs->guarantee_more(2, CHECK);  // utf8_length
					u2  utf8_length = cfs->get_u2_fast();
					u1* utf8_buffer = cfs->get_u1_buffer();
					assert(utf8_buffer != NULL, "null utf8 buffer");
					// Got utf8 string, guarantee utf8_length+1 bytes, set stream position forward.
					cfs->guarantee_more(utf8_length+1, CHECK);  // utf8 string, tag/access_flags
					cfs->skip_u1_fast(utf8_length);

					if (EnableInvokeDynamic && has_cp_patch_at(index)) {
						Handle patch = clear_cp_patch_at(index);

						char* str = java_lang_String::as_utf8_string(patch());
						// (could use java_lang_String::as_symbol instead, but might as well batch them)
						utf8_buffer = (u1*) str;
						utf8_length = (int) strlen(str);
					}

					unsigned int hash;
					Symbol* result = SymbolTable::lookup_only((char*)utf8_buffer, utf8_length, hash);
					if (result == NULL) {
						names[names_count] = (char*)utf8_buffer;
						lengths[names_count] = utf8_length;
						indices[names_count] = index;
						hashValues[names_count++] = hash;
						if (names_count == SymbolTable::symbol_alloc_batch_size) {
							SymbolTable::new_symbols(_loader_data, _cp, names_count, names, lengths, indices, hashValues, CHECK);
							names_count = 0;
						}
					} else {
						_cp->symbol_at_put(index, result);
					}
				}
				break;
			default:
				classfile_parse_error("Unknown constant tag %u in class file %s", tag, CHECK);
				break;
		}
	}

	// Allocate the remaining symbols
	if (names_count > 0) {
		SymbolTable::new_symbols(_loader_data, _cp, names_count, names, lengths, indices, hashValues, CHECK);
	}

	cfs0->set_current(cfs1.current());
}

```

The loop processes length constant pool items, but the first constant
pool item does not need to be processed, so the loop subscript is
initialized to 1.

The first byte of each element is used to desctibe the constant pool
element type. After 
```calling cfs->get_u1_fast()```
to get the element type, it can be processed by the switch statement.


