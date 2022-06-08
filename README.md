# tlp-stress2: tlp-stress without coordinated ommission

This project is a work in progress.

The tlp-stress2 project is a fork of the [tlp-stress](https://thelastpickle.com/tlp-stress/) Cassandra benchmarking tool which adds additional metrics that more accurately the client's experienced performance of interactions with Cassandra.

Quoting from the tlp-stress README.md:

*cassandra-stress is a configuration-based tool for doing benchmarks and testing simple datamodels for Apache Cassandra.  Unfortunately it can be difficult to configure a workload.  There are faily common data models and workloads seen on Apache Cassandra, this tool aims to provide a means of executing configurable, pre-defined profiles.*

Like many benchmarks, tlp-stress doesn't correctly represent the user experience of working with Cassandra (whether the user is a human or another application). The real client experience is response time, meaning the time from when the client issues the request to when it receives a response. But tlp-stress reports mostly service time, meaning the time from when the server receives a request to the time it responds to the request. So if a client issues 10 requests which normally take 10ms and the first one freezes for 1000ms, service time will tell you you had one bad request but all the others responded in 10ms. But from the client's point of view, each of the requests took roughly 1000ms to respond, because requests 2-9 were waiting for request 1 to complete. This is classic [Coordinated Omission](https://www.youtube.com/watch?v=lJ8ydIuPFeU), where freezes look better than they really are.

tlp-stress new functionality:
1. runners.parallelStream was replaced by Executors Fixed Thread Pool. parallelStream depends on system cores number which leads to unexpected results in case if client threads number is larger than number of cores.
2. Additional percentiles are reported in the output: min(p0), p50, p99.9, p99.99, max(p100) in addition to p99
3. Included HdrHistogram collector in addition to exiting collector. Use --hdr <hdr-output-file> to use HdrHistogram
4. Included the Response Time collection in addition to Service Time
5. Introduces a Response Time grace period - the time after which response time starts to include/accumulate waiting time. Use --response-time-warmup <number> to set the grace period. 

We hope to contribute this change back to the tlp-stress project and maintain it there. The remainder of this page is the same as https://github.com/thelastpickle/tlp-stress.

Full docs are here: http://thelastpickle.com/tlp-stress/

# Installation

The easiest way to get started on Linux is to use system packages.  Instructions for installation can be found here: http://thelastpickle.com/tlp-stress/#_installation


# Building

Clone this repo, then build with gradle:

    git clone https://github.com/thelastpickle/tlp-stress.git
    cd tlp-stress
    ./gradlew shadowJar

Use the shell script wrapper to start and get help:

    bin/tlp-stress -h

# Examples

Time series workload with a billion operations:

    bin/tlp-stress run BasicTimeSeries -i 1B

Key value workload with a million operations across 5k partitions, 50:50 read:write ratio:

    bin/tlp-stress run KeyValue -i 1M -p 5k -r .5


Time series workload, using TWCS:

    bin/tlp-stress run BasicTimeSeries -i 10M --compaction "{'class':'TimeWindowCompactionStrategy', 'compaction_window_size': 1, 'compaction_window_unit': 'DAYS'}"

Time series workload with a run lasting 1h and 30mins:

    bin/tlp-stress run BasicTimeSeries -d "1h30m"

Time series workload with Cassandra Authentication enabled:

    bin/tlp-stress run BasicTimeSeries -d '30m' -U '<username>' -P '<password>'
    **Note**: The quotes are mandatory around the username/password
    if they contain special chararacters, which is pretty common for password
