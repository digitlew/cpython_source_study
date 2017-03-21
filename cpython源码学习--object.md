---
Include/object.h & Objects/object.c
---

#*<font color=blue>0.referrence</font>*#
    Pythoon.h
    frameobject.h



#*<font color=blue>1.content</font>*#
interesting point list:

## data allocate only on  heap ##

1. Objects are structures allocated  *on the heap*; 
1. Objects are never allocated statically or on the stack;
1. type objects are exceptions to the first rule; the standard types are represented by statically initialized type objects

## reference count##
    
<font color=green>gc：something like smart ptr in c++ for garbage collected, but on which situation will the function consume ref count is tricky.</font>

digest from source file

> Reference Counts
> 
> It takes a while to get used to the proper usage of reference counts.
> 
> Functions that create an object set the reference count to 1; such new
> objects must be stored somewhere or destroyed again with Py_DECREF().
> Some functions that 'store' objects, such as PyTuple_SetItem() and
> PyList_SetItem(),
> don't increment the reference count of the object, since the most
> frequent use is to store a fresh object.  Functions that 'retrieve'
> objects, such as PyTuple_GetItem() and PyDict_GetItemString(), also
> don't increment
> the reference count, since most frequently the object is only looked at
> quickly.  Thus, to retrieve an object and store it again, the caller
> must call Py_INCREF() explicitly.
> 
> NOTE: functions that 'consume' a reference count, like
> PyList_SetItem(), consume the reference even if the object wasn't
> successfully stored, to simplify error handling.
> 
> It seems attractive to make other functions that take an object as
> argument consume a reference count; however, this may quickly get
> confusing (even the current practice is already confusing).  Consider
> it carefully, it may save lots of calls to Py_INCREF() and Py_DECREF() at
> times.

***create ref*** 

   Head of circular doubly-linked list

   `static PyObject refchain = {&refchain, &refchain};`
 
insert PyObject into ref chain
  
    void
    _Py_AddToAllObjects(PyObject *op, int force)
    {
    #ifdef  Py_DEBUG
    if (!force) {
    /* If it's initialized memory, op must be in or out of
     * the list unambiguously.
     */
    assert((op->_ob_prev == NULL) == (op->_ob_next == NULL));
    }
    #endif
    if (force || op->_ob_prev == NULL) {
    op->_ob_next = refchain._ob_next;
    op->_ob_prev = &refchain;
    refchain._ob_next->_ob_prev = op;
    refchain._ob_next = op;
    }
    }

create PyTypeObject list

    static PyTypeObject *type_list;  // type with instantiation
    static int unlist_types_without_objects;  // without instantial object



