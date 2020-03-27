# Choosing Strings

## Preamble
This note is part of a series of notes that gives complete end-to-end examples
of using Interface Types to export and import functions.

## Introduction

Keeping track of memory that is allocated in order to allow string processing
can be challenging. This example illustrates an extreme case that involves
non-determinism.

This scenario is based on the idea of using run-time information in order to
control the flow of information. In particular, consider the C++ chooser
function:

```C++
typedef std::shared_ptr<std::string> shared_string;

shared_string nondeterministic_choice(shared_string one, shared_string two) {
  return random() > 0.5 ? std::move(one) : std_move(two);
}
```

This function takes two string arguments and returns one of them. Regardless of
the merits of this particular function, it sets up significant challenges should
we want to expose it using Interface Types. Specifically, there are two
resources entering the function, with just one leaving. However, when exposed as
an Interface Type function, all these resources must be created and properly
disposed of within the adapter code itself.

This note focuses on the techniques that enable this to be achieved reliably.

## Exporting the Chooser

The Interface Type function signature for our chooser is simple: the function
takes two `string`s and returns one:

```wasm
(@interface func (export "chooser")
  (param $left string)
  (param $right string)
  (result string)
  ...
)
```

One of the specific challenges in this scenario is the handling of shared
pointers. We 'require' that the core `nondeterministic_choice` function honor
the semantics of proper reference counting the input arguments and the returned
string.

However, there are actually _two_ schemes in play in this scenario: the
Interface Types management of resources and the C++ implementation of
`shared_ptr`. Effectively, in addition to lifting and lowering the `string`
values we must also lift and lower the ownership between Interface Types and
core WASM.

In this scenario we take a somewhat simplified view of C++'s implementation of
shared pointers: a shared pointer is implemented as a pair consisting of a
reference count and a raw pointer to the shared resource.

>Note: In practice, the C++ implementation of shared pointers is somewhat more
>complex; for reasons that are not important here.

This results in _two_ memory allocations for the shared resource: one for the
resource itself and one for the pointer structure -- which contains a reference
count and a raw pointer to the resource.


```wasm
(@interface func $stralloc-x
  (param $str string)
  (result owned<i32>)        ;; return an owned shared ptr of the string
  local.get $str
  string.size
  call-export "malloc-x"     ;; create string entity of right size
  own (i32)
    call-export "free-x"     ;; temporary, until safely re-owned
  end
  let (local $ptr owned<i32>) (result owned<i32>)
    local.get $str
    local.get $ptr
	owned.access             ;; access the owned pointer
	local.get $str
	string.size
    utf8.from.string
	local.get $ptr
  end
)

(@interface func $shared-x
  (param $ptr owned<i32>)
  (result owned<i32>)        ;; return a shared ptr of the entity

  local.get $ptr
  owned.release              ;; access pointer and remove ownership
  i32.const 1                ;; initial reference count of 1
  call-export "shared-builder-x" ;; make it a shared ptr pair
  own (i32)                  ;; re-own it, with a different destructor
    call-export "shared-release-x"
  end
end
)

(@interface func (export "chooser")
  (param $left string)
  (param $right string)
  (result string)

  local.get $left
  invoke-func $stralloc-x
  invoke-func $shared-x     ;; make it a shared ptr
  let (local $lpo owned<i32>) (result string)
    local.get $right
    invoke-func $stralloc-x
	invoke-func $shared-x   ;; make it a shared ptr
    let (local $rp owned<i32>) (result string)
      ;; set up the call to the core chooser 
      local.get $lp
	  owned.access
      local.get $rp
	  owned.access
      call-export "nondeterministic_choice"  ;; call the chooser itself
      own (i32)               ;; own the result
        call-export "shared-release-x"    ;; will eventually call to free string
      end		
      owned.access
	  call-export "shared-get-x"
	  call-export "access-utf8-x"
      string.from.utf8
    end
  end
)
```

For clarity, we separate out two helper functions -- `shared-x` and `stralloc-x`
-- whose role is to create a shared pointer pair and to instantiate a memory
string from an Interface Types's `string` entity.

>Note the `-x` suffix signals that these functions are part of the exporting
>module.

>Note, like the import and export adapters themselves, they will be inlined as
>part of the adapter fusion process.

The returned value from `nondeterministic_choice` is wrapped up as an `own`ed
allocation and lifted to a `string`.

>Note we do not need to `own` the string memory of the arguments because we have
>asserted that both the arguments to `nondeterministic_choice` will be the
>_last_ references to the strings. However, only one of the C++ strings will be
>deallocated -- the other is returned to us. We _do_ need to `own` the return
>result, however.

In addition to the Interface Types memory management, because we are using a
non-trivial C++ structure, we have to invoke the appropriate constructors,
access functions and destructors of our `shared_ptr` structure.

