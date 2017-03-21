---
Objects/obmalloc.c
---

#*<font color=blue>0.referrence</font>*#
    Pythoon.h

----------

----------

#*<font color=blue>1.content</font>*#

<font color=green>
**fisrt, the layers of Python memory architectur**e
</font>

    Object-specific allocators
> **layer3** object-specific memory : string list int
> 
> **layer2** object memory: Python`s object allocator

> **layer1** python memory: python`s raw memory allocator(PyMem_API)

> **layer0** virtual memory allocated for the python process: C library malloc

> **layer-1** kernel dynamic storage(page-base): os-specific virtual memory manager(VMM)

> **layer-2** pyhsical memory ROM/RAM 、 swap

 
 <font color=green>**Allocation strategy abstract:**</font>

>  *
>  * For small requests, the allocator sub-allocates <Bigblocks of memory.
>  * **Requests greater than 256 bytes are routed to the system's allocator**.
>  *
>  * Small requests are grouped in size classes spaced 8 bytes apart, due
>  * to the required valid alignment of the returned address. Requests of
>  * **a particular size are serviced from memory pools of 4K (one VMM page)**.
>  * Pools are fragmented on demand and contain free lists of blocks of one
>  * particular size class. In other words, there is a fixed-size allocator
>  * for each size class. Free pools are shared by the different allocators
>  * thus minimizing the space reserved for a particular size class.
>  *
>  * This allocation strategy is a variant of what is known as "si**mple
>  * segregated storage based on array of free lists**". The main drawback of
>  * simple segregated storage is that we might end up with lot of reserved
>  * memory for the different free lists, which degenerate in time. To avoid
>  * this, we partition each free list in pools and we share dynamically the
>  * reserved space between all free lists. This technique is quite efficient
>  * for memory intensive programs which allocate mainly small-sized blocks.
> 

<font color=green>gc add 3/14/2017 12:08:15 PM :

some annotation about anera . Anera is big memory block ,usually one or more pages(Actually, an Anera has 256KB ,which means 64pages). Allocate an entire memory will ensure addressable for any Byte. </font>


 * The allocator sub-allocates <Bigblocks of memory (called arenas) aligned
 * on a page boundary. This is a reserved virtual address space for the
 * current process (obtained through a malloc call). In no way this means
 * that the memory arenas will be used entirely. A malloc(<Big>) is usually
 * an address range reservation for <Bigbytes, unless all pages within this
 * space are referenced subsequently. So malloc'ing big blocks and not using
 * them does not mean "wasting memory". It's an addressable range wastage...
 *
 * Therefore, allocating arenas with malloc is not optimal, because there is
 * some address space wastage, but this is the most portable way to request
 * memory from the system across various platforms.

unused_arena_objects

> This is a singly-linked list of the arena_objects that are currently not
> being used (no arena is associated with them).  Objects are taken off the
> head of the list in new_arena(), and are pushed on the head of the list in
> PyObject_Free() when the arena is empty.  Key invariant:  an arena_object
> is on this list if and only if its .address member is 0.


usable_arenas

> This is a doubly-linked list of the arena_objects associated with arenas
> that have pools available.  These pools are either waiting to be reused,
> or have not been used before. 
> 
>**The list is sorted to have the most-allocated arenas first(ascending order based on the nfreepools member).**
> 
>**This means that the next allocation will come from a heavily used arena,
> which gives the nearly empty arenas a chance to be returned to the system.
> In my unscientific tests this dramatically improved the number of arenas
> that could be freed.**

<font color=green>annotation about memory pools </font>