## object type ##
1. type is fixed when obj is created.
2. contain a ptr to type obj.
3. type itself has ptr pointing obj which represent the type`s type

## in memory ##
1. obj not float in memory
2. var-size obj has prt to var-size parts of obj(gc：like linked list？)
3. size never change after allocation.

## access obj ##
1. using PyObject* or casting to a longer structs.
2. obj has other data(exclude ref_count and type) should declared as a longer structs, which start with  referenct and type fields.
    

## garbage collection ##

<font color=red>cyclic gc problem</font> , quote from annotation
> Trashcan mechanism, thanks to Christian Tismer.
> 
> When deallocating a container object, it's possible to trigger an unbounded
> chain of deallocations, as each Py_DECREF in turn drops the refcount on "the
> next" object in the chain to 0.  This can easily lead to stack faults, and
> especially in threads (which typically have less stack space to work with).
> 
> A container object that participates in cyclic gc can avoid this by
> bracketing the body of its tp_dealloc function with a pair of macros:
> 
> static void
> mytype_dealloc(mytype *p)
> {
>     ... declarations go here ...
> 
>     PyObject_GC_UnTrack(p);        // must untrack first
>     Py_TRASHCAN_SAFE_BEGIN(p)
>     ... The body of the deallocator goes here, including all calls ...
>     ... to Py_DECREF on contained objects.                         ...
>     Py_TRASHCAN_SAFE_END(p)
> }
> 
> CAUTION:  Never return from the middle of the body!  If the body needs to
> "get out early", put a label immediately before the Py_TRASHCAN_SAFE_END
> call, and goto it.  Else the call-depth counter (see below) will stay
> above 0 forever, and the trashcan will never get emptied.


    #define PyTrash_UNWIND_LEVEL 50
    #define Py_TRASHCAN_SAFE_BEGIN(op) \
    do { \
        PyThreadState *_tstate = PyThreadState_GET(); \
        if (!_tstate || \
            _tstate->trash_delete_nesting < PyTrash_UNWIND_LEVEL) { \
            if (_tstate) \
                ++_tstate->trash_delete_nesting;
            /* The body of the deallocator is here. */
    #define Py_TRASHCAN_SAFE_END(op) \
            if (_tstate) { \
                --_tstate->trash_delete_nesting; \
                if (_tstate->trash_delete_later \
                    && _tstate->trash_delete_nesting <= 0) \
                    _PyTrash_thread_destroy_chain(); \
            } \
        } \
        else \
            # put op into tstate->trash_delete_later, trash_delete_later is  doubly-linked list 
            _PyTrash_thread_deposit_object((PyObject*)op); \
    } while (0);
    # 
    # PyThreadState  is defined in pystate.h
    # _PyTrash_thread_deposit_object is implement in object.c
    void
    _PyTrash_thread_deposit_object(PyObject *op)
    {
        PyThreadState *tstate = PyThreadState_GET();
        assert(PyObject_IS_GC(op));
        assert(_Py_AS_GC(op)->gc.gc_refs == _PyGC_REFS_UNTRACKED);
        assert(op->ob_refcnt == 0);
        _Py_AS_GC(op)->gc.gc_prev = (PyGC_Head *) tstate->trash_delete_later;
        tstate->trash_delete_later = op;
    }

    void
    _PyTrash_destroy_chain(void)
    {
    while (_PyTrash_delete_later) {
    PyObject *op = _PyTrash_delete_later;
    destructor dealloc = Py_TYPE(op)->tp_dealloc;
    
    _PyTrash_delete_later =
    (PyObject*) _Py_AS_GC(op)->gc.gc_prev;
    
    /* Call the deallocator directly.  This used to try to
     * fool Py_DECREF into calling it indirectly, but
     * Py_DECREF was already called on this object, and in
     * assorted non-release builds calling Py_DECREF again ends
     * up distorting allocation statistics.
     */
    assert(op->ob_refcnt == 0);
    ++_PyTrash_delete_nesting;
    (*dealloc)(op);
    --_PyTrash_delete_nesting;
    }
    }

    # implement in Include/objimpl.h
    typedef union _gc_head {  
    struct {
    union _gc_head *gc_next;  
    union _gc_head *gc_prev;
    Py_ssize_t gc_refs;   
    } gc;
    long double dummy;  /* force worst-case alignment */  
    } PyGC_Head;
    
    extern PyGC_Head *_PyGC_generation0;

    /* Test if an object has a GC head */
    #define PyObject_IS_GC(o) (PyType_IS_GC(Py_TYPE(o)) && \
        (Py_TYPE(o)->tp_is_gc == NULL || Py_TYPE(o)->tp_is_gc(o)))
    #define _Py_AS_GC(o) ((PyGC_Head *)(o)-1)
    #define _PyGC_REFS_UNTRACKED                    (-2)
<font color=red>***gc：tstate->trash_delete_nesting never goes to PyTrash_UNWIND_LEVEL? AS the inc and dcr always happened simultaneously*** . </font>




#*<font color=blue>2.key data_struct</font>*#

## maros & variant ##
<font color=green>gc:some maros defined in struct in long struct. 
Any PyVarObject pointer(actually point to python object) can cast to PyObject pointer. This is inheritance and polymorphism like c++. Secondly, python object is actually a point in c implement.

</font>

    #define _PyObject_HEAD_EXTRA            \
    struct _object *_ob_next;               \
    struct _object *_ob_prev;
    
    #define _PyObject_EXTRA_INIT 0, 0,
    ....
 
    /* PyObject_HEAD defines the initial segment of every PyObject. */
    #define PyObject_HEAD                   \
        _PyObject_HEAD_EXTRA                \
        Py_ssize_t ob_refcnt;               \
        struct _typeobject *ob_type;    \\

    #define PyObject_HEAD_INIT(type)        \
        _PyObject_EXTRA_INIT                \
        1, type,

    #define PyObject_VAR_HEAD               \
        PyObject_HEAD                       \
        Py_ssize_t ob_size; /* Number of items in variable part */
    
    typedef struct _object {
        PyObject_HEAD
    } PyObject;

    typedef struct {
        PyObject_VAR_HEAD
    } PyVarObject;

### some flags ###

    /* Objects support garbage collection (see objimp.h) */
    #define Py_TPFLAGS_HAVE_GC (1L<<14)
    ....

### some annotations ###

- the reference count field can never overflow; this can
be proven when the size of the field is the same as the pointer size
- Type objects should never be deallocated; the type pointer in an object
is not considered to be a reference to the type object
 


## function ptr ##
<font color=green>gc: some function pointer , omited</font>

type check:
    
    #define PyObject_TypeCheck(ob, tp) \
    (Py_TYPE(ob) == (tp) || PyType_IsSubtype(Py_TYPE(ob), (tp)))
    
operation on obj:

    PyAPI_FUNC(int) PyObject_Print(PyObject *, FILE *, int);
    PyAPI_FUNC(void) _PyObject_Dump(PyObject *);
    PyAPI_FUNC(PyObject *) PyObject_Repr(PyObject *);
    PyAPI_FUNC(PyObject *) _PyObject_Str(PyObject *);

Tips for PyAPI\_FUNC & PyAPI\_DATA:

    # for cgwin or shared: __declspec(dllimport) means using dll public symbol
    # define PyAPI_FUNC(RTYPE) __declspec(dllimport) RTYPE 
    # define PyAPI_DATA(RTYPE) extern __declspec(dllexport) RTYPE
    # otherwise
    # define PyAPI_FUNC(RTYPE) RTYPE
    # define PyAPI_DATA(RTYPE) extern RTYPE

ref_count operation
    
    #using temp variant to free pointer
    #define Py_CLEAR(op)                       \
    do {                                        \
        if (op) {                               \
            PyObject *_py_tmp = (PyObject *)(op);               \
            (op) = NULL;                        \
            Py_DECREF(_py_tmp);                 \
        }                                       \
    } while (0)
    
    # None in python
    PyAPI_DATA(PyObject) _Py_NoneStruct; /* Don't use this directly */
    #define Py_None (&_Py_NoneStruct)

    /* Macro for returning Py_None from a function */
    #define Py_RETURN_NONE return Py_INCREF(Py_None), Py_None

## struct ##
1. PyNumberMethods:define basis operator for obj like add\sub\multi\divide
2. PySequenceMethods: operator on sequence like slice\concat
3. PyMappingMethods: operator for dict  like  subscript
4. PyBufferProcs: operator on buffer for read/writer or something other

<font color=red>PyTypeObject</font>: very import struct and basic to any type of class(user defined or builtin).  Included

    I) function poiter for getter/setter/destruct/cmp/repr...; 

    II) basis variant like tp_doc/tp_iter/tp_dict... . 

AS all python obj can cast into a point to PyObject, this struct can be see as <font color=green>template class</font>

<font color=red>PyHeapTypeObject</font>:


#*<font color=blue>3.kernel code</font>*#

## function involved in creating an object  ##

----------

add one type object

    void inc_count(PyTypeObject *tp)
    {
    if (tp->tp_next == NULL && tp->tp_prev == NULL) {
        /* first time; insert in linked list */
        if (tp->tp_next != NULL) /* sanity check */
            Py_FatalError("XXX inc_count sanity check");
        if (type_list)
            type_list->tp_prev = tp;
        tp->tp_next = type_list;
        /* Note that as of Python 2.2, heap-allocated type objects
         * can go away, but this code requires that they stay alive
         * until program exit.  That's why we're careful with
         * refcounts here.  type_list gets a new reference to tp,
         * while ownership of the reference type_list used to hold
         * (if any) was transferred to tp->tp_next in the line above.
         * tp is thus effectively immortal after this.
         */
        Py_INCREF(tp);
        type_list = tp;
    #ifdef Py_TRACE_REFS
        /* Also insert in the doubly-linked list of all objects,
         * if not already there.
         */
        _Py_AddToAllObjects((PyObject *)tp, 0);
    #endif
    }
    tp->tp_allocs++;
    if (tp->tp_allocs - tp->tp_frees > tp->tp_maxalloc)
        tp->tp_maxalloc = tp->tp_allocs - tp->tp_frees;
    }

----------

remove one type object


    void dec_count(PyTypeObject *tp)
    {
    tp->tp_frees++;
    if (unlist_types_without_objects &&
        tp->tp_allocs == tp->tp_frees) {
        /* unlink the type from type_list */
        if (tp->tp_prev)
            tp->tp_prev->tp_next = tp->tp_next;
        else
            type_list = tp->tp_next;
        if (tp->tp_next)
            tp->tp_next->tp_prev = tp->tp_prev;
        tp->tp_next = tp->tp_prev = NULL;
        Py_DECREF(tp);
    }
    }

----------

ref operation on object
    
    void
    Py_IncRef(PyObject *o)
    {
    Py_XINCREF(o);
    }
    
    void
    Py_DecRef(PyObject *o)
    {
    Py_XDECREF(o);
    }

----------

create non-variable obj

    PyObject *
    PyObject_Init(PyObject *op, PyTypeObject *tp)
    {
    if (op == NULL)
        return PyErr_NoMemory();
    /* Any changes should be reflected in PyObject_INIT (objimpl.h) */
    Py_TYPE(op) = tp;
    _Py_NewReference(op);
    return op;
    }

----------

create var-length obj

    PyVarObject *
    PyObject_InitVar(PyVarObject *op, PyTypeObject *tp, Py_ssize_t size)
    {
    if (op == NULL)
        return (PyVarObject *) PyErr_NoMemory();
    /* Any changes should be reflected in PyObject_INIT_VAR */
    op->ob_size = size;
    Py_TYPE(op) = tp;
    _Py_NewReference((PyObject *)op);
    return op;
    }

----------

add ref for type and obj
    
    void _Py_NewReference(PyObject *op)
    {                                           
    _Py_INC_REFTOTAL;   //add total ref
    op->ob_refcnt = 1; 
    _Py_AddToAllObjects(op, 1);  //
    _Py_INC_TPALLOCS(op);   // add type of obj referenct
     }


----------

remove ref for obj

<font color=red>gc: should not consider thre ref_cnt is zero?</font>

`void _Py_ForgetReference(register PyObject *op)`



----------

     void _Py_AddToAllObjects(PyObject *op, int force)
    {
     #ifdef  Py_DEBUG
     if (!force) {
        /* If it's initialized memory, op must be in or out of
         * the list unambiguously.
         */
        assert((op->_ob_prev == NULL) == (op->_ob_next == NULL));
    }
    #endif
    if (force || op->_ob_prev == NULL) {
        op->_ob_next = refchain._ob_next;   //refchain maintain whole ref of objs
        op->_ob_prev = &refchain;
        refchain._ob_next->_ob_prev = op;   // add new obj in the head
        refchain._ob_next = op;
    }
    }
    #endif  /* Py_TRACE_REFS */

----------

allocte memory for var_size obj

<font color=green> gc:length of var_size obj is actually fixed. This is same with c++ stl for vector .When add or del  item from var_size obj , the resize operation will trigger .   </font>

    #define _PyObject_VAR_SIZE(typeobj, nitems)     \
    (size_t)                                    \
    ( ( (typeobj)->tp_basicsize +               \
        (nitems)*(typeobj)->tp_itemsize +       \
        (SIZEOF_VOID_P - 1)                     \ 
      ) & ~(SIZEOF_VOID_P - 1)                  \ ensure integer times of SIZEOF_VOID_P
    )

----------

    # allocte for var_size obj
    PyVarObject *
       _PyObject_NewVar(PyTypeObject *tp, Py_ssize_t nitems)
    {
    PyVarObject *op;
    const size_t size = _PyObject_VAR_SIZE(tp, nitems); 
    op = (PyVarObject *) PyObject_MALLOC(size);
    if (op == NULL)
        return (PyVarObject *)PyErr_NoMemory();
    return PyObject_INIT_VAR(op, tp, nitems);
    }




## function involved in dumping an object  ##

\_\_str\_\_ & \_\_repr\_\_

<font color=green>gc: for every PyObject , the dump funciton call repr first, even if __repr__ isn`t define exclipect </font>