This is achieved through some additional core helper functions that are exported
by the core wasm module, although they would not typically be exported as
Interface Type functions:

* `shared-builder-x` is used to create and initialize a `shared_ptr`
  structure. It takes a pointer as an argument and returns a `shared_ptr` with
  an initial reference count of 1.

* `shared-release-x` is used to decrement the reference count of a `shared_ptr`;
  and if that results in the reference count dropping to zero _both the
  underlying resource and the `shared_ptr` structure itself are disposed of_.

* `shared-get-x` is used to access the underlying resource of a `shared_ptr`
  without affecting it's reference count.

* `access-utf8-x` is used to access the memory pointer for a string's text and
  its length.

In practice there would likely be additional support functions -- such as a
version of accessing a shared resource whilst incrementing reference
count. However, we do not need them in this scenario.

## Calling the chooser

We shall assume that the import to `nondeterministic_choice` were as though it
was from a core WebAssembly import whose signature is:

```wasm
(func (import "" "chooser_")
  (param i32 i32)
  (result i32))
```

The two strings are passed as memory addresses of structures -- from which the
text of the string and its length can be ascertained. The structure itself is
unspecified; we will use access functions -- such as `access-utf8` as needed.

>Note that although _we_ believe that the returned value will be the same as one
>of the arguments, the limitations of Interface Types mean that the returned
>string will be a copy of one of the arguments. 

The import adapter for `chooser_` has to lift the two argument `string`s and
lower the return value:

```wasm
(@interface func $stralloc-i
  (param $str string)
  (result i32)           ;; return a shared ptr of the string
  local.get $str
  string.size
  call-export "malloc-i"     ;; create string entity of right size
  let (local $ptr i32) (result i32)
    local.get $str
    local.get $ptr
	local.get $str
	string.size
    utf8.from.string
	local.get $ptr
  end
)

(@interface implement (import "" "chooser_")
  (param $l i32)
  (param $r i32)
  (result i32)

  local.get $l
  call-export "access-utf8-i" ;; get at the text & length of the left string
  string.from.utf8

  local.get $r
  call-export "access-utf8-i" ;; the right string
  string.from.utf8

  call-import "chooser"  ;; leaves a string on stack

  invoke-func "stralloc-i" ;; allocate a local string for the result
)
```

Compared to the export adapter, the import adapter is very straightforward. This
is because we require the caller -- a core wasm function -- to take
responsibility for the argument strings and for the returned string.

## Fusing adapters

The fused adapter consists of the inlined export adapter within the import
adapter; which is then simplified. In this case we have a simple import adapter
being combined with one that has some complexity:

```wasm
(@interface implement (import "" "chooser_")
  (param $l i32)
  (param $r i32)
  (result i32)

  local.get $l
  call-export "access-utf8-i"
  string.from.utf8

  local.get $r
  call-export "access-utf8-i"
  string.from.utf8

  let ($left string
       $right string) (result string)
    local.get $left
    invoke-func $stralloc-x
    invoke-func $shared-x     ;; make it a shared ptr
    let (local $lp owned<i32>) (result string)
      local.get $right
      invoke-func $stralloc-x
	  invoke-func $shared-x   ;; make it a shared ptr
      let (local $rp owned<i32>) (result string)
        ;; set up the call to the core chooser 
        local.get $lp
		owned.access
        local.get $rp
		owned.access
        call-export "nondeterministic_choice"  ;; call the chooser itself
        own (i32)               ;; own the result
          call-export "shared-release-x" 
        end
        owned.access
        call-export "shared-get-x"
        call-export "access-utf8-x"
        string.from.utf8
      end
	end	
  end
  invoke-func "stralloc-i" ;; allocate a local string for the result
)
```

After we expand the helper functions, we get:

