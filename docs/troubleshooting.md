# Troubleshooting Notes

This file records real problems encountered during the project and the lesson
from each one. Do not paste secrets, tokens, kubeconfig contents or private keys
here.

## Phase 1 - Multipass VM Had No External Internet

Symptom:

- the Multipass VM could reach its gateway;
- the VM could not reach the external internet;
- package and image download steps were blocked.

Resolution:

- the Mac was restarted;
- after restart, the Multipass VM network path recovered.

Lesson:

- if the VM reaches the gateway but not the internet, the issue may be on the
  macOS virtualization or network side rather than inside RKE2;
- before changing RKE2 configuration, first verify basic VM connectivity.

Useful checks:

```bash
multipass exec rke2-server -- ip route
multipass exec rke2-server -- ping -c 3 8.8.8.8
multipass exec rke2-server -- curl -I https://get.rke2.io
```

## Phase 1 - RKE2 Stayed In `activating` During First Start

Symptom:

- `rke2-server.service` stayed in `activating` for a long time on first start;
- the cluster was still pulling required images and preparing etcd/Kubernetes
  components.

Resolution:

- waited while images and control plane components initialized;
- once etcd became ready, the service moved to a normal running state.

Lesson:

- first RKE2 startup can take time, especially when images are pulled for the
  first time;
- `activating` is not automatically a failure;
- repeated `failed`, `fatal`, `x509`, `ImagePullBackOff` or disk-space errors
  are stronger signs that troubleshooting is needed.

Useful checks:

```bash
multipass exec rke2-server -- sudo systemctl status rke2-server.service --no-pager
multipass exec rke2-server -- sudo journalctl -u rke2-server -f
```

Stop live log watching with `Ctrl+C`. This stops only the terminal log stream;
it does not stop RKE2.

## Phase 1 - Snapshot Showed A `tls-san` Warning

Symptom:

- an etcd snapshot command showed a `tls-san` warning;
- the snapshot was still saved successfully.

Resolution:

- treated the command as successful because the snapshot was written.

Lesson:

- warnings should be read, but not every warning means the operation failed;
- for snapshot operations, verify the actual snapshot list or saved filename
  before deciding whether recovery action is needed.

Useful checks:

```bash
multipass exec rke2-server -- sudo rke2 etcd-snapshot ls
```