----------


    void _PyObject_Dump(PyObject* op)
    {
    if (op == NULL)
        fprintf(stderr, "NULL\n");
    else {
    #ifdef WITH_THREAD
        PyGILState_STATE gil;    // multi-thread will consider gil
    #endif
        fprintf(stderr, "object  : ");
    #ifdef WITH_THREAD
        gil = PyGILState_Ensure();
    #endif
        (void)PyObject_Print(op, stderr, 0);  // flag = 0 ,so use repr
    #ifdef WITH_THREAD
        PyGILState_Release(gil);
    #endif
        /* XXX(twouters) cast refcount to long until %zd is
           universally available */
        fprintf(stderr, "\n"
            "type    : %s\n"
            "refcount: %ld\n"
            "address : %p\n",
            Py_TYPE(op)==NULL ? "NULL" : Py_TYPE(op)->tp_name,
            (long)op->ob_refcnt,
            op);
    }
    }

----------

    ## function for print

    #define Py_PRINT_RAW    1
    static int
      internal_print(PyObject *op, FILE *fp, int flags, int nesting)
    {
    int ret = 0;
    if (nesting > 10) {
        PyErr_SetString(PyExc_RuntimeError, "print recursion");
        return -1;
    }
    if (PyErr_CheckSignals())
        return -1;
    #ifdef USE_STACKCHECK
    if (PyOS_CheckStack()) {
        PyErr_SetString(PyExc_MemoryError, "stack overflow");
        return -1;
    }
    #endif
    clearerr(fp); /* Clear any previous error condition */
    if (op == NULL) {
        Py_BEGIN_ALLOW_THREADS
        fprintf(fp, "<nil>");
        Py_END_ALLOW_THREADS
    }
    else {
        if (op->ob_refcnt <= 0)
            /* XXX(twouters) cast refcount to long until %zd is
               universally available */
            Py_BEGIN_ALLOW_THREADS
            fprintf(fp, "<refcnt %ld at %p>",
                (long)op->ob_refcnt, op);
            Py_END_ALLOW_THREADS
        else if (Py_TYPE(op)->tp_print == NULL) {
            PyObject *s;
            if (flags & Py_PRINT_RAW)
                s = PyObject_Str(op);
            else
                s = PyObject_Repr(op);
            if (s == NULL)
                ret = -1;
            else {
                ret = internal_print(s, fp, Py_PRINT_RAW,
                                     nesting+1);
            }
            Py_XDECREF(s);
        }
        else
            ret = (*Py_TYPE(op)->tp_print)(op, fp, flags);
    }
    if (ret == 0) {
        if (ferror(fp)) {
            PyErr_SetFromErrno(PyExc_IOError);
            clearerr(fp);
            ret = -1;
        }
    }
    return ret;
    }

