Do not use this, it's a very specific use case 

The included Jenkinsfile is a pipeline that will grab images from docker hub (or some other public registry), retag them, push them into a private registry/scratch repo and queue them for scanning in Anchore.  If the evaluation passes, they will retag again and push into a "production" repo for internal developers to use.

The hypothetical use case:
* internal developers are blocked from accessing Docker Hub (and other public registries) by firewall rules
* policy allows public images to be used if they pass a scan
* the internal anchore and jenkins installations are behind the firewall and cannot directly pull from Docker Hub (&c).
* there is a jump host in the DMZ that we can ssh to and that can access external public registries, internal registries, and the Anchore API endpoint.
