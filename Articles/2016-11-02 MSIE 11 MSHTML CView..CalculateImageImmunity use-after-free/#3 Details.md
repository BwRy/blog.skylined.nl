Details
-------
Setting the `listStyleImage` property of an Element object causes MSIE to
allocate 0x4C bytes for an "image context" structure, which contains a
reference to the document object as well as a reference to the same `CMarkup`
object as the document. When the element is removed from the document
(-fragment), this image context is freed on the next "draw". However, the code
continues to use the freed context almost immediately after it is freed.

Exploit
-------
I tried a few tricks to see if there was an easy way to reallocate the freed
memory before the reuse, but was unable to find anything. I do not know if
there is a way to cause further reuse of the freed memory later on in the code.
Running the repro as-is without page heap does not appear to trigger crashes.
It does not appear that there is enough time between the free and reuse to
exploit this issue.

Time-line
---------
* *May 2014*: This vulnerability was found through fuzzing.
* *June 2014*: This vulnerability was submitted to [ZDI][].
* *July 2014*: ZDI rejects the submission.
* *November 2016*: The issue does not reproduce in the latest build of MSIE 11.
* *November 2016*: Details of this issue are released.

Unfortunately, my records of what happened after ZDI rejected the issue are
patchy. It appears that I did not pursue reporting the issue anywhere else, but
Microsoft does appear to have patched the issue, as I can no longer reproduce
it.

[ZDI]: http://www.zerodayinitiative.com/