----------

    ## repr function
    # 1. use py_type(o)->tp_repr ,if defined by user
    # 2. use PyString_FromFormat to print Py_TYPE(o) and o`s address(64-bit is 0xffffffffffffffff )

    PyObject_Repr(PyObject *v)
    {
    if (PyErr_CheckSignals())
        return NULL;
    #ifdef USE_STACKCHECK
    if (PyOS_CheckStack()) {
        PyErr_SetString(PyExc_MemoryError, "stack overflow");
        return NULL;
    }
    #endif
    if (v == NULL)
        return PyString_FromString("<NULL>");
    else if (Py_TYPE(v)->tp_repr == NULL)
        return PyString_FromFormat("<%s object at %p>",
                                   Py_TYPE(v)->tp_name, v);
         ## print var_size parameters ,detail in function :PyString_FromFormatV
    else {
        PyObject *res;
        res = (*Py_TYPE(v)->tp_repr)(v);
        if (res == NULL)
            return NULL;
    #ifdef Py_USING_UNICODE
        if (PyUnicode_Check(res)) {
            PyObject* str;
            str = PyUnicode_AsEncodedString(res, NULL, NULL);
            Py_DECREF(res);
            if (str)
                res = str;
            else
                return NULL;
        }
    #endif
        if (!PyString_Check(res)) {
            PyErr_Format(PyExc_TypeError,
                         "__repr__ returned non-string (type %.200s)",
                         Py_TYPE(res)->tp_name);
            Py_DECREF(res);
            return NULL;
        }
        return res;
    }
    }


## function involved in cmparison  objects (two or more)  ##

    int _Py_SwappedOp[] = {Py_GT, Py_GE, Py_EQ, Py_NE, Py_LT, Py_LE};

define compare function as: 