> **Pool table -- headed, circular, doubly-linked lists of partially used pools.**
> 
> This is involved.  For an index i, usedpools[i+i] is the header for a list of
> all partially used pools holding small blocks with "size class idx" i. So
> usedpools[0] corresponds to blocks of size 8, usedpools[2] to blocks of size
> 16, and so on:  index 2*i <-blocks of size (i+1)<<ALIGNMENT_SHIFT.
> 
> Pools are carved off an arena's highwater mark (an arena_object's pool_address
> member) as needed.  Once carved off, a pool is in one of three states forever
> after:
> 
> **used** == partially used, neither empty nor full
>     At least one block in the pool is currently allocated, and at least one
>     block in the pool is not currently allocated (note this implies a pool
>     has room for at least two blocks).
>     This is a pool's initial state, as a pool is created only when malloc
>     needs space.
>     The pool holds blocks of a fixed size, and is in the circular list headed
>     at usedpools[i] (see above).  It's linked to the other used pools of the
>     same size class via the pool_header's nextpool and prevpool members.
>     If all but one block is currently allocated, a malloc can cause a
>     transition to the full state.  If all but one block is not currently
>     allocated, a free can cause a transition to the empty state.
> 
> **full** == all the pool's blocks are currently allocated
>     On transition to full, a pool is unlinked from its usedpools[] list.
>     It's not linked to from anything then anymore, and its nextpool and
>     prevpool members are meaningless until it transitions back to used.
>     A free of a block in a full pool puts the pool back in the used state.
>     Then it's linked in at the front of the appropriate usedpools[] list, so
>     that the next allocation for its size class will reuse the freed block.
> 
> **empty** == all the pool's blocks are currently available for allocation
>     On transition to empty, a pool is unlinked from its usedpools[] list,
>     and linked to the front of its arena_object's singly-linked freepools list,
>     via its nextpool member.  The prevpool member has no meaning in this case.
>     Empty pools have no inherent size class:  the next time a malloc finds
>     an empty list in usedpools[], it takes the first pool off of freepools.
>     If the size class needed happens to be the same as the size class the > pool
>     
> **Block Management**
> 
> Blocks within pools are again carved out as needed.  pool->freeblock points to
> the start of a singly-linked list of free blocks within the pool.  When a
> block is freed, it's inserted at the front of its pool's freeblock list.  Note
> that the available blocks in a pool are *not* linked all together when a pool
> is initialized.  Instead only "the first two" (lowest addresses) blocks are
> set up, returning the first such block, and setting pool->freeblock to a
> one-block list holding the second such block.  This is consistent with that
> pymalloc strives at all levels (arena, pool, and block) never to touch a piece
> of memory until it's actually needed.
> 
> So long as a pool is in the used state, we're certain there *is* a block
> available for allocating, and pool->freeblock is not NULL.  If pool->freeblock
> points to the end of the free list before we've carved the entire pool into
> blocks, that means we simply haven't yet gotten to one of the higher-address
> blocks.  The offset from the pool_header to the start of "the next" virgin
> block is stored in the pool_header nextoffset member, and the largest value
> of nextoffset that makes sense is stored in the maxnextoffset member when a
> pool is initialized.  All the blocks in a pool have been passed out at least
> once when and only when nextoffset maxnextoffset.
> 
> 
> Major obscurity:  While the usedpools vector is declared to have poolp
> entries, it doesn't really.  It really contains two pointers per (conceptual)
> poolp entry, the nextpool and prevpool members of a pool_header.  The
> excruciating initialization code below fools C so that
> 
> usedpool[i+i]
> 
> "acts like" a genuine poolp, but only so long as you only reference its
> nextpool and prevpool members.  The "- 2*sizeof(block *)" gibberish is
> compensating for that a pool_header's nextpool and prevpool members
> immediately follow a pool_header's first two members:
> 
> union { block *_padding;
> uint count; } ref;
> block *freeblock;
> each of which consume sizeof(block *) bytes.  So what usedpools[i+i] > really
> contains is a fudged-up pointer p such that *if* C believes it's a poolp
> pointer, then p->nextpool and p->prevpool are both p (meaning that the headed
> circular list is empty).
> 
> It's unclear why the usedpools setup is so convoluted.  It could be to
> minimize the amount of cache required to hold this heavily-referenced table
> (which only *needs* the two interpool pointer members of a pool_header). OTOH,
> referencing code has to remember to "double the index" and doing so isn't
> free, **usedpools[0] isn't a strictly legal pointer, and we're crucially relying**
> on that C doesn't insert any padding anywhere in a pool_header at or before
> the prevpool member.


