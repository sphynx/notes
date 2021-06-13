# Command line

* `rsync` is a better `cp`. You can easily get a copy with progress bar by just using `rsync -P file_a file_b` It also has a ton of other useful functions. Originally picked it up from this post: [https://solovyov.net/blog/2011/rsync-better-cp/](https://solovyov.net/blog/2011/rsync-better-cp/)
* To get the full processor name on Mac:

```text
➜ sysctl -n machdep.cpu.brand_string
Intel(R) Core(TM) i7-3740QM CPU @ 2.70GHz
```

* To create a zero-filled file of specified size:

```bash
dd if=/dev/zero of=32mb.bin bs=32m count=1
```

* To run a program and report its maximum memory usage and other interesting things we can use usual GNU `time`\(on Mac: `brew install gnu-time` will install it as `gtime`\).

```text
➜  gtime -v target/release/c
iterator returned: 384

	Command being timed: "target/release/c"
	User time (seconds): 0.77
	System time (seconds): 0.62
	Percent of CPU this job got: 99%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:01.40
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 3125868
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 781620
	Voluntary context switches: 3
	Involuntary context switches: 112
	Swaps: 0
	File system inputs: 0
	File system outputs: 0
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
```

* Or even simpler, to just print maximum resident set size at the end: `gtime -f %M program_name`
* Convert an SVG files to multiple PNG files with different resolutions and pack them all in an ICO file:

```text
# install `svgexport` and `imagemagick`:
brew install imagemagick
npm install -g svgexport

# this worked better than ImageMagick for me:
svgexport favicon.svg 16.png 16:16
svgexport favicon.svg 32.png 32:32

# and so on, and then combine them:
convert 16.png 24.png 32.png -colors 256 favicon.ico
```

* Run a command every 5 seconds: `watch -n5 command`
* Convert GPS routes from .gpx to .kml without creating lots of points:

  ```text
  gpsbabel -i gpx -f input.gpx -o kml,points=0 -F output.kml
  ```

