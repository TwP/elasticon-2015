July 2014 - Heap exhaustion on a code search storage node

[support 3813](https://support.elasticsearch.com/requests/3813)

## background

At this point in time our `codesearch` cluster is running ES 0.90.10. There are
10 Dell R720 servers with ~7TB of SSD storage and 10GB network connectivity. We
also have 3 smaller machines that serve only as masters. This cluster has been
running happily over the past eight months, and performance has been great.

This cluster has gone through several upgrades of the installed ES version.
There have been some hiccups, but overall upgrades have gone smoothly.

## what went wrong

## how we fixed it

## lessons learned

```
codesearch-storage1       6.9T  5.9T 1022G  86%
codesearch-storage2       6.9T  6.2T  699G  91%
codesearch-storage3       6.9T  6.1T  841G  89%
codesearch-storage4       6.9T  6.0T  935G  87%
codesearch-storage5       6.9T  6.3T  630G  92%
codesearch-storage6       6.9T  6.2T  672G  91%
codesearch-storage7       6.9T  6.1T  859G  88%
codesearch-storage8       6.9T  6.1T  843G  88%
codesearch-storage9       6.9T  6.1T  870G  88%
codesearch-storage10      6.9T  6.0T  921G  87%
```