<font color=green>gc: not quitely understand PyInstance_Type used for </font>

    #define PyInstance_Check(op) ((op)->ob_type == &PyInstance_Type)

    # use for what purpose?
    static int try_3way_compare(PyObject *v, PyObject *w)
    {
    int c;
    cmpfunc f;

    /* Comparisons involving instances are given to instance_compare,
       which has the same return conventions as this function. */

    f = v->ob_type->tp_compare;
    if (PyInstance_Check(v))
        return (*f)(v, w);
    if (PyInstance_Check(w))
        return (*w->ob_type->tp_compare)(v, w);

    /* If both have the same (non-NULL) tp_compare, use it. */
    if (f != NULL && f == w->ob_type->tp_compare) {
        c = (*f)(v, w);
        return adjust_tp_compare(c);
    }

    /* If either tp_compare is _PyObject_SlotCompare, that's safe. */
    # call either __cmp__ function 
    if (f == _PyObject_SlotCompare ||
        w->ob_type->tp_compare == _PyObject_SlotCompare)
        return _PyObject_SlotCompare(v, w);

<font color=red>***gc：why we call PyNumber_	CoerceEx? what hanppend to v and w ?*** . </font>

    # res = (*v->ob_type->tp_as_number->nb_coerce)(pv, pw);
    #  or
    # res = (*w->ob_type->tp_as_number->nb_coerce)(pw, pv);
    # nb_coerce is only a function pointer , the implemention I don`t find now
    # typedef int (*coercion)(PyObject **, PyObject **);
    c = PyNumber_CoerceEx(&v, &w);
    if (c < 0)
        return -2;
    if (c > 0)
        return 2;
    f = v->ob_type->tp_compare;
    if (f != NULL && f == w->ob_type->tp_compare) {
        c = (*f)(v, w);
        Py_DECREF(v);
        Py_DECREF(w);
        return adjust_tp_compare(c);
    }

    /* No comparison defined */
    Py_DECREF(v);
    Py_DECREF(w);
    return 2;
    }


----------

    # cmp tp_name?
    static int default_3way_compare(PyObject *v, PyObject *w)


----------

    # console for cmp function, it will call all those cmp function above in each  situation

    static int do_cmp(PyObject *v, PyObject *w)
    {
    int c;
    cmpfunc f;

    if (v->ob_type == w->ob_type
        && (f = v->ob_type->tp_compare) != NULL) {
        c = (*f)(v, w);
        if (PyInstance_Check(v)) {
            /* Instance tp_compare has a different signature.
               But if it returns undefined we fall through. */
            if (c != 2)
                return c;
            /* Else fall through to try_rich_to_3way_compare() */
        }
        else
            return adjust_tp_compare(c);
    }
    /* We only get here if one of the following is true:
       a) v and w have different types
       b) v and w have the same type, which doesn't have tp_compare
       c) v and w are instances, and either __cmp__ is not defined or
          __cmp__ returns NotImplemented
    */
    c = try_rich_to_3way_compare(v, w);
    if (c < 2)
        return c;
    c = try_3way_compare(v, w);
    if (c < 2)
        return c;
    return default_3way_compare(v, w);
    }

----------

    # cmp two objects ,start from this function !
    int  PyObject_Compare(PyObject *v, PyObject *w)
    {
    int result;

    if (v == NULL || w == NULL) {
        PyErr_BadInternalCall();
        return -1;
    }
    if (v == w)
        return 0;
    if (Py_EnterRecursiveCall(" in cmp")) //  something like control PyThreadState_GET()->recursion_depth , add 1
        return -1;
    result = do_cmp(v, w);
    Py_LeaveRecursiveCall();   // minuus 1
    return result < 0 ? -1 : result;
    }



define richcompare  function as:

<font color=green>gc:richcompare mean not only greater/less/equal, but also combination of above </font>

    # template function for rich_compare within any type of two objs
    static PyObject * try_rich_compare(PyObject *v, PyObject *w, int op)

----------


    # call try_rich_compare ,translate res to (-1,0,1,2)
    static int try_rich_compare_bool(PyObject *v, PyObject *w, int op)

----------

    #  call try_3way_compare 
    #  if c>=2 try_3way_compare not implement of undefined
    #  -2<c<2  call conver_3way_to_object
    static PyObject * try_3way_to_rich_compare(PyObject *v, PyObject *w, int op)

----------

    # console for richcompare
    static PyObject * do_richcmp(PyObject *v, PyObject *w, int op)
    {
    PyObject *res;

    res = try_rich_compare(v, w, op);
    if (res != Py_NotImplemented)
        return res;
    Py_DECREF(res);

    return try_3way_to_rich_compare(v, w, op);
    }


----------

 gate of compare function ,get in this function and then goes to some function above. 

    PyObject * PyObject_RichCompare(PyObject *v, PyObject *w, int op)
    {
    PyObject *res;

    assert(Py_LT <= op && op <= Py_GE);
    if (Py_EnterRecursiveCall(" in cmp"))
        return NULL;

    /* If the types are equal, and not old-style instances, try to
       get out cheap (don't bother with coercions etc.). */
    if (v->ob_type == w->ob_type && !PyInstance_Check(v)) {
        cmpfunc fcmp;
        richcmpfunc frich = RICHCOMPARE(v->ob_type);
        /* If the type has richcmp, try it first.  try_rich_compare
           tries it two-sided, which is not needed since we've a
           single type only. */
        if (frich != NULL) {
            res = (*frich)(v, w, op);
            if (res != Py_NotImplemented)
                goto Done;
            Py_DECREF(res);
        }
        /* No richcmp, or this particular richmp not implemented.
           Try 3-way cmp. */
        fcmp = v->ob_type->tp_compare;
        if (fcmp != NULL) {
            int c = (*fcmp)(v, w);
            c = adjust_tp_compare(c);
            if (c == -2) {
                res = NULL;
                goto Done;
            }
            res = convert_3way_to_object(op, c);
            goto Done;
        }
    }

    /* Fast path not taken, or couldn't deliver a useful result. */
    res = do_richcmp(v, w, op);
    Done:
    Py_LeaveRecursiveCall();
    return res;
    }


