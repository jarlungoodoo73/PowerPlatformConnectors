# Analysis: xpipe-webtop Docker Build Failure

## Issue Reference
- **Repository**: xpipe-io/xpipe-webtop
- **Workflow Run**: https://github.com/xpipe-io/xpipe-webtop/actions/runs/20578223451/job/59099982309
- **Date**: December 29, 2025
- **Status**: Failed

## Problem Description

The Docker build workflow is failing during the AWS SSM Session Manager plugin installation step with exit code 1.

### Failed Command
```bash
RUN echo "**** aws ssm ****" && \
    curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o "/tmp/session-manager-plugin.deb" && \
    sudo dpkg -i "/tmp/session-manager-plugin.deb" && \
    rm -rf "/tmp/aws" "/tmp/session-manager-plugin.deb"
```

### Error Message
```
ERROR: failed to build: failed to solve: process "/bin/sh -c echo \"**** aws ssm ****\" && 
curl \"https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb\" 
-o \"/tmp/session-manager-plugin.deb\" &&   sudo dpkg -i \"/tmp/session-manager-plugin.deb\" &&   
rm -rf \"/tmp/aws\" \"/tmp/session-manager-plugin.deb\"" did not complete successfully: exit code: 1
```

## Root Cause Analysis

The failure occurs at Dockerfile lines 131-133. Possible causes:

1. **Download Failure**: The curl command may be failing to download the .deb package
   - Network connectivity issues
   - URL changed or deprecated
   - AWS S3 bucket access restrictions

2. **Installation Failure**: The dpkg installation may be failing
   - Missing dependencies
   - Architecture mismatch
   - Corrupted package download

3. **Permission Issues**: Even with `sudo`, there may be permission problems in the Docker build context

## Recommended Fixes

### Fix 1: Add Error Handling and Logging
```dockerfile
RUN echo "**** aws ssm ****" && \
    curl -v -f "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" \
         -o "/tmp/session-manager-plugin.deb" && \
    ls -lh "/tmp/session-manager-plugin.deb" && \
    (sudo dpkg -i "/tmp/session-manager-plugin.deb" || \
     (echo "dpkg failed, checking logs:" && cat /var/log/dpkg.log && false)) && \
    rm -rf "/tmp/aws" "/tmp/session-manager-plugin.deb"
```
Note: This approach preserves temporary files on failure for debugging. For production, ensure cleanup always runs using a trap or separate cleanup step.

### Fix 2: Install Dependencies First
```dockerfile
# Install dependencies before session manager
RUN apt-get update && \
    apt-get install -y curl sudo && \
    apt-get clean

RUN echo "**** aws ssm ****" && \
    curl -f "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" \
         -o "/tmp/session-manager-plugin.deb" && \
    (sudo dpkg -i "/tmp/session-manager-plugin.deb" || \
     (sudo apt-get install -f -y && sudo dpkg -i "/tmp/session-manager-plugin.deb")) && \
    rm -rf "/tmp/aws" "/tmp/session-manager-plugin.deb"
```
Note: This fix attempts to install dependencies if dpkg fails, then retries the installation.

### Fix 3: Add Retry Logic and Better Error Handling
```dockerfile
RUN echo "**** aws ssm ****" && \
    for i in 1 2 3; do \
        curl -f "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" \
             -o "/tmp/session-manager-plugin.deb" && break || sleep 5; \
    done && \
    test -f "/tmp/session-manager-plugin.deb" && \
    sudo dpkg -i "/tmp/session-manager-plugin.deb" && \
    rm -rf "/tmp/aws" "/tmp/session-manager-plugin.deb"
```
Note: This adds retry logic for the download which can help with transient network issues.

### Fix 4: Pin to Specific Version
Instead of using `/latest/`, pin to a specific known-working version:
```dockerfile
RUN echo "**** aws ssm ****" && \
    curl -f "https://s3.amazonaws.com/session-manager-downloads/plugin/1.2.463.0/ubuntu_64bit/session-manager-plugin.deb" \
         -o "/tmp/session-manager-plugin.deb" && \
    sudo dpkg -i "/tmp/session-manager-plugin.deb" && \
    rm -rf "/tmp/aws" "/tmp/session-manager-plugin.deb"
```

## Investigation Steps

To diagnose the exact cause, the xpipe-webtop maintainers should:

1. **Test the download URL directly**:
   ```bash
   curl -I "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb"
   ```

2. **Check if the package still exists**:
   - Visit AWS Session Manager plugin documentation
   - Verify the correct download URL for ubuntu_64bit

3. **Review Dockerfile for required dependencies**:
   - Ensure `curl` and `sudo` are installed before this step
   - Check if any system libraries are needed

4. **Test locally**:
   ```bash
   docker build --target <build-stage> .
   ```

## Additional Resources

- [AWS Systems Manager Session Manager Plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)
- [Docker Build Troubleshooting](https://docs.docker.com/build/troubleshooting/)

## Note

This analysis is created in the `jarlungoodoo73/PowerPlatformConnectors` repository for reference purposes. 
The actual fix must be applied in the `xpipe-io/xpipe-webtop` repository by opening a PR there or 
creating an issue for the maintainers to address.