```wasm2
(@interface implement (import "" "chooser_")
  (param $l i32)
  (param $r i32)
  (result i32)

  local.get $l
  call-export "access-utf8-i"
  string.from.utf8

  local.get $r
  call-export "access-utf8-i"
  string.from.utf8

  let ($left string
       $right string)
    local.get $left
    string.size
    call-export "malloc-x"     ;; create string entity of right size
	own (i32)                  ;; until we can safely package
      call-export "free-x"
	end
    let (local $ptr owned<i32>)(result owned<i32>)
      local.get $left
      local.get $ptr
	  owned.access
      local.get $left
      string.size
      utf8.from.string
      local.get $ptr
    end
    owned.release              ;; release the owned
    i32.const 1                ;; initial reference count of 1
    call-export "shared-builder-x" ;; make it a shared ptr pair
    own (i32)
      call-export "shared-release-x"
    end
    let (local $lp i32) (result string)
      local.get $right
	  string.size
      call-export "malloc-x"     ;; create string entity of right size
      own (i32)                  ;; until we can safely package
        call-export "free-x"
	  end
      let (local $ptr owned<i32>)(result owned<i32>)
        local.get $right
        local.get $ptr
		owned.access
        local.get $right
        string.size
        utf8.from.string
        local.get $ptr
      end
      owned.release
      i32.const 1                ;; initial reference count of 1
      call-export "shared-builder-x" ;; make it a shared ptr pair
      own (i32)
        call-export "shared-release-x"
      end
      let (local $rp owned<i32>) (result string)
        ;; set up the call to the core chooser 
        local.get $lp
		owned.access
        local.get $rp
		owned.access
        call-export "nondeterministic_choice"  ;; call the chooser itself
        own (i32)               ;; own the result
          call-export "shared-release-x" 
        end
        owned.access
        call-export "shared-get-x"
        call-export "access-utf8-x"
        string.from.utf8
      end
    end
  end
  let (local $str string)(result i32)
    local.get $str
    string.size
    call-export "malloc-i"     ;; create string entity of right size
    let (local $ptr i32)(result i32)
      local.get $str
      local.get $ptr
	  local.get $str
      string.size
      utf8.from.string
      local.get $ptr           ;; our final return value
    end
  end
)	
```

Reordering to bring parameter use closer to definition

```wasm3
(@interface implement (import "" "chooser_")
  (param $l i32)
  (param $r i32)
  (result i32)

  local.get $l
  call-export "access-utf8-i"
  string.from.utf8

  let ($left string) (result string)
    local.get $left
    string.size
    call-export "malloc-x"     ;; create string entity of right size
	own (i32)                  ;; until we can safely package
      call-export "free-x"
	end
    let (local $ptr owned<i32>)(result owned<i32>)
      local.get $left
      local.get $ptr
	  owned.access
      local.get $left
      string.size
      utf8.from.string
      local.get $ptr
    end
    owned.release              ;; release the owned
    i32.const 1                ;; initial reference count of 1
    call-export "shared-builder-x" ;; make it a shared ptr pair
    own (i32)
      call-export "shared-release-x"
    end
    let (local $lp owned<i32>) (result string)
      local.get $r
      call-export "access-utf8-i"
      string.from.utf8
      let ($right string)(result string)
        local.get $right
        invoke-func $stralloc-x
        string.size
        call-export "malloc-x"     ;; create string entity of right size
        own (i32)                  ;; until we can safely package
          call-export "free-x"
	    end
        let (local $ptr owned<i32>)
          local.get $right
          local.get $ptr
		  owned.access
          local.get $right
          string.size
          utf8.from.string
          local.get $ptr
        end
        owned.release
        i32.const 1                ;; initial reference count of 1
        call-export "shared-builder-x" ;; make it a shared ptr pair
        own (i32)
          call-export "shared-release-x"
        end
        let (local $rp owned<i32>) (result owned<i32>)
          ;; set up the call to the core chooser 
          local.get $lp
          owned.access
          local.get $rp
          owned.access
          call-export "nondeterministic_choice"  ;; call the chooser itself
          own (i32)               ;; own the result
            call-export "shared-release-x" 
          end
          owned.access
          call-export "shared-get-x"
          call-export "access-utf8-x"
          string.from.utf8
        end
      end
	end	
  end
  let (local $str string)(result i32)
    local.get $str
    string.size
    call-export "malloc-x"     ;; create string entity of right size
    let (local $ptr i32)
      local.get $str
      local.get $ptr
	  local.get $str
      string.size
      utf8.from.string
      local.get $ptr           ;; our final return value
    end
  end
)
```

Folding and fusing the string lifting and lowering operators:


```wasm4
(@interface implement (import "" "chooser_")
  (param $l i32)
  (param $r i32)
  (result i32)

  local.get $l
  call-export "access-utf8-i" ;; return base & len
  
  let (local $lbase i32)(local $lsize i32)(result i32)
    local.get $lbase
    local.get $lsize
    call-export "malloc-x"
    own (i32) 
      call-export "free-x"
    end
    let (local $ptr owned<i32>)(result owned<i32>)
      local.get $lbase
      local.get $ptr
      owned.access
      local.get $lsize
      memory.copy "mem-i" "mem-x" ;; copy string across
      local.get $ptr
      owned.release
      i32.const 1                ;; initial reference count of 1
      call-export "shared-builder-x" ;; make it a shared ptr pair
      own (i32)                  ;; $lp is owned
        call-export "shared-release-x"
      end
    end
    let (local $lp owned<i32>) (result i32)
      local.get $r
      call-export "access-utf8-i"
      let (local $rbase i32)(local $rsize i32)(result i32)
        local.get $rbase
        local.get $rsize
        call-export "malloc-x"
        own (i32) 
          call-export "free-x"
        end
	let (local $ptr owned<i32>)(result owned<i32>)
          local.get $rbase
	  local.get $rsize
          local.get $ptr
          owned.access
	  local.get $rsize
          memory.copy "mem-i" "mem-x" ;; copy string across
          local.get $ptr
          owned.release
          i32.const 1                ;; initial reference count of 1
          call-export "shared-builder-x" ;; make it a shared ptr pair
          own (i32)
            call-export "shared-release-x"
          end
        end
        let (local $rp i32)(result i32)
          local.get $lp
          owned.access
          local.get $rp
          owned.access
          call-export "nondeterministic_choice"  ;; call the chooser itself
          own (i32)               ;; own the result
            call-export "shared-release-x" 
          end
          owned.access 
          call-export "shared-get-x"
          call-export "access-utf8-x"
          let (local $xbase i32) (local $xsize i32) (result i32)
            local.get $xsize
            call-export "malloc-i"     ;; create string entity of right size
            let (local $ptr i32)
              local.get $xbase
              local.get $ptr
              local.get $xsize
              memory.copy "mem-x" "mem-i"
              local.get $ptr           ;; our final return value
            end
          end
        end
      end
    end
  end
)	
```

Our final step is unwrapping the `own`ed blocks and moving their contents to the
correct place in the final adapter. This requires us to know where in the code
the last reference to the owned value are.

This is achieved in two phases: creating local variables that reference the
stack values captured by the `own` instructions, and then moving the `own`ed
block of instructions to the appropriate location.

Note that the various `owned.access` instructions disappear at this point.

In this case, the `$lp` and `$rp` values have no mention after the call to
`"nondterministic_choice"`, and so we can move the `"shared-release"` calls to
just after that call. Similarly, the release of the return value can be
performed after returned string has been copied into the importing module:

```wasm5
(@interface implement (import "" "chooser_")
  (param $l i32)
  (param $r i32)
  (result i32)
  (local $o1 i32) ;; left string shared-ptr memory
  (local $o2 i32) ;; right string shared-ptr memory
  (local $o3 i32)

  local.get $l
  call-export "access-utf8-i" ;; return base & len
  
  let (local $lbase i32)(local $lsize i32)(result i32)
    local.get $lbase
    local.get $lsize
    call-export "malloc-x"
    let (local $ptr i32)(result i32)
	  local.get $lbase
	  local.get $ptr
	  local.get $lsize
      memory.copy "mem-i" "mem-x" ;; copy string across
      local.get $ptr
      i32.const 1                ;; initial reference count of 1
      call-export "shared-builder-x" ;; make it a shared ptr pair
      local.tee $o1
    end
    let (local $lp i32) (result i32)
      local.get $r
      call-export "access-utf8-i"
      let (local $rbase i32)(local $rsize i32)(result i32)
        local.get $rbase
        local.get $rsize
        call-export "malloc-x"
	let (local $ptr i32)(result i32)
          local.get $rbase
	  local.get $rsize
          local.get $ptr
	  local.get $rsize
          memory.copy "mem-i" "mem-x" ;; copy string across
          local.get $ptr
          i32.const 1                ;; initial reference count of 1
          call-export "shared-builder-x" ;; make it a shared ptr pair
	  local.tee $o2
        end
        let (local $rp i32)(result i32)
          local.get $lp
          local.get $rp
          call-export "nondeterministic_choice"  ;; call the chooser itself
	  local.tee $o3
          call-export "shared-get-x"
          call-export "access-utf8-x"
          let (local $xbase i32) (local $xsize i32) (result i32)
            local.get $xsize
            call-export "malloc-i"     ;; create string entity of right size
            let (local $ptr i32)
              local.get $xbase
              local.get $ptr
              local.get $xsize
              memory.copy "mem-x" "mem-i"
              local.get $ptr           ;; our final return value
            end
          end
        end
      end
    end
  end
  local.get $o1
  call-export "shared-release-x"
  local.get $o2
  call-export "shared-release-x"
  local.get $o3
  call-export "shared-release-x" 
)
```

For simplicity, we migrated all the deallocations to the end of the fused
adapter. In some situations, for example when processing arrays, we may wish to
be more aggressive in invoking the memory release code.

Although fairly long, this fused adapter has a striaghtfoward structure: the
input strings are copied from the import memory to the export memory -- and also
wrapped as shared pointer structures as required by the signature of the
`nondeterministic_chooser` function. The resulting string is copied from the
export localtion to the import memory. And, finally, any temporary structures
allocated are released.

Note that the string returned by `nondeterministic_chooser` is not directly
freed -- using a call to `"free"` -- is _released_. This is because the return
is also a shared pointer and we are required merely to decrement the reference
count.

We also decrement the reference count of the created argument strings -- just in
case the callee keeps an additional reference to either one.

