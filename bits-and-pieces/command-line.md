# Command line

* `rsync` is a better `cp`. You can easily get a copy with progress bar by just using `rsync -P file_a file_b` It also has a ton of other useful functions. Originally picked it up from this post: [https://solovyov.net/blog/2011/rsync-better-cp/](https://solovyov.net/blog/2011/rsync-better-cp/)
* To get full processor name on Mac:

```text
> sysctl -n machdep.cpu.brand_string
Intel(R) Core(TM) i7-3740QM CPU @ 2.70GHz
```

* Create a zero-filled file of specified size:

```bash
dd if=/dev/zero of=32mb.bin bs=32m count=1
```







