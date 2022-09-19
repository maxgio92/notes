---
Title:  Kubectl resources
---

Assume unit CPU: m, Memory: Ki

```
allocatable_cpu=$(kubectl describe node |grep Allocatable -A 5|grep cpu | awk '{sum+=$NF;} END{print sum;}')
allocatable_mem=$(kubectl describe node |grep Allocatable -A 5|grep memory| awk '{sum+=$NF;} END{print sum;}')
the allocated resource in request field

allocated_req_cpu=$(kubectl describe node |grep Allocated -A 5|grep cpu | awk '{sum+=$2; } END{print sum;}')
allocated_req_mem=$(kubectl describe node |grep Allocated -A 5|grep memory| awk '{sum+=$2; } END{print sum;}')
finally, we got the resource spaces left

usable_req_cpu=$(( allocatable_cpu - allocated_req_cpu ))
usable_req_mem=$(( allocatable_mem - allocated_req_mem ))

echo Usable CPU request $usable_req_cpu M
echo Usable Memory request $usable_req_mem Ki
```
