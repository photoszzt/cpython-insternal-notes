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
        oparg = NEXTARG();

HAS__ARG is defined in Include/opcode.h line 166.

	#define HAS_ARG(op) ((op >= HAVE_ARGUMENT))

HAVE_ARGUMENT is defined in line 94.

	#define HAVE_ARGUMENT 90

The opcode less than 90 does not take an argument and opcode larger than 90
takes argument.