## function involved in hash  ##


<font color=green>gc:    
    
    #hash(1.0 /3.0) = 2147450709 why?
    # mantissa = frexp(1.0/3.0, expo)

    # ===> double represent in IEEE : (1+1/3) * 2**(-2) 

    # but, frexp is implemented as:
>       {
>        *exp = (value == 0) ? 0 : (int)(1 + std::logb(value));
>        return std::scalbn(value, -(*exp));
>        }
>        #return muse in range of [0.5,1] or [-0.5,-1] 

    # so the mantissa is not the one represent in bits (at [51,0])
    # ===> mantissa = 0.6666667
    # ===> v = mantissa * 2147483648.0 (2**32)
    # ===> li = (long)v; v=(v-li)*2147483648.0(2**32)
    # ===> m = li + (long)v + expo(8) << 15  
    # ===> 2147450709
</font>  

    # source code:
    long _Py_HashDouble(double v)
    {
    double intpart, fractpart;
    int expo;
    long hipart;
    long x;             /* the final hash value */

    if (!Py_IS_FINITE(v)) {
        if (Py_IS_INFINITY(v))
            return v < 0 ? -271828 : 314159;
        else
            return 0;
    }
    fractpart = modf(v, &intpart);
    if (fractpart == 0.0) {
        /* This must return the same hash as an equal int or long. */
        if (intpart > LONG_MAX/2 || -intpart > LONG_MAX/2) {
            /* Convert to long and use its hash. */
            PyObject *plong;                    /* converted to Python long */
            plong = PyLong_FromDouble(v);
            if (plong == NULL)
                return -1;
            x = PyObject_Hash(plong);
            Py_DECREF(plong);
            return x;
        }
        /* Fits in a C long == a Python int, so is its own hash. */
        x = (long)intpart;
        if (x == -1)
            x = -2;
        return x;
    }
    /* The fractional part is non-zero, so we don't have to worry about
     * making this match the hash of some other type.
     * Use frexp to get at the bits in the double.
     * Since the VAX D double format has 56 mantissa bits, which is the
     * most of any double format in use, each of these parts may have as
     * many as (but no more than) 56 significant bits.
     * So, assuming sizeof(long) >= 4, each part can be broken into two
     * longs; frexp and multiplication are used to do that.
     * Also, since the Cray double format has 15 exponent bits, which is
     * the most of any double format in use, shifting the exponent field
     * left by 15 won't overflow a long (again assuming sizeof(long) >= 4).
     */
    v = frexp(v, &expo);
    v *= 2147483648.0;          /* 2**31 */
    hipart = (long)v;           /* take the top 32 bits */
    v = (v - (double)hipart) * 2147483648.0; /* get the next 32 bits */
    x = hipart + (long)v + (expo << 15);
    if (x == -1)
        x = -2;
    return x;
    }

    # first need to judge whether X is inf, as inf in double-precise ,it represent as
    # 7ff0 0000 0000 000016   = Infinity
    #  fff0 0000 0000 000016   = −Infinity
    # define Py_IS_INFINITY(X) ((X) &&                                   \
               (Py_FORCE_DOUBLE(X)*0.5 == Py_FORCE_DOUBLE(X)))

<font color=green>gc: call v->ob_type->tp_hash function for hash(v)
    if v->ob_type not ready(maybe some member/function not define), call **PyType\_Ready** to inherent from base.
</font>

    long PyObject_Hash(PyObject *v)
    {
    PyTypeObject *tp = v->ob_type;
    if (tp->tp_hash != NULL)
        return (*tp->tp_hash)(v);
    /* To keep to the general practice that inheriting
     * solely from object in C code should work without
     * an explicit call to PyType_Ready, we implicitly call
     * PyType_Ready here and then check the tp_hash slot again
     */
    if (tp->tp_dict == NULL) {
        if (PyType_Ready(tp) < 0)
            return -1;
        if (tp->tp_hash != NULL)
            return (*tp->tp_hash)(v);
    }
    if (tp->tp_compare == NULL && RICHCOMPARE(tp) == NULL) {
        return _Py_HashPointer(v); /* Use address as hash value */
    }
    /* If there's a cmp but no hash defined, the object can't be hashed */
    return PyObject_HashNotImplemented(v);
    }
    

## function involved in reflection  ##

<font color=green>include: **getter\setter\hasattr** function </font>


----------

    # first call PyType(v)->tp_getattr 
    # otherwise translate char to PyObject 
    #           and call PyObject_GetAttr 
    PyObject * PyObject_GetAttrString(PyObject *v, const char *name)

    # PyString_Check(name)
    # tp->tp_getattro
    # tp->tp_getattr
    # do tp->tp_getattr twice??
    PyObject * PyObject_GetAttr(PyObject *v, PyObject *name)

    setattr function the same logic as getter, omit.


<font color=red>gc: dictptr = obj + dictoffset 
            
  dictoffset =  _PyObject_VAR_SIZE(tp, tsize)
 
  itemsize means the elements in var_size obj ,**but why the dict ptr come behinde all elements?**

 </font>

    PyObject ** _PyObject_GetDictPtr(PyObject *obj)

<font color=red>gc: get element from obj(lookup order of its bases).

