# Wireguard Helm Chart

[![CI Status](https://github.com/manics/wireguard-helm-chart/workflows/Test%20and%20Publish/badge.svg)](https://github.com/manics/wireguard-helm-chart/actions?query=branch%3Amain)

Deploys a Wireguard server on Kubernetes.
Wireguard listens on UDP port 51820 by default.

## Kubernetes node prerequisites

Wireguard requires that the host kernel includes the Wireguard module.

For example, on AWS EKS the default AMIs do not currently (February 2023) support Wireguard, but the [BottleRocket AMIs](https://docs.aws.amazon.com/eks/latest/userguide/launch-node-bottlerocket.html) do.

## Configuration

- `wireguard.accessibleIps`: Comma separate list of CIDRs that are accessible from the Wireguard network, e.g. `10.0.0.0/8, 172.16.0.0/20`, default `0.0.0.0/0`.
- `wireguard.clientPeers`: Either the number of client configurations to generate, or a comma separated list of client names that will be used to generate the client configuration files, default `example1, example2`.
- `wireguard.peerDns`: The DNS server to advertise to clients, default is the same as the Wireguard server (unlikely to work unless the DNS server is included in `accessibleIps`).
- `persistence.enabled`: The generated server and client configuration files are stored in a persistent volume, default `true`.

See [`values.yaml`](./values.yaml) for the full set of configuration parameters and defaults.

### Load-balancer configuration

Wireguard listens on UDP port 51820 by default.
If you are using a load-balancer be aware that some load-balancers will not forward traffic unless the Wireguard service provides a TCP or HTTP health check.
The Wireguard pod includes a simple HTTP server listening on port 58000 that returns status code `200` if Wireguard is running.

For example, to use an external AWS load-balancer add the following annotations to the service:

```yaml
service:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "58000"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: TCP
```

## Client configuration

When Wireguard starts it should generate configuration directories `/config/peer_*` for each client in `wireguard.clientPeers`.
These files can be copied from the Wireguard pod:

```
kubectl exec deploy/wireguard -- ls /config/
kubectl exec deploy/wireguard -c wireguard -- cat /config/peer_example1/peer_example1.conf > peer_example1.conf
```

If necessary change `Endpoint` in `peer_example1.conf` to the external IP address of the loadbalancer.

To connect to the Wireguard network on Linux:

```
wg-quick up peer_example1.conf
```

Or using NetworkManager:

```
nmcli con import type wireguard file peer_example1.conf
```

## References

- https://www.perdian.de/blog/2022/02/21/setting-up-a-wireguard-vpn-using-kubernetes/
- https://www.procustodibus.com/blog/2021/03/wireguard-health-check-for-python-3/
- https://docs.linuxserver.io/images/docker-wireguard