<font color=red>
gc add 3/14/2017 7:13:19 PM
actully, this is very tricky. To cheat C compile every element is poolp type, but the realy situation is: 
</font>
![memory allocation in cpython](https://www.processon.com/chart_image/58c7cf10e4b0897e6c31bfe7.png)

----------

----------

#*<font color=blue>2.key data_struct</font>*#

<font color=green>gc add 3/14/2017 11:03:34 AM :

some structure and marcos we should know
</font>

    #size_t 32
    #define PY_SSIZE_T_MAX ((Py_ssize_t)(((size_t)-1)>>1))  #31
    #define SST SIZEOF_SIZE_T                               # 8
    #define SIZEOF_SIZE_T           8
    #define ALIGNMENT               8               /* must be 2^N */  
    #define ALIGNMENT_SHIFT         3
    #define ALIGNMENT_MASK          (ALIGNMENT - 1)
    #define INDEX2SIZE(I) (((uint)(I) + 1) << ALIGNMENT_SHIFT)
    #define SMALL_REQUEST_THRESHOLD 256
    #define NB_SMALL_SIZE_CLASSES   (SMALL_REQUEST_THRESHOLD / ALIGNMENT)
    #define SMALL_MEMORY_LIMIT      (64 * 1024 * 1024)      /* 64 MB -- more? */

    #define ARENA_SIZE              (256 << 10)     /* 256KB */
    #define POOL_SIZE               SYSTEM_PAGE_SIZE  4kB

    #define FORBIDDENBYTE  0xFB

<font color=green>**struct about Anera**</font>
    
    /* Record keeping for arenas. */
    struct arena_object {

    uptr address;

    /* Pool-aligned pointer to the next pool to be carved off. */
    block* pool_address;

    /* The number of available pools in the arena:  free pools + never-
     * allocated pools.
     */
    uint nfreepools;

    /* The total number of pools in the arena, whether or not available. */
    uint ntotalpools;

    /* Singly-linked list of available pools. */
    struct pool_header* freepools;

    struct arena_object* nextarena;
    struct arena_object* prevarena;
    };

    /* Array of objects used to track chunks of memory (arenas). */
    static struct arena_object* arenas = NULL;
    /* Number of slots currently allocated in the `arenas` vector. */
    static uint maxarenas = 0;

     /* How many arena_objects do we initially allocate?
     16 = can allocate 16 arenas = 16 * ARENA_SIZE = 4MB before growing the
     `arenas` vector.
     */
     #define INITIAL_ARENA_OBJECTS 16


<font color=green>
gc add 3/15/2017 6:27:59 PM 

**management of Anera, distinguish associated(doubly-linked and order by nfreepools asc) from un associated(single-link)**
</font>

>     /* Whenever this arena_object is not associated with an allocated
>      * arena, the nextarena member is used to link all unassociated
>      * arena_objects in the singly-linked `unused_arena_objects` list.
>      * The prevarena member is unused in this case.
>      *
>      * When this arena_object is associated with an allocated arena
>      * with at least one available pool, both members are used in the
>      * doubly-linked `usable_arenas` list, which is maintained in
>      * increasing order of `nfreepools` values.
>      *
>      * Else this arena_object is associated with an allocated arena
>      * all of whose pools are in use.  `nextarena` and `prevarena`
>      * are both meaningless in this case.
>      */




<font color=green>**struct about pool**</font>

    /* When you say memory, my mind reasons in terms of (pointers to) blocks */
    #define uchar   unsigned char
    typedef uchar block;

    /* Pool for small blocks. */
    struct pool_header {
    union { block *_padding;
            uint count; } ref;          /* number of allocated blocks    */
    block *freeblock;                   /* pool's free list head         */
    struct pool_header *nextpool;       /* next pool of this size class  */
    struct pool_header *prevpool;       /* previous pool       ""        */
    uint arenaindex;                    /* index into arenas of base adr */
    uint szidx;                         /* block size class index        */
    uint nextoffset;                    /* bytes to virgin block         */
    uint maxnextoffset;                 /* largest valid nextoffset      */
    };

    typedef struct pool_header *poolp;


<font color=green>**struct about usedpools**</font>

    #define PTA(x)  ((poolp )((uchar *)&(usedpools[2*(x)]) - 2*sizeof(block *)))
    #define PT(x)   PTA(x), PTA(x)

    static poolp usedpools[2 * ((NB_SMALL_SIZE_CLASSES + 7) / 8) * 8] = {
    PT(0), PT(1), PT(2), PT(3), PT(4), PT(5), PT(6), PT(7)
    #if NB_SMALL_SIZE_CLASSES > 8
    , PT(8), PT(9), PT(10), PT(11), PT(12), PT(13), PT(14), PT(15)
    #if NB_SMALL_SIZE_CLASSES > 16
    , PT(16), PT(17), PT(18), PT(19), PT(20), PT(21), PT(22), PT(23)
    #if NB_SMALL_SIZE_CLASSES > 24
    , PT(24), PT(25), PT(26), PT(27), PT(28), PT(29), PT(30), PT(31)



<font color=green>**marco for malloc(implemented in Include/pymem.h)**</font>

<font color=gren>
gc add 3/20/2017 4:08:53 PM 

there are two ways for memory allocator, one is pymalloc , the other is malloc using c lib, they are distinguish from macro parameter.


</font>

    #ifdef PYMALLOC_DEBUG
    /* Redirect all memory operations to Python's debugging allocator. */
    #define PyMem_MALLOC_PyMem_DebugMalloc
    #define PyMem_REALLOC   _PyMem_DebugRealloc
    #define PyMem_FREE  _PyMem_DebugFree
    
    #else   /* ! PYMALLOC_DEBUG */
    
    /* PyMem_MALLOC(0) means malloc(1). Some systems would return NULL
       for malloc(0), which would be treated as an error. Some platforms
       would return a pointer with no memory behind it, which would break
       pymalloc. To solve these problems, allocate an extra byte. */
    /* Returns NULL to indicate error if a negative size or size larger than
       Py_ssize_t can represent is supplied.  Helps prevents security holes. */
    #define PyMem_MALLOC(n) ((size_t)(n) > (size_t)PY_SSIZE_T_MAX ? NULL \
    : malloc((n) ? (n) : 1))
    #define PyMem_REALLOC(p, n) ((size_t)(n) > (size_t)PY_SSIZE_T_MAX  ? NULL \
    : realloc((p), (n) ? (n) : 1))
    #define PyMem_FREE  free
    
    #endif  /* PYMALLOC_DEBUG */




