Python Value stack

Object uses refcount and when refcount equals to zero, the python interpreter 
frees the allocated memory.

# Frames, function calls and scope

python program:

    x= 10
    def foo(x):
        y = x * 2
        return bar(y)

    def bar(x):
        y = x / 2
        return y
    z = foo(x)

The corresponding python bytecode:

    $ python -m dis test.py
    1           0 LOAD_CONST               0 (10)
                3 STORE_NAME               0 (x)

    2           6 LOAD_CONST               1 (<code object foo at 0x7fbefd022830, file "test.py", line 2>)
                9 MAKE_FUNCTION            0
               12 STORE_NAME               1 (foo)

    6          15 LOAD_CONST               2 (<code object bar at 0x7fbefd022630, file "test.py", line 6>)
               18 MAKE_FUNCTION            0
               21 STORE_NAME               2 (bar)

    10         24 LOAD_NAME                1 (foo)
               27 LOAD_NAME                0 (x)
               30 CALL_FUNCTION            1
               33 STORE_NAME               3 (z)
               36 LOAD_CONST               3 (None)
               39 RETURN_VALUE

    >>> c = compile(open('test.py').read(), 'test.py', 'exec')
    >>> c.co_code() # shows the bytecode in string

Python interpreter:
ceval.c file contains the main interpreter loop

PyEval_EvalFrameEx executes one frame and returns a PyObject.
The following code is on line 1012 from cevel.c in Python dir.

    co = f->f_code; // Get the code object
    names = co->co_names;
    consts = co->co_consts;
    fastlocals = f->f_localsplus;
    freevars = f->f_localsplus + co->co_nlocals;
    first_instr = (unsigned char*) PyString_AS_STRING(co->co_code);

line 1057 is the interpreter loop. On line 1167, 

    opcode = NEXTOP() // Get the opcode
    oparg = 0;   /* allows oparg to be stored in a register because
            it doesn't have to be remembered across a full loop */
    if (HAS_ARG(opcode))
        oparg = NEXTARG(); // Get the argument

HAS__ARG is defined in Include/opcode.h line 166.

	#define HAS_ARG(op) ((op >= HAVE_ARGUMENT))

HAVE_ARGUMENT is defined in line 94.

	#define HAVE_ARGUMENT 90

The opcode less than 90 does not take an argument and opcode larger than 90
takes argument.

On line 1199, the switch statement checks each opcode and perform the corresponding 
operation. For example:

    TARGET_NOARG(POP_TOP)
        {
            v = POP(); // pop the value off the value stack
            Py_DECREF(v); // decrement the ref count of this object
            FAST_DISPATCH();
        }

There is also a function to increase the reference count: Py_INCREF.


python program:

    x= 10
    def foo(x):
        y = x * 2
        return bar(y)

    def bar(x):
        y = x / 2
        return y
    z = foo(x)

The corresponding python bytecode:

    $ python -m dis test.py
    1           0 LOAD_CONST               0 (10)
                3 STORE_NAME               0 (x)

    2           6 LOAD_CONST               1 (<code object foo at 0x7fbefd022830, file "test.py", line 2>)
                9 MAKE_FUNCTION            0
               12 STORE_NAME               1 (foo)

    6          15 LOAD_CONST               2 (<code object bar at 0x7fbefd022630, file "test.py", line 6>)
               18 MAKE_FUNCTION            0
               21 STORE_NAME               2 (bar)

    10         24 LOAD_NAME                1 (foo)
               27 LOAD_NAME                0 (x)
               30 CALL_FUNCTION            1
               33 STORE_NAME               3 (z)
               36 LOAD_CONST               3 (None)
               39 RETURN_VALUE

All the code is compiled but the function is not bound until runtime. The function
object contains code object and envirnment. The function object is a closure. 
A closure is a piece of code and the definding environment. The reason that the
function binds at runtime is it needs to bind to the right environment. The function
contains code and pointer to its environment. In this case, the environment is 
the global x = 10. 