there are some problem about this order , ex:http://www.ctolib.com/topics-87493.html

it has something with the imlpement of **tp->mro**, which is not deep-first order but layer-first order?  </font>

<font color =green>this is description of tp->mro (method resolution order) from python org.</font>

> PyObject* PyTypeObject.tp_mro¶
> Tuple containing the expanded set of base types, starting with the type itself and ending with object, in Method Resolution Order.
> 
> This field is not inherited; it is calculated fresh by PyType_Ready().


<font color=green>so, what is very important is the really order of call function when we use \_\_get\_\_ or attr lookup function.</font>

----------

    # generic getattr function , usually put in tp_getattro slot
    PyObject * _PyObject_GenericGetAttrWithDict(PyObject *obj, PyObject *name, PyObject *dict)

        # lookup in type
        descr = _PyType_Lookup(tp, name);
        # call  _PyType_Lookup(tp, name) inline function typeobject.c
        PyObject * _PyType_Lookup(PyTypeObject *type, PyObject *name)
        # compute 
           h = MCACHE_HASH_METHOD(type, name);
        # get
           return method_cache[h].value
        # otherwise, get name from mro
              for (i = 0; i < n; i++) {
                   base = PyTuple_GET_ITEM(mro, i);
                   if (PyClass_Check(base))
                       # has different with tp_dict ?
                       dict = ((PyClassObject *)base)->cl_dict; 
                  else {
                       assert(PyType_Check(base));
                       dict = ((PyTypeObject *)base)->tp_dict;
                  }
                  assert(dict && PyDict_Check(dict));
                  res = PyDict_GetItem(dict, name);
                  if (res != NULL)
                        break;
               }
           
         # is data description?
                f = NULL;
                if (descr != NULL &&
                   PyType_HasFeature(descr->ob_type, Py_TPFLAGS_HAVE_CLASS)) {
                   f = descr->ob_type->tp_descr_get;
                   if (f != NULL && PyDescr_IsData(descr)) {
                        # what for?
                        res = f(descr, obj, (PyObject *)obj->ob_type);
                        Py_DECREF(descr);
                        goto done;
                   }
                }
         # lookup in obj
         # if dict is Null , call _PyObject_GetDictPtr to get dict
          res = PyDict_GetItem(dict, name);
         
         # is dict has no name ,res is Null
          res = f(descr, obj, (PyObject *)Py_TYPE(obj));

         # is descr exist
           res = descr
         