----------

----------

#*<font color=blue>3.kernel code</font>*#


<font color=green>**management of arena**</font>

<font color=gren>
gc: add 3/17/2017 5:08:03 PM 

actully, there are two ways to implement this memory management.

1. like cpython, use realloc to get new location of memory. Uauslly, you can apply more then you need ,like double original or plus a const size bigger.
2. use linked list. Apply for new node of stuct , and insert new node in the list.Perhaps. this method is using in the situation which we needn`t sequence memory.

see this discussion below:

[https://groups.google.com/forum/#!topic/comp.lang.c/keLL2G-sCpQ](https://groups.google.com/forum/#!topic/comp.lang.c/keLL2G-sCpQ)

And, cpython choose first.

</font>

>     void* realloc (void* ptr, size_t size);
> Reallocate memory block
> Changes the size of the memory block pointed to by ptr.
> 
> The function may move the memory block to a new location (whose address is returned by the function).
> 
> The content of the memory block is **preserved up to the lesser of the new and old sizes**, even if the block is moved to a new location. If the new size is larger, the value of the newly allocated portion is indeterminate.


    static struct arena_object* new_arena(void)
    {
    struct arena_object* arenaobj;
    uint excess;        /* number of bytes above pool alignment */

    #ifdef PYMALLOC_DEBUG
    if (Py_GETENV("PYTHONMALLOCSTATS"))
        _PyObject_DebugMallocStats();
    #endif
    if (unused_arena_objects == NULL) {
        uint i;
        uint numarenas;
        size_t nbytes;

        /* Double the number of arena objects on each allocation.
         * Note that it's possible for `numarenas` to overflow.
         */
        numarenas = maxarenas ? maxarenas << 1 : INITIAL_ARENA_OBJECTS;
        if (numarenas <= maxarenas)
            return NULL;                /* overflow */
    #if SIZEOF_SIZE_T <= SIZEOF_INT
        if (numarenas > PY_SIZE_MAX / sizeof(*arenas))
            return NULL;                /* overflow */
    #endif
        nbytes = numarenas * sizeof(*arenas);
        arenaobj = (struct arena_object *)realloc(arenas, nbytes);
        if (arenaobj == NULL)
            return NULL;
        arenas = arenaobj;

                assert(usable_arenas == NULL);
        assert(unused_arena_objects == NULL);

        /* Put the new arenas on the unused_arena_objects list. */
        for (i = maxarenas; i < numarenas; ++i) {
            arenas[i].address = 0;              /* mark as unassociated */
            arenas[i].nextarena = i < numarenas - 1 ?
                                   &arenas[i+1] : NULL;
        }

        /* Update globals. */
        unused_arena_objects = &arenas[maxarenas];
        maxarenas = numarenas;
    }

         /* Take the next available arena object off the head of the list. */
    assert(unused_arena_objects != NULL);
    arenaobj = unused_arena_objects;
    unused_arena_objects = arenaobj->nextarena;
    assert(arenaobj->address == 0);

<font color=red>
gc: add 3/20/2017 11:33:54 AM 
i don`t queit understand here. we`ve already got engouh space, have we? if so, why still malloc a new space for aerna?

gc:add 3/20/2017 2:35:36 PM 

   arenaobj = (struct arena_object *)realloc(arenas, nbytes)

   only apply space for arena_object, which is header of real arena. there is onlye one problem, if realloc failed  ,python will nerver apply for new space?what we can then ?

</font>

    # define ARENA_SIZE              (256 << 10)  256KB
    arenaobj->address = (uptr)malloc(ARENA_SIZE);  
    if (arenaobj->address == 0) {
        /* The allocation failed: return NULL after putting the
         * arenaobj back.
         */
        arenaobj->nextarena = unused_arena_objects;
        unused_arena_objects = arenaobj;
        return NULL;
    }

        ++narenas_currently_allocated;
    #ifdef PYMALLOC_DEBUG
    ++ntimes_arena_allocated;
    if (narenas_currently_allocated > narenas_highwater)
        narenas_highwater = narenas_currently_allocated;
    #endif
    arenaobj->freepools = NULL;

<font color=gren>
gc add 3/20/2017 11:33:54 AM 


if excess == 0 ,which means the first pool start with the address of arena, then where should we store variant like nfreepools?\

gc add  3/20/2017 2:43:13 PM 

variant like nfreepools actully stroed in the arena_object not inner Arena. Arena is onlye for pools (besides excess space ). so arenaobj->address pointer to the header of Aerna space, and arenaobj->pool_address pointer to aligned address of header of pools, while the arenaobj->freepools is the one pointer to next freepool address.   

</font>

    /* pool_address <- first pool-aligned address in the arena
       nfreepools <- number of whole pools that fit after alignment */
    arenaobj->pool_address = (block*)arenaobj->address;
    arenaobj->nfreepools = ARENA_SIZE / POOL_SIZE;
    assert(POOL_SIZE * arenaobj->nfreepools == ARENA_SIZE);
    #POOL_SIZE_MASK= 0xFFF 
    excess = (uint)(arenaobj->address & POOL_SIZE_MASK);
    if (excess != 0) {
        --arenaobj->nfreepools;
        arenaobj->pool_address += POOL_SIZE - excess;
    }
    arenaobj->ntotalpools = arenaobj->nfreepools;

    return arenaobj;
    }


<font color=green>**obmalloc space check**</font>

<font color=gren>
gc add 3/20/2017 3:23:52 PM 

a test marco for addres check . The overwhelming advantage is
that this test determines whether an arbitrary address is controlled by
obmalloc in a small constant time, independent of the number of arenas
obmalloc controls.

there are three conditions:


1. **POOL->arenaindex < maxarenas;**
2. **P - arenas[POOL->arenaindex].address < ARENA_SIZE**
3. **arena[POOL->arenaindex].addres ! = 0** 

</font>

    #define Py_ADDRESS_IN_RANGE(P, POOL)                    \
    ((arenaindex_temp = (POOL)->arenaindex) < maxarenas &&              \
     (uptr)(P) - arenas[arenaindex_temp].address < (uptr)ARENA_SIZE && \
     arenas[arenaindex_temp].address != 0)




<font color=green>**malloc(implemented in Object/object.c)**</font>

    void *
    PyMem_Malloc(size_t nbytes)
    {
        return PyMem_MALLOC(nbytes);
    }
  


<font color=green>**malloc function**</font>

<font color=gren>
gc add 3/20/2017 4:33:45 PM 

Steps for pymalloc

1. determine size for usedpools;
2. get pool form usedpools[size+size]
3. if pool != pool->nextpool, means still has pool for this size. we chech pool-freeblock, if it is not NULL ,return this block; if it is NULL, check whether nextoff<=maxoff(maybe some offset block have been free?),use its block;otherwise, pool is full,has no space for new obj,use pool->nextpool,but return NULL?
4. if pool == pool->nextpool, means has no pool for this size. we need to apply new pool for aernas
5. if usable_arenas == NULL; call new_arena(), get usable_arenas, then pool = usable_arenas->freepools
6. pool != NULL, check usbale_arenas->nfreepools == 0? if true to set usable_arenas = usable_arenas->nextarena; if false, do nothing;then, frontlink this pool in usedpoos[size + size]i; initialize this pool ,set  pool->szidx,pool->nextoffset\maxoffset,freeblock
7. pool == NULL, carve one pool from usable_arenas, as usable_arenas must have at least one pool, otherwise the arena could not be called as usable as the definition description.
</font>

![PyObject_Malloc function](https://www.processon.com/chart_image/58cfc0d1e4b0fdc355ae13ed.png)



----------


    #ifdef WITH_PYMALLOC
    void * PyObject_Malloc(size_t nbytes)
    {
    block *bp;
    poolp pool;
    poolp next;
    uint size;

    #ifdef WITH_VALGRIND
    if (UNLIKELY(running_on_valgrind == -1))
        running_on_valgrind = RUNNING_ON_VALGRIND;
    if (UNLIKELY(running_on_valgrind))
        goto redirect;
    #endif 

    /*
     * Limit ourselves to PY_SSIZE_T_MAX bytes to prevent security holes.
     * Most python internals blindly use a signed Py_ssize_t to track
     * things without checking for overflows or negatives.
     * As size_t is unsigned, checking for nbytes < 0 is not required.
     */
    if (nbytes > PY_SSIZE_T_MAX)
        return NULL;

    /*
     * This implicitly redirects malloc(0).
     */
    if ((nbytes - 1) < SMALL_REQUEST_THRESHOLD) {
        LOCK();
        /*
         * Most frequent paths first
         */
        size = (uint)(nbytes - 1) >> ALIGNMENT_SHIFT;
        pool = usedpools[size + size];
        if (pool != pool->nextpool) {
            /*
             * There is a used pool for this size class.
             * Pick up the head block of its free list.
             */
            ++pool->ref.count;
            bp = pool->freeblock;
            assert(bp != NULL);
            if ((pool->freeblock = *(block **)bp) != NULL) {
                UNLOCK();
                return (void *)bp;
            }
            /*
             * Reached the end of the free list, try to extend it.
             */
            if (pool->nextoffset <= pool->maxnextoffset) {
                /* There is room for another block. */
                pool->freeblock = (block*)pool +
                                  pool->nextoffset;
                pool->nextoffset += INDEX2SIZE(size);
                *(block **)(pool->freeblock) = NULL;
                UNLOCK();
                return (void *)bp;
            }
            /* Pool is full, unlink from used pools. */
            next = pool->nextpool;
            pool = pool->prevpool;
            next->prevpool = pool;
            pool->nextpool = next;
            UNLOCK();
            return (void *)bp;
        }

        /* There isn't a pool of the right size class immediately
         * available:  use a free pool.
         */
    


----------

    # else  /* not WITH_PYMALLOC*/

    void * PyObject_Malloc(size_t n)
    {
        return PyMem_MALLOC(n);
    }

    


----------

    void * _PyMem_DebugMalloc(size_t nbytes)
    {
        return _PyObject_DebugMallocApi(_PYMALLOC_MEM_ID, nbytes);
    }



<font color=green>**realloc function**</font>





<font color=gren>
gc add 3/21/2017 11:16:22 AM 

there two situation:

1.nbytes < size(of origin block, which means shrinkage). if nbytes>75%size, we can tolerant this waste of memory, return origin pointor; else nbytes<75%size, apply new block bp with nbyte,memcpy(bp,p,nbytes)

2.nbytes > size , apply new block bp with nbyte,memcpy(bp,p,nbytes)


</font>

    void * PyObject_Realloc(void *p, size_t nbytes)


----------


    #define PyMem_REALLOC           _PyMem_DebugRealloc

    _PyMem_DebugRealloc(void *p, size_t nbytes)
    {
    return _PyObject_DebugReallocApi(_PYMALLOC_MEM_ID, p, nbytes);
    }


----------

<font color=green>**free obj**</font>

<font color=gren>
gc add 3/20/2017 8:31:21 PM 
free one obj,usually cause 5 situation:

1. aerna of the pool of the block may need free,if the nfreepools == ntotalpools;
2. aerna of the pool of the block may be avaliable , as the nfreepools ==1
3. the order of aerna in usbale_arenas may need modify ,as the nfreepools inc;
4. pool of the block is avaliable again, need to be inserted into usedpools, frontlink it to the size of userdpool; **this stragety mimics LRU**
5. nothing to do;


</font>


----------

<font color=green>debug mode</font>

> Let S = sizeof(size_t).  The debug malloc asks for 4*S extra bytes and
>    fills them with useful stuff, here calling the underlying malloc's result p:
> 
> 1. p[0: S] Number of bytes originally asked for.  This is a size_t, big-endian (easier to read in a memory dump).
> 1. p[S: 2*S] Copies of FORBIDDENBYTE.  Used to catch under- writes and reads.
> 1. p[2*S: 2*S+n] The requested memory, filled with copies of CLEANBYTE.Used to catch reference to uninitialized memory.&p[2*S] is returned.  Note that this is 8-byte aligned if pymallochandled the request itself.
> 1. p[2*S+n: 2*S+n+S] Copies of FORBIDDENBYTE.  Used to catch over- writes and reads.
> 1. p[2*S+n+S: 2*S+n+2*S] A serial number, incremented by 1 on each call to _PyObject_DebugMalloc and _PyObject_DebugRealloc.This is a big-endian size_t.If "bad memory" is detected later, the serial number gives an excellent way to set a breakpoint on the next run, to capture the instant at which this block was passed out.


----------


    void * _PyObject_DebugMallocApi(char id, size_t nbytes)
    {
    uchar *p;           /* base address of malloc'ed block */
    uchar *tail;        /* p + 2*SST + nbytes == pointer to tail pad bytes */
    size_t total;       /* nbytes + 4*SST */

    bumpserialno();    // get new serialno, serialno is static number
    total = nbytes + 4*SST;
    if (total < nbytes)
        /* overflow:  can't represent total as a size_t */
        return NULL;

    p = (uchar *)PyObject_Malloc(total);  // apply for total=nbyte +32 bit
    if (p == NULL)
        return NULL;

    /* at p, write size (SST bytes), id (1 byte), pad (SST-1 bytes) */
    write_size_t(p, nbytes);
    p[SST] = (uchar)id;
    memset(p + SST + 1 , FORBIDDENBYTE, SST-1);   #0xFB

    if (nbytes > 0)
        memset(p + 2*SST, CLEANBYTE, nbytes);     #0xCB

    /* at tail, write pad (SST bytes) and serialno (SST bytes) */
    tail = p + 2*SST + nbytes;
    memset(tail, FORBIDDENBYTE, SST);
    write_size_t(tail + SST, serialno);

    return p + 2*SST;
    }

----------

    void * _PyObject_DebugReallocApi(char api, void *p, size_t nbytes)
    {
    uchar *q = (uchar *)p;
    uchar *tail;
    size_t total;       /* nbytes + 4*SST */
    size_t original_nbytes;
    int i;

    if (p == NULL)
        return _PyObject_DebugMallocApi(api, nbytes);

    _PyObject_DebugCheckAddressApi(api, p);
    bumpserialno();
    original_nbytes = read_size_t(q - 2*SST);
    total = nbytes + 4*SST;
    if (total < nbytes)
        /* overflow:  can't represent total as a size_t */
        return NULL;

    if (nbytes < original_nbytes) {
        /* shrinking:  mark old extra memory dead */
        memset(q + nbytes, DEADBYTE, original_nbytes - nbytes + 2*SST);
    }

    /* Resize and add decorations. We may get a new pointer here, in which
     * case we didn't get the chance to mark the old memory with DEADBYTE,
     * but we live with that.
     */
    q = (uchar *)PyObject_Realloc(q - 2*SST, total);
    if (q == NULL)
        return NULL;

    write_size_t(q, nbytes);
    assert(q[SST] == (uchar)api);
    for (i = 1; i < SST; ++i)
        assert(q[SST + i] == FORBIDDENBYTE);
    q += 2*SST;
    tail = q + nbytes;
    memset(tail, FORBIDDENBYTE, SST);
    write_size_t(tail + SST, serialno);

    if (nbytes > original_nbytes) {
        /* growing:  mark new extra memory clean */
        memset(q + original_nbytes, CLEANBYTE,
               nbytes - original_nbytes);
    }

    return q;
    }


----------

<font color=gren>
gc add 3/21/2017 12:03:34 PM 

1. check api (api must equal id writer in p[-SST])
2. check p[-SST+1:-1] each Byte must be FORBIDDENBYTE
3. get length of p from p[-2SST], where record the nbyte of p
4. check p[nbytes: nbytes+ST] ,which should be FORBIDDENBYTE





</font>

     void _PyObject_DebugCheckAddressApi(char api, const void *p)

----------

<font color=gren>

print all memory stats runtime, including:

1. size in userdpool (every 8B inc in a slot), and total pool for each size, and used/ avaliable block for each size;
2. usable aerna
3. free pools in eachi aerna


 
</font>

     void _PyObject_DebugMallocStats(void)


----------


<font color=green>**lock**</font>

> Python's threads are serialized, so object malloc locking is disabled

    #define as nothing
    #define SIMPLELOCK_DECL(lock)   /* simple lock declaration  */
    #define SIMPLELOCK_INIT(lock)   /* allocate (if needed) and initialize  */
    #define SIMPLELOCK_FINI(lock)   /* free/destroy an existing lock*/
    #define SIMPLELOCK_LOCK(lock)   /* acquire released lock */
    #define SIMPLELOCK_UNLOCK(lock) /* release acquired lock */

