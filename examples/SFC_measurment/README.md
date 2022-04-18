SFC measurenment
==
This NF marked imcoming packet with timestamps, and it measures latency and throughput when packets it marked are returned to itself.
If you want to get lantency infomation, send packets back to itself.

Compilation and Execution
--
```
cd examples
make clean
make
```
Usage
--
./go.sh my_service_number -d dst_service_number
