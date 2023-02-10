
# HTTP2 HAPROXY+NGINX

This is an example to try reproduce an [issue](https://github.com/openshift/router/issues/448) we have encountered.

To start it up you need docker compose plugin.

```bash
$ docker compose up
```

Then haproxy is available at port 8443
```bash
curl -I --output /dev/null --limit-rate 10M -k https://localhost:8443/testimage
```

## The issue

### HTTP2 curl: (18) transfer closed with 29821952 bytes remaining to read 

We have encountered an issue when downloading large files with limited bandwidth with curl against OSD based api.openshift.com with HTTP2.

The problem manifests itself as follows:

```bash
podman run curlimages/curl:7.65.3 curl --limit-rate 10M -vvv 'https://api.openshift.com/api/assisted-images/boot-artifacts/rootfs?arch=x86_64&version=4.12' -o /tmp/bla
% Total % Received % Xferd Average Speed Time Time Time Current
Dload Upload Total Spent Left Speed
97 1052M 97 1024M 0 0 9.9M 0 0:01:45 0:01:42 0:00:03 10.1M
curl: (18) transfer closed with 29821952 bytes remaining to read 
```

The download always stops at 1GB (1024^3 bytes exactly)

This is true with all versions of curl up to 7.68 included, however we have noticed that from curl 7.69 this _seems_ to not happen.
With the help of RHEL curl maintainer we found out that in this version the initial window size would change from 1<<30 (which is the same as 1024^3) to 32MB (32 * 1024 * 1024)
https://bugzilla.redhat.com/show_bug.cgi?id=2166254

Surely enough, if we rate limit enough this behaviour occurs with curl 7.69 or greater

```bash
curl -A "banana" --limit-rate 538000 --output /dev/null 'https://api.openshift.com/api/as
sisted-images/boot-artifacts/rootfs?arch=x86_64&version=4.12'
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  3 1052M    3 32.0M    0     0   540k      0  0:33:13  0:01:00  0:32:13  784k
curl: (18) transfer closed with 1070009344 bytes remaining to read
```

And the difference it's again `total_image_size - 32MB`.
Interestigly, trying to find the maximum speed at which the transfer would fail, we noticed that if we're on the limit sometimes the download would fail at what it looks like to be the second window:

```bash
~ $ curl -A "banana" --limit-rate 538000 --output /dev/null 'https://api.openshift.com/api/as
sisted-images/boot-artifacts/rootfs?arch=x86_64&version=4.12'
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  4 1052M    4 48.0M    0     0   530k      0  0:33:50  0:01:32  0:32:18  501k
curl: (18) transfer closed with 1053230365 bytes remaining to read
```

In this case above it seems that they have negotiated a second window of half the size of the first one, but still errored at the end of the second window.

The above behaviour seems to happen only with the following factors:
* HTTP2 (if we force HTTP1.1, this behaviour won't happen)
* slow download of the window size (for 32MB above 1 minute, and for 1GB seemed to be triggered above 74s)
* files need to be bigger than window size
* only curl tested, but might happen on other client if they have similar window sizes

This does not happen on other HTTP2 servers, like cloudfront (mirror.openshift.com for example)

We have tested local plain haproxy with HTTP2, trying to reproduce the bug, but we were unable to do so
