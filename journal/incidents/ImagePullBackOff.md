## Sympton:
 
The pod is not ready

## Mitigation: 

roll back to a known image tag. The image reference is incorrect

## Root Cause:

The image tag does not exist or the image reference is wrong


## Cause category:

Configuration or process

## Prevention

Add image tag validation before merge
Use Renovate/image automation
Use Digest-pinned images for production
Run post-deploy image pull checks


## Final Analysis

The pod is not ready, because kubernetes cannaot pull the image, because the image does not exist. The bad tag reached git, becuase the deployment accepted the manual image edit. This can be prevented with Ci image validation or automated image updates

