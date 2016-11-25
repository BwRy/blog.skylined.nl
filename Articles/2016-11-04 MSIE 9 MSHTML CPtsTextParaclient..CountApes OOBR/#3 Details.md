Description
-----------
After adding two elements with specific style properties during the `onload`
event handler, MSIE refreshes the layout, at which point the "content" style
causes it to update a counter, which triggers a call to
`CPtsTextParaclient::CountApes`, in which the exception happens on x86:

```x86asm
MSHTML!CPtsTextParaclient::CountApes:
    mov     edi,edi
    push    ebp
    mov     ebp,esp
    sub     esp,8
    push    ebx
    mov     ebx,dword ptr [eax+20h]
    push    esi
    lea     ecx,[eax+24h]
    push    edi
    mov     dword ptr [ebp-8],ecx
    mov     dword ptr [ebp-4],0
    test    ebx,ebx
    je      MSHTML!CPtsTextParaclient::CountApes+0x1b7
    cmp     ebx,dword ptr [ebp-8]
    je      MSHTML!CPtsTextParaclient::CountApes+0x1b3
    mov     eax,dword ptr [ebx]  ds:0023:dcbabbbb=????????
```

I enabled page-heap to make triggering the issue more reliable and get a better
idea of what is going on. To understand how, a bit of background on how page
heap works is needed. When you enable full page-heap in an application, every
heap allocation will be given its own "page". This page contains a data
structure that contains information used by page-heap to store information
about the allocation, followed by the allocated memory itself and then some
optional padding. This structure is stored at the end of the page, with the
user allocation aligned as required (hence the optional padding). This memory
page is followed by a reserved page, which causes any out-of-bounds access
immediately after the allocation to cause an access violation exception. Full
details can be found in the [Application Verifier documentation on-line][].

[Application Verifier documentation on-line]: https://msdn.microsoft.com/en-us/library/ms220938(v=vs.90).aspx]

As the documentation shows, the `0xdcbabbbb` value in `ebx` that causes the
access violation is used by page-heap as the "Prefix end magic": a marker at
the end of the structure used by page-heap to store information about the
allocation that comes immediately before the actual allocation. From the
assembly we can see that `ebx` was read from `eax + 0x20`, so it might be
interesting to ask page-heap where that points to:

```
1:020> !heap -p -a @eax
    address 0b00efb4 found in
    _DPH_HEAP_ROOT @ 51000
    in busy allocation (  DPH_HEAP_BLOCK:         UserAddr         UserSize -         VirtAddr         VirtSize)
                                 af126e8:          b00efd8               24 -          b00e000             2000
    71908e89 verifier!AVrfDebugPageHeapAllocate+0x00000229
    77c15ede ntdll!RtlDebugAllocateHeap+0x00000030
    77bda40a ntdll!RtlpAllocateHeap+0x000000c4
    77ba5ae0 ntdll!RtlAllocateHeap+0x0000023a
    683928a3 MSHTML!CGeneratedTreeNode::InitBeginPos+0x00000016
    683926b4 MSHTML!CGeneratedContent::InsertOneNode+0x00000044
    6839264d MSHTML!CGeneratedContent::CreateNode+0x000000b8
    68392be1 MSHTML!CGeneratedContent::CreateContent+0x000000d6
    68392b0b MSHTML!CGeneratedContent::ApplyContentExpressionCore+0x00000109
    681a397c MSHTML!CElement::ComputeFormatsVirtual+0x000021c9
    682e9421 MSHTML!CElement::ComputeFormats+0x000000f1
<<<snip>>>
```

This tells us that `eax` points to 0x0b00efb4, which is 0x24 bytes before the
user allocated memory at 0xb00efd8. So `eax + 0x20` must point 4 bytes before
it and tada: this is where page-heap stores the "Prefix end magic".

It seems that this method is called to operate on an object using a pointer at
an offset *before* the actually allocated memory. This does not make much sense
until you've analyzed a lot of MSIE bugs: it's quite common in MSIE for an
object to "contain" another object in memory, and for MSIE to add offsets to
pointers to find a contained object, or to subtract offsets to find the
container of such a contained object. It looks like this is the case here as
well.

Looking at the caller, CPtsTextParaclient::GetNumberApeCorners, it appears to
loop through some data structures. The call to CPtsTextParaclient::CountApes is
made in the third loop.

```x86asm
MSHTML!CPtsTextParaclient::GetNumberApeCorners+0x103
    mov     ecx,dword ptr [esi+0Ch]
    mov     eax,dword ptr [ecx]
    and     eax,1
    lea     edx,[ebp+0Ch]
    lea     eax,[eax+eax*2]
    push    edx
    lea     eax,[ecx+eax*8-24h]
    call    MSHTML!CPtsTextParaclient::CountApes
```

This code uses a pointer to a memory structure (`esi`) to find pointer to a
second structure (`ecx`). It reads a flag in `eax` and multiplies it by 0x18
(3 x 8: `eax+eax*2` and `eax*8`), then subtracts 0x24. It then adds this to
`ecx` to produce the `eax` value seen during the crash. Since the flag can be
either 0 or 1, the result in `eax` can be either `ecx - 0x24` or `ecx`.
Obviously, in this case it is the former.

It appear that the code is using the flag to determine if `ecx` is a
"stand-alone" object or a "contained" object. The bug is that either the code
is using this flag incorrectly (the flag is correct, but does not indicate the
object is a "contained" object) or the flag has been set incorrectly (the code
is correct, but the flag should not have been set as the object is not
"contained" in another object).

Exploitation
------------
Using [Heap Feng-Shui][], it may be possible to allocated a heap block
immediately before the one used in the bug and control its content in order to 
control the data the code is operating on. Unfortunately, at the time I did not
look at what the code did with the data if the access violation could be
prevented, so it's not possible for me to say exactly what an attacker might do
with this vulnerability. But one can speculate that this might allow an
attacker to have the code use some secret value (e.g. a pointer to a function
in a modules) in a way that allows him/her to retrieve the value (i.e.
information disclosure). It might be possible to have the code modify a value
located anywhere in memory, and/or have the code call/jump to a location of an
attackers choosing (i.e. arbitrary code execution).

I did not investigate the crash on x64, but I can only imagine the code is the
same, but the offsets are different.

[Heap Feng-Shui]:https://www.blackhat.com/presentations/bh-europe-07/Sotirov/Presentation/bh-eu-07-sotirov-apr19.pdf

Time-line
---------
* *June 2014*: This vulnerability was found through fuzzing.
* *August 2014*: This vulnerability was submitted to [ZDI][].
* *September 2014*: ZDI rejects the submission.
* *November 2016*: Details of this issue are released.

[ZDI]: http://www.zerodayinitiative.com/