# k8s-claimer

[![Build Status](https://travis-ci.org/deis/k8s-claimer.svg?branch=master)](https://travis-ci.org/deis/k8s-claimer)
[![Go Report Card](http://goreportcard.com/badge/deis/k8s-claimer)](http://goreportcard.com/report/deis/k8s-claimer)
[![Docker Repository on Quay](https://quay.io/repository/deisci/k8s-claimer/status "Docker Repository on Quay")](https://quay.io/repository/deisci/k8s-claimer)

`k8s-claimer` is a leasing server for a pool of Kubernetes clusters. It will be used as part of our
[deis-workflow end-to-end test](https://github.com/deis/workflow-e2e) infrastructure.

Note that this repository is a work in progress. The code herein is under heavy development,
provides no guarantees and should not be expected to work in any capacity.

As such, it currently does not follow the
[Deis contributing standards](http://docs.deis.io/en/latest/contributing/standards/).

# Design

This server is responsible for holding and managing a set of [Google Container Engine](https://cloud.google.com/container-engine/)
(GKE) clusters. Each cluster can be in the `leased` or `free` state, and this server is responsible for
responding to requests to change a cluster's state, and then safely making the change.

A client who holds the lease for a cluster has a [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier)
indicating their ownership as well as the guarantee that nobody else will get the lease before
either their lease duration expires or someone releases the lease with their UUID. The client
specifies the lease duration when they acquire it.

For implementation details, see [the architecture document](doc/architecture.md)

# API

The server exposes a REST API to acquire and release leases for clusters. The subsections
herein list each endpoint.

## `POST /lease`

Acquire a new lease.

### Request Body

```json
{"max_time": 30}
```

Note that the value of `max_time` is the maximum lease duration in seconds. It must be a number.
After this duration expires, the lease will be automatically released.

### Responses

Unless otherwise noted, all responses except for `200 OK` indicate that the lease was not acquired.

In non-200 response code scenarios, a body may be returned with an explanation of the error,
but the existence or contents of that body are not guaranteed.

#### `401 Bad Request`

This response code is returned with no specific body if the request body was malformed.

#### `500 Internal Server Error`

This response code is returned if any of the following occur:

- The server couldn't communicate with the Kubernetes Master to get the service object
- The server couldn't communicate with the GKE API
- A cluster was available, but the new lease information couldn't be saved
- An expired lease exists but it points to a non-existent cluster
- The lease was succesful but the response body couldn't be rendered

#### `409 Conflict`

This response code is returned if there are no clusters available for lease.

#### `200 OK`

This response code is returned along with the below response body if a lease was successfully
acquired.

```json
{
  "kubeconfig": "RFC 4648 base64 encoded Kubernetes config file. After decoding, this value can be written to ~/.kube/config for use with kubectl",
  "ip": "The IP address of the Kubernetes master server in GKE",
  "token": "The token of the lease. This is your proof of ownership of the cluster, until the lease expires or you release it",
  "cluster_name": "The name of the cluster. This value is purely informational, and fetched from GKE"
}
```

## `Delete /lease/{token}`

Release an existing lease, identified by `{token}`.

### Responses

All responses except for `200 OK` indicate that no leases were changed. Since the state of the
lease identified by `{token}` (if there was one) can change over time, there are no guarantees
on the state of the lease after this API call returns in these cases.

In all cases, a body may be returned with an explanation of the response, but the existece or
contents of that body are not guaranteed.

#### `401 Bad Request`

This response code is returned in the following cases:

- The URL path did not include a lease token
- The lease token was malformed

#### `500 Internal Server Error`

This response code is returned in the following cases:

- The server couldn't communicate with the Kubernetes Master to get the service object
- The lease was found and deleted, but the updated lease statuses couldn't be saved

#### `409 Conflict`

This response code is returned when no lease exists with the given token.

#### `200 OK`

The lease was successfully deleted. The given token is no longer valid and should not be reused.