foo --> function object --> code object
x --> object(10)

    import dis
    import test
    >>> dis.dis(test.foo)
    3           0 LOAD_FAST                0 (x)
                3 LOAD_CONST               1 (2)
                6 BINARY_MULTIPLY     
                7 STORE_FAST               1 (y)

    4          10 LOAD_GLOBAL              0 (bar)
               13 LOAD_FAST                1 (y)
               16 CALL_FUNCTION            1
               19 RETURN_VALUE        
    >>> dis.dis(test.bar)
    7           0 LOAD_FAST                0 (x)
                3 LOAD_CONST               1 (2)
                6 BINARY_DIVIDE       
                7 STORE_FAST               1 (y)

    8          10 LOAD_FAST                1 (y)
               13 RETURN_VALUE

In Include/code.h, line 10

	/* Bytecode object */
	typedef struct {
	    PyObject_HEAD
	    int co_argcount;›   ›   /* #arguments, except *args */
	    int co_nlocals;››   /* #local variables */
	    int co_stacksize;›  ›   /* #entries needed for evaluation stack */
	    int co_flags;›  ›   /* CO_..., see below */
	    PyObject *co_code;› ›   /* instruction opcodes */
	    PyObject *co_consts;›   /* list (constants used) */
	    PyObject *co_names;››   /* list of strings (names used) */
	    PyObject *co_varnames;› /* tuple of strings (local variable names) */
	    PyObject *co_freevars;› /* tuple of strings (free variable names) */
	    PyObject *co_cellvars;      /* tuple of strings (cell variable names) */
	    /* The rest doesn't count for hash/cmp */
	    PyObject *co_filename;› /* string (where it was loaded from) */
	    PyObject *co_name;› ›   /* string (name, for reference) */
	    int co_firstlineno;››   /* first source line number */
	    PyObject *co_lnotab;›   /* string (encoding addr<->lineno mapping) See
	›   ›   ›   ›      Objects/lnotab_notes.txt for details. */
	    void *co_zombieframe;     /* for optimization only (see frameobject.c) */
	    PyObject *co_weakreflist;   /* to support weakrefs to code objects */
	} PyCodeObject;

PyCodeObject gives information of the code and it should be immutable.

	typedef struct _frame {
	    PyObject_VAR_HEAD
	    struct _frame *f_back;› /* previous frame, or NULL */
	    PyCodeObject *f_code;›  /* code segment */
	    PyObject *f_builtins;›  /* builtin symbol table (PyDictObject) */
	    PyObject *f_globals;›   /* global symbol table (PyDictObject) */
	    PyObject *f_locals;››   /* local symbol table (any mapping) */
	    PyObject **f_valuestack;›   /* points after the last local */
	    /* Next free slot in f_valuestack.  Frame creation sets to f_valuestack.
	       Frame evaluation usually NULLs it, but a frame that yields sets it
	       to the current stack top. */
	    PyObject **f_stacktop;
	    PyObject *f_trace;› ›   /* Trace function */
	
	    /* If an exception is raised in this frame, the next three are used to
	     * record the exception info (if any) originally in the thread state.  See
	     * comments before set_exc_info() -- it's not obvious.
	     * Invariant:  if _type is NULL, then so are _value and _traceback.
	     * Desired invariant:  all three are NULL, or all three are non-NULL.  That
	     * one isn't currently true, but "should be".
	     */
	    PyObject *f_exc_type, *f_exc_value, *f_exc_traceback;
	
	    PyThreadState *f_tstate;
	    int f_lasti;›   ›   /* Last instruction if called */
	    /* Call PyFrame_GetLineNumber() instead of reading this field
	       directly.  As of 2.3 f_lineno is only valid when tracing is
	       active (i.e. when f_trace is set).  At other times we use
	       PyCode_Addr2Line to calculate the line from the current
	       bytecode index. */
	    int f_lineno;›  ›   /* Current line number */
	    int f_iblock;›  ›   /* index in f_blockstack */
	    PyTryBlock f_blockstack[CO_MAXBLOCKS]; /* for try and loop blocks */
	    PyObject *f_localsplus[1];› /* locals+stack, dynamically sized */
	} PyFrameObject; 

There is a frame stack. Every function should have one. It's a linked list connected
by the f_back pointer. In the example code, it will be

    global <-- foo <-- bar




## Function frame stack