----------

     int _PyObject_GenericSetAttrWithDict(PyObject *obj, PyObject *name,
                                 PyObject *value, PyObject *dict)

     # like getter
     # firtse get descr
     descr = _PyType_Lookup(tp, name);
     ....
        if (f != NULL && PyDescr_IsData(descr)) {
            res = f(descr, obj, value);
            goto done;
        }
     ....
     # use dict, is dict is Null , create new dict
        dictptr = _PyObject_GetDictPtr(obj);
        if (dictptr != NULL) {
            dict = *dictptr;
            if (dict == NULL && value != NULL) {
                dict = PyDict_New();
                if (dict == NULL)
                    goto done;
                *dictptr = dict;
            }
        } 
     # if dict not null, value null, delete key
        if (dict != NULL) {
            Py_INCREF(dict);
            if (value == NULL)
                res = PyDict_DelItem(dict, name);
            else
                res = PyDict_SetItem(dict, name, value);
    


## function involved in test  ##


    int PyObject_IsTrue(PyObject *v)
    # ob_type as number
        res = (*v->ob_type->tp_as_number->nb_nonzero)(v);
    # ob_type as mappint
        res = (*v->ob_type->tp_as_mapping->mp_length)(v);
    # ob_type as sequence
        res = (*v->ob_type->tp_as_sequence->sq_length)(v);
    # otherwise true!!
        return 1;


----------

    # defined a function for two numberic types converted to a common type
    # return a tuple coerce(x,y) == return (x1,y1)
    int PyNumber_CoerceEx(PyObject **pv, PyObject **pw)


----------

\_\_call\_\_

    int PyCallable_Check(PyObject *x)
    # return obj __call__
           if (PyInstance_Check(x)) {
        PyObject *call = PyObject_GetAttrString(x, "__call__");

    # otherwise return ob_type->tp_call
       return x->ob_type->tp_call != NULL;


## function involved in helper  ##

\_\_dir\_\_


----------
    # recursive add aclass`s __dict__ to dict, using  PyDict_Update in dictobject.c
    static int merge_class_dict(PyObject* dict, PyObject* aclass)
     
    int   PyDict_Update(PyObject *a, PyObject *b)
    {               
       return PyDict_Merge(a, b, 1); // 1 means overrider considering same entry with equavalent hashkey and key
    } 


----------
    # the same logic as dict ,using PyDict_SetItem(dict, item, Py_None) ,to insert entry from list to target dict
    static int merge_list_attr(PyObject* dict, PyObject* obj, const char *attrname)

----------
    # 
    static PyObject * _dir_locals(void)
    
       # get local env from PyThreadState
        PyObject * PyEval_GetLocals(void)
        {
           PyFrameObject *current_frame = PyEval_GetFrame();
           if (current_frame == NULL)
              return NULL;
           PyFrame_FastToLocals(current_frame);
           return current_frame->f_locals;
        }

        PyFrameObject * PyEval_GetFrame(void)
        {
           PyThreadState *tstate = PyThreadState_GET();
           return _PyThreadState_GetFrame(tstate);
        }

       # get keys from local ,and return

       #define PyMapping_Keys(O) PyObject_CallMethod(O,"keys",NULL)
      

----------

<font color=red>
gc: there are difference between type and generic object in dir function.
  
type need recursive for its \_\_base\_\_ , while object need \_\_class\_\_

</font>
> Helper for PyObject_Dir of type objects: returns __dict__ and __bases__.
>    We deliberately don't suck up its __class__, as methods belonging to the
>    metaclass would probably be more confusing than helpful.
> 


     `static PyObject * _specialized_dir_type(PyObject *obj)
    {
    PyObject *result = NULL;
    PyObject *dict = PyDict_New();

    if (dict != NULL && merge_class_dict(dict, obj) == 0)
        result = PyDict_Keys(dict);

    Py_XDECREF(dict);
    return result;
    }`


----------


> Helper for PyObject_Dir of generic objects: returns __dict__, __class__,
>    and recursively up the __class__.__bases__ chain.

`static PyObject *_generic_dir(PyObject *obj)`

    # add __member__ and __methods__
    if (merge_list_attr(dict, obj, "__members__") < 0)
        goto error;
    if (merge_list_attr(dict, obj, "__methods__") < 0)
        goto error;


----------

gerneric obj \_\_dir\_\_

    static PyObject * _dir_object(PyObject *obj)
    {
    PyObject *result = NULL;
    static PyObject *dir_str = NULL;
    PyObject *dirfunc;

    assert(obj);
    if (PyInstance_Check(obj)) {  
        dirfunc = PyObject_GetAttrString(obj, "__dir__");
        if (dirfunc == NULL) {
            if (PyErr_ExceptionMatches(PyExc_AttributeError))
                PyErr_Clear();
            else
                return NULL;
        }
    }
    else {
        dirfunc = _PyObject_LookupSpecial(obj, "__dir__", &dir_str);
        if (PyErr_Occurred())
            return NULL;
    }
    # non defined __dir__ function, use default
    if (dirfunc == NULL) {
        /* use default implementation */
        if (PyModule_Check(obj))
            result = _specialized_dir_module(obj);
        else if (PyType_Check(obj) || PyClass_Check(obj))
            result = _specialized_dir_type(obj);
        else
            result = _generic_dir(obj);
    }
    else {
        /* use __dir__ */
        result = PyObject_CallFunctionObjArgs(dirfunc, NULL);
        Py_DECREF(dirfunc);
        if (result == NULL)
            return NULL;

        /* result must be a list */
        /* XXX(gbrandl): could also check if all items are strings */
        if (!PyList_Check(result)) {
            PyErr_Format(PyExc_TypeError,
                         "__dir__() must return a list, not %.200s",
                         Py_TYPE(result)->tp_name);
            Py_DECREF(result);
            result = NULL;
        }
    }

    return result;
    

## function involved in None  ##

<font color=green>gc: any object of PyNone_Type is  reference to the same memory
and the refcnt to None is must be larger than 1  </font>

    static PyTypeObject PyNone_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "NoneType",
    0,
    0,
    none_dealloc,       /*tp_dealloc*/ /*never called*/
    0,                  /*tp_print*/
    0,                  /*tp_getattr*/
    0,                  /*tp_setattr*/
    0,                  /*tp_compare*/
    none_repr,          /*tp_repr*/
    0,                  /*tp_as_number*/
    0,                  /*tp_as_sequence*/
    0,                  /*tp_as_mapping*/
    (hashfunc)_Py_HashPointer, /*tp_hash */
    };

    PyObject _Py_NoneStruct = {
      _PyObject_EXTRA_INIT
    1, &PyNone_Type
    };

## some function invovle in memory  ##

----------
<font color=green>gc: detail in memory charpter </font>


## other function  ##

<font color=red>gc: do not get the point . maybe use this funciton to get args from a string convert to tuple </font>

    PyObject * _Py_GetObjects(PyObject *self, PyObject *args)
    {
    int i, n;
    PyObject *t = NULL;
    PyObject *res, *op;

    if (!PyArg_ParseTuple(args, "i|O", &n, &t))
        return NULL;
    op = refchain._ob_next;
    res = PyList_New(0);
    if (res == NULL)
        return NULL;
    for (i = 0; (n == 0 || i < n) && op != &refchain; i++) {
        while (op == self || op == args || op == res || op == t ||
               (t != NULL && Py_TYPE(op) != (PyTypeObject *) t)) {
            op = op->_ob_next;
            if (op == &refchain)
                return res;
        }
        if (PyList_Append(res, op) < 0) {
            Py_DECREF(res);
            return NULL;
        }
        op = op->_ob_next;
    }
    return res;
    }


hack some module
    
    /* Hack to force loading of capsule.o */
    PyTypeObject *_Py_capsule_hack = &PyCapsule_Type;
    
    
    /* Hack to force loading of cobject.o */
    PyTypeObject *_Py_cobject_hack = &PyCObject_Type;
    
    
    /* Hack to force loading of abstract.o */
    Py_ssize_t (*_Py_abstract_hack)(PyObject *) = PyObject_Size;


control infinite recursion in repr 

    # returns 0 the first time it is called for a particular object and 1 every time thereafter
    # using 
    #  dict = PyThreadState_GetDict();
    #  list = PyDict_GetItemString(dict, "Py_Repr");
    # to record obj
    int Py_ReprEnter(PyObject *obj)

    void Py_ReprLeave(PyObject *obj)


s