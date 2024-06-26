# Minimal reproducible example for failing to run a container test against a tarball using Bazel

This is an MRE for an issue I ran into recently. I originally posted on [StackOverflow][1],
but that didn't get any traction, possibly because my example was too complicated.

I found examples online that use older versions of the OCI rules.
I'm either doing something wrong or they only work because of the version difference.

* [privacysandbox/protected-auction-key-value-service][5]
* [stewartbutler/rules_debian_packages][6]

## High-level description of what I'm trying to do

I want to build container images using Bazel.
The recommended Bazel rules for this use case is [Aspect's `rules_oci`](https://github.com/bazel-contrib/rules_oci).
`rules_oci` mostly provides a way to *build* the images (which I can do successfully),
but the project recommends using [`container_structure_test`][4] to test the image structure.

>[We recommend container_structure_test to run tests against an oci_image target (with driver="docker")
or an oci_tarball target (with driver="tar").][4]


I'm reluctant to test against the `oci_image` output because it's not the final artifact.
As far as I understand, you always have to use `oci_tarball` to generate the final artifact,
so testing against `oci_image` is inherently lower-fidelity.

## Reproduction steps

I'm using an Apple Silicon Mac, but you should be able to run this on most ARM Linux machines.

### Setup

* Install [Bazelisk][2]
* [Install Docker Engine or Docker Desktop][3]
* Clone this repository somewhere.

### Successfully build and load the tarball into Docker.

Run the following commands from the repo directory. They should both succeed.

```
bazel build //my_package:empty_file_image_tarball
bazel run //my_package:empty_file_image_tarball
```

The run command should've loaded the tarball image into Docker.
You should now be able to open a bash session and see the empty file.

```
➜  mre-container-tar-test git:(main) ✗ docker run -it empty_file:latest bash
ubuntu@eacee0e03d58:/$ ls
bin  boot  dev  empty-file.txt  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
ubuntu@eacee0e03d58:/$ stat empty-file.txt
  File: empty-file.txt
  Size: 0         	Blocks: 0          IO Block: 4096   regular empty file
Device: 0,59	Inode: 527936      Links: 1
Access: (0555/-r-xr-xr-x)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2000-01-01 00:00:00.000000000 +0000
Modify: 2000-01-01 00:00:00.000000000 +0000
Change: 2024-06-14 02:55:51.692770004 +0000
 Birth: 2024-06-14 02:55:51.692770004 +0000
```

### Successfully run the non-tarball image test

Running against the intermediary image is fine.
We can see this by running `bazel test //my_package:empty_file_layer_test`.

```
➜  mre-container-tar-test git:(main) ✗ bazel test //my_package:empty_file_layer_test
INFO: Analyzed target //my_package:empty_file_layer_test (1 packages loaded, 5 targets configured).
INFO: Found 1 test target...
Target //my_package:empty_file_layer_test up-to-date:
  bazel-bin/my_package/empty_file_layer_test.sh
INFO: Elapsed time: 0.817s, Critical Path: 0.73s
INFO: 5 processes: 5 darwin-sandbox.
INFO: Build completed successfully, 5 total actions
//my_package:empty_file_layer_test                                       PASSED in 0.4s

Executed 1 out of 1 test: 1 test passes.
There were tests whose specified size is too big. Use the --test_verbose_timeout_warnings command line option to see which ones these are.
```

### Fail to run the test.

Running `bazel test //my_package:empty_file_image_tarball_test` should fail to run the test.
Bazel should complain that the tarball doesn't end in `.tar`.

```
➜  mre-container-tar-test git:(main) ✗ bazel test //my_package:empty_file_image_tarball_test
ERROR: /Users/josalvatorre/code/mre-container-tar-test/my_package/BUILD.bazel:26:25: in container_structure_test rule //my_package:empty_file_image_tarball_test:
Traceback (most recent call last):
	File "/private/var/tmp/_bazel_josalvatorre/bb1e2552732b81b1b69870fcd931a2e6/external/container_structure_test~/bazel/container_structure_test.bzl", line 55, column 17, in _structure_test_impl
		fail("when the 'driver' attribute is not 'docker', then the image must be a .tar file")
Error in fail: when the 'driver' attribute is not 'docker', then the image must be a .tar file
ERROR: /Users/josalvatorre/code/mre-container-tar-test/my_package/BUILD.bazel:26:25: Analysis of target '//my_package:empty_file_image_tarball_test' failed
ERROR: Analysis of target '//my_package:empty_file_image_tarball_test' failed; build aborted
INFO: Elapsed time: 0.079s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
ERROR: Build did NOT complete successfully
ERROR: No test targets were found, yet testing was requested
```

[1]: https://stackoverflow.com/questions/78573248/cannot-run-container-structure-test-against-oci-tarball-in-bazel-rules-oci-con
[2]: https://github.com/bazelbuild/bazelisk
[3]: https://docs.docker.com/manuals/
[4]: https://github.com/bazel-contrib/rules_oci/blob/3d43cb1a1bb2f5edc15c7f48b406be3fb225e673/README.md?plain=1#L95-L97
[5]: https://github.com/privacysandbox/protected-auction-key-value-service/blob/9a60180f9d6f52a4ca805e5463ecc9e5e80e88f9/production/packaging/aws/data_server/BUILD.bazel#L120-L132
[6]: https://github.com/stewartbutler/rules_debian_packages/blob/a7931ba880bad577a43b8274c12fcf30b7e14886/e2e/smoke/BUILD.bazel#L52-L63
