# Implementation and reproduction guide

## 1. Kernel geometry

In the original project tree, inspect `kernel/fs.h`:

```c
#define NDIRECT 11
#define NINDIRECT (BSIZE / sizeof(uint))
#define NDINDIRECT (NINDIRECT * NINDIRECT)
#define MAXFILE (NDIRECT + NINDIRECT + NDINDIRECT)
```

Both `struct dinode` and the in-memory inode in `kernel/file.h` use `uint addrs[NDIRECT + 2]`. This retains 13 slots. For 512-byte blocks, logical blocks 0–10 are direct, 11–138 single-indirect and 139–16522 double-indirect.

## 2. Mapping and reclamation

In `kernel/fs.c`, handle direct and single-indirect ranges first. For the double-indirect range, subtract 139 from the original logical block number before computing the root index (`bn / NINDIRECT`) and leaf index (`bn % NINDIRECT`). Allocate missing index/data blocks, log modified index buffers and release the root buffer before reading the leaf. In `itrunc()`, free data blocks first, then leaf index blocks, then the root; clear pointers and update the inode. These are implementation descriptions, not a complete patch.

## 3. Kernel test

Obtain the original tree and its matching compiler/QEMU configuration. Check BSIZE and disk capacity; rebuild a fresh disposable filesystem image because the pointer interpretation changed. Register `bigfile.c` in the build if needed. Boot using the project's Makefile target (often `make qemu`, but not confirmed here). Run `bigfile` in xv6: write 600 blocks, reopen, verify integer tags, then unlink. Capture console logs. Repeat allocation/deletion and check free-block counts to strengthen reclamation evidence.

## 4. RPC and deployment

Obtain `rpc_server.c`, `rpc_client.c`, Dockerfile and Kubernetes manifest. Inspect the actual client arguments and build instructions. The report describes native C request fields `int opcode`, `int arg1`, `char data[256]`; response fields `int status`, `char message[256]`. Test on a trusted local network only.

| Opcode | Operation | Expected behavior |
|---|---|---|
| 1 | GET_STORAGE_INFO | 16,523 blocks / approximately 8.46 MB |
| 2 | WRITE_DATA_BLOCK | Simulated write, e.g. block 500 |
| 3 | PING | PONG - Edge Node Healthy |

Illustrative deployment commands, once original files and image settings are available:

```sh
docker build -t xv6-edge-rpc .
docker run --rm --memory=128m --cpus=.25 -p 127.0.0.1:8080:8080 xv6-edge-rpc
# In another terminal:
docker stats --no-stream
# With a configured test cluster and accessible image:
kubectl apply -f <original-manifest.yaml>
kubectl get pods,svc
```

Use memory request 64Mi / limit 128Mi, CPU request 100m / limit 250m, container port 8080 and NodePort 30080 as documented. Do not expose the unauthenticated service publicly. The original 448 KiB measurement is not a peak guarantee.
