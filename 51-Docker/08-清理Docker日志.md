```shell
#!/bin/bash

echo "clean docker containers logs start"

logs=$(find /var/lib/docker/containers/ -name *-json.log)
 
for log in $logs
	do
		echo "clean logs : $log"
		cat /dev/null > $log
	done
 
echo "clean docker containers logs end"
```