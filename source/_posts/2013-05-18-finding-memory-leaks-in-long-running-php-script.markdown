---
layout: post
title: "Finding memory leaks in long running php script"
date: 2013-05-18 18:56
comments: true
categories:
published: false
---

Recently I solved a problem of memory leaks in a long running php script. There were things I'd like to share, which might be helpful to others.

First of all, I'll give some background infomation about the script which leaks memory. The php script is a symfony task, which is running as a consumer of a rabbitmq queue. What this script does is basically fetch available jobs from the rabbitmq queue, and process these jobs, in this case, creating an email from the data of the job and sending the email. The logic of this script is quite simple, but it makes use of a lot of php libraries, which makes it not so easy to find a memory leaks directly.

At the beginning, I looked at the problem and thought, it should not be such a big deal to fix this, then I tried to tackle the problem the quick way, which was proved the wrong way at last. I added some [memory_get_usage](http://php.net/manual/en/function.memory-get-usage.php) in some critical functions to collect the memory usage information, unfortunately, such information is not sufficient and can't provide any useful information to help me identify which part of the code cause the memory leaks. Adding more memory_get_usage is in fact doing memory profiling manually, which is not feasible and acceptable for me. There should be some tools to do such works automatically, I guessed.

Yes, there is! Then I found a great php extension, [php-memprof](https://github.com/arnaud-lb/php-memory-profiler). This extension is extremely helpful to find memory leaks in php scripts.

To use this php extension, first we need install and enable it, this is pretty simple, just follow the steps in the README, pay attention to the dependencies, make sure that you install those dependencies before you install the php extension.

After installed the extension, we can now do some simple experiments with it to get some ideas how it works.

From the README.md of the project:

> php-memprof profiles memory usage of PHP scripts, and especially can tell which function has allocated every single byte of memory currently allocated.

What does this mean? Let's look a simple example,

``` php

<?php

memprof_enable();

function f1()
{
    $arr = range(1, 1000);
}

f1();

$profile = memprof_dump_array();

print_r($profile);

```

We'll see output like:

``` php
Array
(
    [memory_size] => 32
    [blocks_count] => 1
    [memory_size_inclusive] => 32
    [blocks_count_inclusive] => 1
    [calls] => 1
    [called_functions] => Array
        (
            [f1] => Array
                (
                    [memory_size] => 0
                    [blocks_count] => 0
                    [memory_size_inclusive] => 0
                    [blocks_count_inclusive] => 0
                    [calls] => 1
                    [called_functions] => Array
                        (
                            [range] => Array
                                (
                                    [memory_size] => 0
                                    [blocks_count] => 0
                                    [memory_size_inclusive] => 0
                                    [blocks_count_inclusive] => 0
                                    [calls] => 1
                                    [called_functions] => Array
                                        (
                                        )
                                )
                        )
                )

            [memprof_dump_array] => Array
                (
                    [memory_size] => 0
                    [blocks_count] => 0
                    [memory_size_inclusive] => 0
                    [blocks_count_inclusive] => 0
                    [calls] => 1
                    [called_functions] => Array
                        (
                        )
                )
        )
)

```

The output of the profiling is pretty straightforward, it was of the same structure as the call stack of the php code executed. The most top level keys give the information of __memory usage__ of the php code by the time when _memprof\_dump\_array_ was called. Here by __memory usage__ it means the difference of the memory allocated, for example, in the example above, the value of key _memory\_size_ is 32 bytes, it means the memory usage was increased by 32 bytes since the time when _memprof\_enable_ was called.

Though there was a huge memory allocated by creating an 1000-elements array inside function _f1_, but that memory was immediately freed when function _f1_ returns, so it should not be added to memory usage.

From the value of key _memory\_size\_inclusive_, which stands for the memory used by both the code itself and code of all the functions it called, we can also come up with the same conclusion, for example, here the value of _memory\_size\_inclusive_ is also 32 bytes, which means all functions it called didn't contributes anything to the memory usage.

Let's run this example again with a slight change to the code.

``` php

<?php

memprof_enable();

function f1()
{
    global $arr;
    $arr = range(1, 1000);
}

f1();

$profile = memprof_dump_array();

print_r($profile);

```

Here we add a line in f1:

> global $arr;

The output is:

``` php
Array
(
    [memory_size] => 32
    [blocks_count] => 1
    [memory_size_inclusive] => 111403
    [blocks_count_inclusive] => 2005
    [calls] => 1
    [called_functions] => Array
        (
            [f1] => Array
                (
                    [memory_size] => 79371
                    [blocks_count] => 1004
                    [memory_size_inclusive] => 111371
                    [blocks_count_inclusive] => 2004
                    [calls] => 1
                    [called_functions] => Array
                        (
                            [range] => Array
                                (
                                    [memory_size] => 32000
                                    [blocks_count] => 1000
                                    [memory_size_inclusive] => 32000
                                    [blocks_count_inclusive] => 1000
                                    [calls] => 1
                                    [called_functions] => Array
                                        (
                                        )
                                )
                        )
                )

            [memprof_dump_array] => Array
                (
                    [memory_size] => 0
                    [blocks_count] => 0
                    [memory_size_inclusive] => 0
                    [blocks_count_inclusive] => 0
                    [calls] => 1
                    [called_functions] => Array
                        (
                        )
                )
        )
)
```

We see a big difference in the profiling output. This time the _memory\_size\_inclusive_ is much larger than _memory\_size_, the difference is caused by the reference of the $arr variable inside _f1_, here we made it a global variable, which prevent the PHP GC freed the memory. We can easily see that the code which cause the high memory usage is:

> INIT_CALL() -> f1() -> range()

We just follow the _memory\_size\_inclusive_ keys which are great than 0 to get this. Yes, it is easy to do this in simple cases. But in a more complicated cse, you will not want to do this manually. I am not going to put a real example here to prove this, instead, I am going to give another simple example to show you how to do this easily.

Let's say we have a complicated, long running php script, which leaks memory. How can we make use of php-memprof to quickly find the places that leaks memory? The structure of such php script normally is something like this:

``` php
<?php

// do some initialization, setup, etc

while(true) {

    // get a job from somewhere

    // process the job;

    // do some report, clean up, etc

    // maybe sleep some time;
}

```

First we need do some changes to add the profiling code, for example,


``` php
<?php

// enable profiling
if (extension_loaded('memprof')) {
    memprof_enable();
    $count = 0;
}

// do some initialization, setup, etc

while(true) {
    // get a job from somewhere

    // process the job;

    // do some report, clean up, etc

    // maybe sleep some time;

    // dump the profiling result every N runs
    if (extension_loaded('memprof')) {
        $count += 1;
        if ($count % N == 1) {
            $profile  = memprof_dump_array();
            $filepath = 'profile_' . $count;
            file_put_contents($filepath, serialize($profile));
        }
    }
}

```

After some time of running, we got a set of files. If there was no memory leaks, those results should be different only in the value of the key _calls_, otherwise, we should see the difference in keys like _memory\_size\*_, _blocks\_count\*_. I wrote a simple function to compare 2 results to filter out the difference which we care about.

This diff function will compare 2 profiling result recursively, give the difference. Here is a example.

``` php

<?php

function diff($a1, $a2, &$out, $call_path = array('INIT_CALL')) {
    if (empty($a1) && empty($a2)) return;

    $ms1  = isset($a1['memory_size']) ? $a1['memory_size'] : 0;
    $ms2  = isset($a2['memory_size']) ? $a2['memory_size'] : 0;

    $msi1 = isset($a1['memory_size_inclusive']) ? $a1['memory_size_inclusive'] : 0;
    $msi2 = isset($a2['memory_size_inclusive']) ? $a2['memory_size_inclusive'] : 0;

    $diff_self = $ms2 - $ms1;
    $diff_incl = $msi2 - $msi1;

    if ($diff_self !=0 || $diff_incl != 0) {
        $out[] = array(
            'call_path' => $call_path,
            'diff_self' => $diff_self,
            'diff_incl' => $diff_incl,
            );
    }

    $c1 = isset($a1['called_functions']) ? $a1['called_functions'] : array();
    $c2 = isset($a2['called_functions']) ? $a2['called_functions'] : array();
    $fns = array_unique(array_merge(array_keys($c1), array_keys($c2)));

    foreach($fns as $fn) {
        $p1 = isset($c1[$fn]) ? $c1[$fn] : array();
        $p2 = isset($c2[$fn]) ? $c2[$fn] : array();
        diff($p1, $p2, $out, array_merge($call_path, array($fn)));
    }
}

function bar()
{
    global $arr;
    $arr = range(1, 1000);
}

function foo()
{
    bar();
}

memprof_enable();
$o1 = memprof_dump_array();
foo();
$o2 = memprof_dump_array();

$out = array();
diff($o1, $o2, $out);
print_r($out);

```

The output is:

``` php

Array
(
    [0] => Array
        (
            [call_path] => Array
                (
                    [0] => INIT_CALL
                )

            [diff_self] => 106
            [diff_incl] => 111477
        )

    [1] => Array
        (
            [call_path] => Array
                (
                    [0] => INIT_CALL
                    [1] => foo
                )

            [diff_self] => 0
            [diff_incl] => 111371
        )

    [2] => Array
        (
            [call_path] => Array
                (
                    [0] => INIT_CALL
                    [1] => foo
                    [2] => bar
                )

            [diff_self] => 79371
            [diff_incl] => 111371
        )

    [3] => Array
        (
            [call_path] => Array
                (
                    [0] => INIT_CALL
                    [1] => foo
                    [2] => bar
                    [3] => range
                )

            [diff_self] => 32000
            [diff_incl] => 32000
        )
)

```

We check all the call_path key, we get all the places that leaks memory.

It is pretty simple, isn't it?
