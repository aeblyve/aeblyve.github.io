---
create_date: 2024-05-12 23:51:41+00:00
edit_date: 2024-05-12 23:51:41+00:00
build_date: 2024-07-06 16:05:44+00:00
title: On Optimizing Large S3 Transfers
link: https://leonid.belyaev.systems/posts/on-optimizing-large-s3-transfers.html
---


Let's say that you're lucky enough that your hardware is not a bottleneck. Disks, CPU, network, all exceed the capacity of the naive software methods. Further, let's say you're unlucky enough that the payload to transfer is large enough to where rising to the capacity of the hardware Would Be Great. Now we can talk a bit about what gratis software choices can improve performance in your case, making good use of what was paid for.

#### Fast Data Algorithms

The first thing I did was read [Joel Lynch's Excellent Blog Post](https://jolynch.github.io/posts/use_fast_data_algorithms/) on the same-similar question, which acted as my starting point. In sum, the article recommends the use of more modern tools like `xxh64sum` over `md5sum` and `zstd` over `tar` for significant benefits in speed. I ended up with a Bash snippet that looked much like the ones he lists near the end of the article, with various flags and logging added (and another one, for the download side).

#### S3 protocol tuning

As it so happens, `aws-cli` has S3 protocol defaults that are definitely not appropriate for high-performance systems.

You may observe this in seeing large files that have a disprortionately higher upload speed. While part of this effect may be the typical divide between sequential and random I/O operations (in which sequential operations are typically faster, in turn explained by cache hit rates, transaction overhead, etc), a potentially nonobvious part is S3 multiparting. For a single file, S3 imposes a 10,000 multipart limit, i.e., at most 10,000 multiparts can be used to upload it.

While the default `aws-cli` multipart size is 8MB, uploading a file over the size of 10000 * 8MB = 80GB (as might be encountered in a large transfer of scientific data) will start to affect the multipart size so as to fit inside this 10,000 part limit, which in itself has a potentially significant effect on bandwidth used, especially in the typically high-latency links to an S3 service as accessed over the Internet.

Nothing stops you from setting this multipart size value higher in the absence of such a large file, and performance-focused S3 clients such as `s5cmd` often do this by default (it uses 50MiB by default).

You can also look into the other more straightforwarding setting of `max_concurrent_requets`, which you can expect to linearly use more bandwidth when raised (up until the usual "knee point").

#### s5cmd

For very fast networks, and especially for upload of many small files, you may want to consider `s5cmd` over `aws-cli`. It is an S3-focused tool written in Go aiming to achieve exactly best performance.

Joshua Robinson wrote [an article](https://joshua-robinson.medium.com/s5cmd-for-high-performance-object-storage-7071352cc09d) about the performance gains possible. In my case, I was not able to run s5cmd because of a `ulimit` on a shared machine -- it more aggressively uses multiprocessing.

#### Get to High Ground

Depending on the bandwidth you want, you should survey your network for a position with the greatest speed. On university HPC clusters, for instance, it is not unusual for login node machines to have better internet linerates than compute nodes. However, this might present some concerns for neighborly behavior. It's best to confer with helpdesk or available documentation about what is appropriate (they might also advise you on approach w.r.t. filesystems -- for example, it is best to read off of an SSD-backed Lustre scratch system where the files are originally generated vs. the tape-backed archive system.).

#### --retry probably won't retry all errors

Over a long-enough timescale and/or when pushing the system(s) sufficiently hard, you're likely going to run into transient errors when talking to the S3 service. These can be errors semantically (429, too many requests, try again later) or brutally (connection reset).

You should know that the AWS-SDK-approved ideology here is for not all errors to be retryable. Therefore, the `--retry` flag that you stick onto your command of choice may not catch these. [The reason for this](https://github.com/aws/aws-sdk-go/issues/4793) is that it could cause non-idempotent requests to be repeated.

While S3 client implementations can make their own choices on top of the general AWS SDK, they may also go with the default behavior. I have found that both `s5cmd` and `aws-cli`, [in some cases in spite of effort](https://github.com/peak/s5cmd/issues/294), follow this default AWS SDK behavior.

Long-story-short, you may need a retry loop of your own. [This ElasticSearch PR](https://github.com/elastic/elasticsearch/pull/48560) shows one possible Bash implementation.