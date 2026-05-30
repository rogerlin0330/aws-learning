# EBS Encryption

## What it is

EBS volumes are block storage devices presented over the network and stored resiliently within a single availability zone, but at the infrastructure level they live on one or more physical storage devices. **By default, no encryption is applied** — data is persisted to the raw disks exactly as the operating system writes it, in plain text. This creates a physical attack vector.

**EBS encryption provides at-rest encryption for both volumes and snapshots.** It's built on KMS, costs nothing to use, and has no performance penalty — so it's something to enable by default.

![[ebs-encryption.png]]

## Architecture / how it works

**Without encryption:** an instance generates data, and it's stored on the volume in plain text on the underlying raw storage.

**With encryption**, the flow is:

1. **Volume creation** — EBS uses a KMS key, which is either:
    - the **AWS managed default key** (named `aws/ebs`), or
    - a **customer managed KMS key** you create and control.
2. That KMS key generates a per-volume **data encryption key (DEK)** via the `GenerateDataKeyWithoutPlaintext` API call. EBS only ever receives the _encrypted_ form of the DEK, which is stored alongside the volume on the raw storage. It can only be decrypted by KMS, and only by an entity with permission to decrypt using the corresponding KMS key. _(In the diagram, this is the **red arrow** running from the `GenerateDataKeyWithoutPlaintext` label to the DEK — the encrypted DEK being produced and stored at rest.)_
3. A new volume starts empty — it's just an allocation of space, so there's nothing to encrypt yet.
4. **First use** (mounting the volume, or launching the instance) — EBS asks KMS to decrypt the DEK, which is then loaded into the **memory of the EC2 host** using the volume. The decrypted DEK exists _only_ in host memory, never on disk. _(In the diagram, this is the round-trip formed by the **blue arrow** pointing up into KMS — the request to decrypt the DEK — and the **yellow arrow** running from KMS down to the green key on the EC2 Host bar — the decrypted DEK being delivered into host memory.)_
5. The host uses the decrypted DEK to **encrypt/decrypt data as it moves between the instance and the raw storage**. What's written to the raw storage is therefore **ciphertext** — encrypted at rest. Data exists in unencrypted form only inside the EC2 host's memory.
6. **When the instance moves to another host**, the decrypted DEK is discarded, leaving only the encrypted DEK with the disk. To use the volume again, that encrypted DEK must be decrypted and loaded into the new host.

**Snapshots and derived volumes:** a snapshot of an encrypted volume reuses the _same_ DEK, so the snapshot is also encrypted. Any volume created from that snapshot is likewise encrypted with that same DEK.

## Exam summary points

### Where the KMS key fits

- An AWS account can be configured to **encrypt EBS volumes by default**, with either a set default KMS key or a key chosen manually each time.
- The KMS key **does not directly encrypt or decrypt volume data**. It only generates the per-volume DEK. (This indirection is a common point of confusion.)

### DEK reuse rules (frequent exam trap)

- Every volume created **from scratch** gets a **unique DEK** — create two from-scratch volumes, you get two different DEKs.
- A **snapshot** of an encrypted volume uses the **same DEK** as the source volume.
- A volume **created from that snapshot** also uses that **same DEK**.

So a single DEK chains through: original volume → its snapshots → volumes restored from those snapshots.

### Encryption is irreversible

- There is **no way to remove encryption** from a volume or snapshot through the console, CLI, or APIs. Once encrypted, always encrypted.
- The only workaround is to **clone the data from inside the operating system** onto a separate unencrypted volume — possible only because the OS sees plain text.

### The OS is unaware of encryption

- To the operating system, the volume is **plain text** — encryption happens transparently _between the EC2 host and the volume_, using **AES-256**.
- If you specifically need the **OS to hold the keys and perform encryption** itself, that's **software disk encryption**, configured within the OS. It can be layered on top of EBS encryption at the same time (rarely necessary).

### Why use it

EBS encryption is efficient: no keys to manage, no cost, and no performance loss.