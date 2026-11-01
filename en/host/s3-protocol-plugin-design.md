# S3 Protocol Support: Plugin Design and Implementation

> Written: 2026-08-13 (rewritten on 2026-08-14 for "the full protocol + operable from the UI"; rewritten again the
> same day as a **plugin**, with sections 5 and 10 rewritten and section 10 added)　　Baseline code: `main` @ `ab8a9fa`
>
> Prerequisite reading: [`ftp-client-feasibility-research.md`](ftp-client-feasibility-research.md) — the seam established
> in its first section (a remote file contract keyed by session id that returns protocol-neutral entries) is reused here
> directly and not re-explained; and [`../templates/dev-guide.md`](../templates/dev-guide.md) §5.13 (protocol capability).

---

## 0. What this version changed

S3 support **moved from being built into the host to a plugin** (`plugins/VelaShell.Plugin.S3`). The protocol
implementation, the AWSSDK dependency, the bucket manager and object inspector UIs, and 149 S3 strings all left the
host; on the host side a general-purpose **protocol capability domain** was added (the plugin SDK's
`IProtocolFileSystem` + the host's registry and adapter), and S3 is merely its first consumer.

What that buys:

1. **Users who do not use S3 no longer pay for it**: AWSSDK.S3 + AWSSDK.Core (about 1 MB each) stay out of the host
   process, and the plugin directory is distributed on demand; unless the S3 tab is opened, not a single line of it is
   loaded.
2. **The host no longer knows any specific protocol**: `ConnectionType` no longer has `S3`, only a single `Plugin`;
   bringing in WebDAV / GCS / the native Alibaba Cloud OSS SDK would not change a single line of the host.
3. **Third parties can bring their own protocols**: the capability domain is part of the public SDK, not a private
   back door for S3.

The costs and boundaries are in §10.

---

## 1. Conclusion first

**S3 is not a protocol as small as "upload and download".** Counting `IAmazonS3` by reflection, it has
**116 operations**:

| Category | Count | Examples |
|---|---|---|
| Object reads, writes and transfer | about 20 | The whole Get/Put/Copy/Delete/Multipart family |
| Bucket-level configuration | about 55 | Versioning, lifecycle, policy, ACL, CORS, encryption, object lock, replication, notifications, inventory… |
| Object-level properties | about 20 | Tags, ACL, retention, legal hold, archive restore, attributes |
| Other | about 20 | Presigning, S3 Select, directory buckets, metadata tables |

**So the feature runs on two tracks, and both share the same session and the same client:**

| Component | Contract | Responsibility |
|---|---|---|
| `S3ProtocolFileSystem` | The SDK's `IProtocolFileSystem` | File-manager semantics: translating the flat key space into a directory tree |
| `S3ManagementService` | The plugin's own `IS3ManagementService` | Everything else: bucket configuration, versions, ACLs, object lock, tags, restore, multipart-upload management, S3 Select |

The UI has the same two tracks: the host's existing dual-pane file browser (reused with zero changes), plus two
plugin panels, the **S3 bucket manager** and the **S3 object inspector**.

---

## 2. Why AWSSDK.S3 rather than a hand-written implementation

**This decision was overturned and redone on 2026-08-14, and the story is worth recording.**

The first version chose to implement SigV4 + REST by hand. That judgement rested on a **wrong assumption about
scope** — that "S3 support" roughly meant the dozen or so operations of listing, uploading, downloading, deleting and
copying. Under that assumption, signing requests by hand in 200 lines really was cheaper than dragging in a whole SDK.

Once the scope was redrawn to the protocol's actual size (116 operations), the sums immediately flipped:

| Dimension | Hand-written | AWSSDK.S3 |
|---|---|---|
| Code to cover 116 operations | XML serialisation + error-code mapping written once per operation, thousands of lines | Already there |
| New or changed AWS capabilities | Redone every time | Upgrade the package |
| Concurrency / retries / aborts for multipart upload | Written by hand | `TransferUtility` |
| Licence | — | Apache-2.0, compatible with this repository's AGPL-3.0 + commercial dual licence |
| Cost | — | Two more assemblies (AWSSDK.S3 + AWSSDK.Core, about 1 MB each) |

**Lesson**: a dependency trade-off must start by measuring the protocol surface; only then talk about "two hundred
lines vs. one package". The earlier comparison had the wrong denominator.

> The same yardstick was used for picking FluentFTP on the FTP side: when the protocol surface is large, the state
> machine complicated and writing it by hand risky, choose a mature library. Both conclusions agree; the first version
> simply misjudged the size of S3's protocol surface.

**The layering constraint is unchanged; only the boundary moved**: AWSSDK types (`AmazonS3Exception`,
`S3StorageClass` and so on) **must not cross the plugin assembly boundary** — the same hard rule as for FluentFTP /
Tmds.Ssh in the host. Inside the plugin the `VelaS3*` exception family is used (translated in one place, `S3Interop`);
at the exit it is translated into the SDK's four `Protocol*` kinds (authentication / certificate / connection /
unsupported), and the host side translates those into the neutral exception family of `VelaShell.Core.Protocols`.

The two translation steps look redundant but each has its own job: `VelaS3*` has to distinguish "never configured /
unsupported / insufficient permission / does not exist" — the bucket manager's empty states depend entirely on that —
whereas the host only cares about those four kinds, and the extra distinctions mean nothing to it.

---

## 3. Core design 1: flat key space → directory tree

S3's key space is flat; `/` is just an ordinary character in a key name. The agreed mapping
(`Core/S3/S3ObjectPath.cs`):

| Path | Meaning |
|---|---|
| `/` | Root = the list of buckets (each bucket shown as a directory) |
| `/bucket` | Bucket root = list with prefix `""` and delimiter `/` |
| `/bucket/a/b` | The key `a/b`, or the prefix `a/b/` (depending on whether it is an object or a "directory") |

Four rules:

1. **"Subdirectories" come from `CommonPrefixes`** (folded by the server when listing with `delimiter=/`).
2. **An empty directory is a zero-byte object whose key ends in `/`.** This is an established convention of the S3
   ecosystem; creating a directory creates one, and listing hides it.
3. **Rename/move has no primitive**; it is always "server-side copy + delete". The order cannot be reversed: the
   source is deleted only after the copy succeeds.
4. **chmod has no corresponding semantics**, and explicitly throws `VelaS3UnsupportedOperationException`; access
   control goes through ACLs and bucket policies.

### 3.1 Where the semantics differ from SFTP/FTP

| Capability | SFTP / FTP | S3 | Handling |
|---|---|---|---|
| Resuming uploads | The server holds half a file to resume from | **Writes are atomic**; a failure leaves nothing behind | `resumeOffset` is ignored and the whole file is re-sent (stated plainly in the method documentation) |
| Resuming downloads | Supported | Supported (`Range`) | Implemented normally; when the server ignores `Range`, it falls back to writing from the start |
| Same-site copy | Not in FTP | **Server-side copy** | S3 beats FTP on this one |
| Directory modification time | Present | None (directories are fictional) | Returns `DateTime.MinValue` so the UI leaves it blank |
| Owner / group | Present | Owner only | Group is left blank, **not filled with the storage class** — filling it in would only make the column header lie |
| Setting the modification time (SDK 2.1.3 `IProtocolLastWriteTime`) | SFTP has it, FTP depends on MFMT | **None**: `LastModified` is the moment the server wrote the object; "copy the object onto itself + replace metadata" rewrites the object, changes its version id and ETag, and still records the time of the copy | The optional interface is **not implemented**; the host honestly reports "cannot set modification times" and directory sync compares by size / digest instead |
| Server-side digests (SDK 2.1.3 `IProtocolChecksum`) | SFTP via remote commands, FTP depends on HASH | Stored only when the object was uploaded **with** an `x-amz-checksum-*`; a multipart upload stores a composite digest (a digest of the part digests) | **Implemented**: HEAD with `x-amz-checksum-mode: ENABLED` returns the stored full-object digest (SHA-256 / SHA-512 / SHA-1 / MD5); composite digests, nothing stored, directories / buckets and no permission all give null; the object is never read back and **the ETag is not passed off as an MD5**. The plugin's own uploads send no checksum (`WHEN_REQUIRED`, see §6), so objects it uploaded always give null |

### 3.2 Only empty buckets are deleted; they are never emptied on the user's behalf

`DeleteBucket` requires the bucket to be empty, and that is a **safety property**: a bucket is a top-level resource,
often shared by several parties, and once its name is released someone else may grab it. In the file browser, buckets
under the root view look exactly like ordinary directories, and the delete confirmation says only "Delete folder
xxx?". If the client emptied the bucket on the user's behalf and then deleted it, one misclick would take the whole
bucket's data with it. So `BucketNotEmpty` is passed through as is.

> Recursively deleting a **prefix** is supported as usual — that matches the "delete folder" semantics of SFTP/FTP,
> and users expect the same thing.

---

## 4. Core design 2: bucket configuration collapsed into one set of triple methods

S3's twenty-odd kinds of bucket configuration all have the same shape in the protocol: `GetBucketXxx` /
`PutBucketXxx` / (for some) `DeleteBucketXxx`, differing only in the payload. So `IS3ManagementService` does not open
three methods per kind of configuration (that would bloat the interface to more than seventy members), but collapses
them into:

```csharp
Task<S3ConfigResult> GetBucketConfigAsync(session, bucket, S3ConfigKind kind, id?);
Task PutBucketConfigAsync(session, bucket, S3ConfigKind kind, string json, id?);
Task DeleteBucketConfigAsync(session, bucket, S3ConfigKind kind, id?);
Task<IReadOnlyList<string>> ListBucketConfigIdsAsync(session, bucket, S3ConfigKind kind);
```

**Adding a kind of configuration = one enum member + one switch branch + one navigation entry**, each a few lines.
That is exactly why this feature set can offer "full coverage" rather than "half coverage".

### 4.1 Why the payload is JSON

Building a DTO in Core for every kind of configuration and hand-writing the mapping in both directions would cost
over a thousand lines of plumbing, and — **worse** — any field left out would be silently cleared on write-back (a
field read in but not rendered is gone once serialised back). That would genuinely break the user's production
configuration.

Serialising the SDK's model objects directly is complete by nature: what is read is what you get. The structured form
is merely a **view** of that JSON; writing back goes through incremental edits on the JSON DOM
(`S3ConfigForm.Apply`), overwriting only the paths the form actually edited and keeping everything else as it was.
`S3ConfigFormTests.Apply_PreservesFieldsTheFormDoesNotKnowAbout` guards this property.

### 4.2 Form vs. JSON editor

| Editor | Suited to | Configuration items |
|---|---|---|
| Structured form | Few fields, stable shape | Versioning, public access block, object ownership, encryption, object lock, tags, static website, access logging, transfer acceleration, requester pays |
| JSON document editor | **Things that are documents to begin with** | Lifecycle, bucket policy, ACL, CORS, replication, event notifications, inventory, analytics, metrics, intelligent tiering, metadata tables, ABAC |

Forcing the second group into forms could never be complete and would break the configuration on meeting an unknown
field — the AWS console also gives JSON for these items. **Offering a document editor is not laziness; it is the more
complete and safer choice.**

---

## 5. UI

### 5.1 Connection settings page (rendered by the host, only declared by the plugin)

The S3 tab is **not** written in `ConnectionProfileView.axaml`; it is a piece of data the plugin declares in
`contributes.protocols` in `plugin.json`. The connection form (region / addressing style / HTTPS / default bucket /
session token / part size / concurrency / storage class / encryption / directory placeholder) is data too — the plugin
declares each field's key, label, shape and default in `ProtocolDescriptor.Fields`, and the host renders a control
according to the shape.

Ten fields laid out in one column pushed the connection dialog off the screen (user feedback), so the last six
(session token / part size / concurrency / storage class / encryption / directory placeholder) are marked
`IsAdvanced`: the host puts them on the connection dialog's "Advanced" page and the tab shows the field count `6`;
when editing an existing profile in which any of them differs from its default, a dot appears on the tab. The first
four decide "whether it can connect at all" and stay on the "General" page.

**So the plugin has not a single line of connection-dialog UI code.** This is a different thing from the general
declarative UI that blueprint 08 explicitly rules out: that is arbitrary UI, whereas this is only a parameter table of
closed shape (text / password / boolean / integer / dropdown — five kinds), with fixed semantics, validatable and
themeable. The connection dialog is a core part of the host's UI, and its layout and interaction should be owned by
the host in one place.

### 5.2 File browser (existing, reused with zero changes + three declarative menu items)

The host's remote file contract is protocol-neutral, so dual-pane browsing, transfers, rate limiting, drag and drop
and conflict policies all work with zero changes. The three extra items in the context menu come from
`ProtocolDescriptor.Actions`:

- **Copy share link** — a presigned URL (7 days)
- **S3 object management…** — opens the object inspector
- **Bucket management…** — opens the bucket manager

The actions are **declarative** (supplied together with the protocol descriptor), so the menu can be drawn in the very
frame the right button goes down; switching to "call back into the plugin on every right-click" would add an
asynchronous round trip, and the menu would visibly lag. On click, the host only hands "which action + which path" to
the plugin, and what happens is entirely up to the plugin.

### 5.3 S3 bucket manager / object inspector (plugin panels)

Both windows are now panels the plugin draws itself in full Avalonia, opened through `context.Ui.ShowPanelAsync` with
`PanelDisplayMode.Window` — the window shell (title bar, rounded corners, light/dark theme) is provided by the host,
the plugin draws only the content and always uses `{DynamicResource VelaXxx}` theme tokens, so it still looks like
part of the host.

Navigation is driven by `S3ConfigDescriptor.All`: **whatever the protocol has, this has.**

| Page | Operations covered |
|---|---|
| Overview | `GetBucketLocation`, `GetBucketPolicyStatus`, `GetBucketVersioning`, `GetObjectLockConfiguration` |
| Object versions | `ListVersions` + deleting a given version + restoring an earlier version (implemented as a copy) |
| Incomplete multipart uploads | `ListMultipartUploads` + `AbortMultipartUpload` |
| 22 configuration items | See `S3ConfigKind` in §4 |

The object inspector has tabs: Details / Tags / Permissions / Retention and legal hold / Storage and encryption /
S3 Select, covering `GetObjectAttributes`, `GetObjectMetadata`, `Get/Put/DeleteObjectTagging`, `Get/PutObjectAcl`,
`Get/PutObjectRetention`, `Get/PutObjectLegalHold`, `RestoreObject`, `CopyObject` (changing storage class /
encryption / metadata), `SelectObjectContent` and `GetPreSignedURL`.

> The panel view models **do not use ReactiveUI**: the `RxApp` in the plugin's ALC and the host's are two independent
> instances, its `MainThreadScheduler` is not hooked up to the Avalonia dispatcher automatically, and commands'
> `CanExecuteChanged` would trigger binding updates on a background thread. The plugin's own eighty-line
> `ObservableObject` + `AsyncCommand` is more robust, and also saves three assemblies that would otherwise ship with
> the plugin.

## 6. Compatibility notes (easy pitfalls)

1. **`RequestChecksumCalculation` must be set to `WHEN_REQUIRED`.** AWSSDK v4 computes a CRC32 for every request by
   default and expects responses to carry a checksum back; a good share of S3-compatible implementations (older MinIO /
   Ceph RGW / gateways of all kinds) do not understand these headers, which shows up as rejected uploads or checksum
   failures on download.
2. **`AllowAutoRedirect` must be false.** Region redirects have to be re-signed and re-sent by the SDK according to
   `x-amz-bucket-region`; following them automatically takes the old region's signature to the new address, which is
   bound to produce `SignatureDoesNotMatch`.
3. **The addressing-style default must be conservative**: `Auto` uses virtual-hosted style only for AWS's own endpoints
   (`*.amazonaws.com`) and path style for everything else. Self-deployed MinIO / Ceph does not resolve `bucket.host`
   by default (it needs `MINIO_DOMAIN`), whereas path style works on both.
4. **"Never configured" and "unsupported" are not errors**; they are empty states. AWS has a dedicated error code for
   every capability that has not been configured (`NoSuchLifecycleConfiguration`, `NoSuchCORSConfiguration`…), while
   compatible implementations often just return 501. Treating them as errors would turn the bucket manager red the
   moment it opens — `S3Interop.IsMissingOrUnsupported` exists specifically to recognise this class.
5. **Self-signed certificates** go through exactly the same "reject first → prompt → record the fingerprint and
   reconnect" flow as FTPS; both protocols share the same dialog and the same set of `Cert_*` strings. It deliberately
   does not wait synchronously for the user to click a button inside the TLS callback (that would mean blocking an
   asynchronous dialog into a synchronous one, which deadlocks very easily).
6. **The part count is capped at 10000**: the part size is raised to `max(configured value, file size / 10000)`;
   otherwise a 100 GB file would be rejected at part 10001.
7. **The content type can only be set at the step that initiates the multipart upload** (neither
   `UploadPart` nor `Complete` gets another chance to supply it).
8. **Presigned URLs must be given `Protocol` explicitly.** The SDK always produces HTTPS URLs and **ignores**
   `config.UseHttp` — a self-deployed endpoint on plain HTTP gets an `https://` link that cannot connect when pasted
   into a browser, and the symptom looks exactly like a broken feature.

### 6.1 Reading an object means trying three paths, not one

On real endpoints, "download" is far from a single `GetObject`. The layers met in the field (2026-08-17, an
S3-compatible endpoint behind a CDN):

| Symptom | Cause | Response |
|---|---|---|
| Every download fails, and the error is just `Error Code Forbidden … No further error information` | What fails is **the HEAD before the download**, and the HEAD's 403 **has an empty response body**, so the SDK can only make up that sentence | HEAD becomes best effort: it only provides the total length and mtime (to report progress, compute the resume offset and align timestamps), which the GET response carries as well |
| Public-read objects fail to download when signed, while private objects work | Public readability comes from anonymous grants, whereas a request carrying an `Authorization` header is judged by that key's policy — no allow in the policy means deny | Not something the client can fix; but the error must say clearly which object and which error code |
| Direct download is refused, presigned download is allowed | Presigning puts the credentials in the **query string** rather than in headers, and the two often take different paths on the server: a bucket policy can allow only the presigned form, and CDNs / gateways in front commonly strip or rewrite the `Authorization` header | On a 401/403 from the direct fetch, automatically fetch once more through a presigned URL |

So the rule for `OpenObjectAsync` is: **use HEAD if it works, do not let it block if it does not; when the direct
fetch is refused, switch to presigned.** It falls back only on 401/403 — a 404 is still missing on a second try, 5xx
has already been retried by the SDK itself, and a network error does not get through on a different URL either. When
both paths fail, the error from the **direct** fetch is reported: the presigned attempt is an extra try the client
made on its own initiative, and using its failure to explain "why the download failed" would only mislead.

The presigned path does not go through the SDK, so it needs its own `HttpClient` — which must share the same
certificate probe as the SDK's; otherwise a self-signed endpoint the user clicked "Trust this certificate" for while
connecting would be judged untrusted again on the presigned path.

---

## 7. Credentials and security

- **Access Key ID → `SessionProfile.Username`, Secret Access Key → `SessionProfile.Password`.**
  The general-purpose fields are reused instead of opening two new ones: the AES-encrypted at-rest storage of
  "Remember password", the login prompt, and the importers' credential recovery all work with zero changes.
- **The STS session token** is `S3Settings.SessionToken`; it is a credential too, and is encrypted separately before
  being written to disk.
- **Anonymous access is a legitimate path** (public read-only buckets): when both are empty, no login prompt is shown
  and the connection is made anonymously; only "an Access Key filled in but no Secret" counts as genuinely missing
  credentials.
- **Plain HTTP** gets a warning in the connection dialog: the keys are still protected by the signature, but the object
  content travels in the clear.

---

## 8. Tests

| Test | Count | What it guards |
|---|---|---|
| `S3FileServiceIntegrationTests` | 31 | End to end against `LoopbackS3Server`: directory folding, paging, placeholder hiding, byte-for-byte identical multipart uploads, content type, Range resume, server-side copy, recursive delete, non-empty buckets not emptied, presigned URLs |
| `S3ManagementServiceTests` | 20 | Mapping and **error classification** on the 116-operation side: empty state vs. unsupported vs. real failure, restoring a version must be implemented as a copy, changing storage class is a self-copy, metadata must carry REPLACE |
| `S3ConfigFormTests` | 11 | **The form must not swallow unknown fields**, nested paths are filled in automatically, an empty value writes null, numbers are written as JSON numbers |
| `PluginProtocolTests` (host side) | 9 | The whole declare → register → lazy activation → unregister cycle, protocol id prefix validation, host contract routing, SDK exception translation, defaults filled in for old profiles |
| `S3InteropTests` | 10 | The boundary from AWSSDK exceptions → the Core neutral exception family |
| `S3ObjectPathTests` | 13 | Round-trip consistency of path ↔ bucket/key, adding the trailing slash to prefixes, bucket-name validation |
| `SigV4VerifierTests` | 11 | The correctness of **the test infrastructure itself** (see below) |

### 8.1 Why the loopback server still verifies signatures

`LoopbackS3Server` recomputes SigV4 from the **raw request line** it receives and compares. Signing on the product
side is now the SDK's job, but this check guards the other half: **whether the client configuration (endpoint /
region / addressing style / credentials) actually reached the wire correctly.** When any of these is wrong, a real
server only answers `SignatureDoesNotMatch`, with no clue whatsoever.

The verifier is itself the yardstick, so it must be verified too — `SigV4VerifierTests` checks it segment by segment
(canonical request / string to sign / signature) against the 5 examples in AWS's official documentation; otherwise a
wrong verifier would let anything through.

> **Still to be verified on real servers**: all of the above holds on the loopback server. Before going live, run it
> once against MinIO (self-signed certificate + path style) and once against real AWS S3 (virtual-hosted style +
> cross-region bucket + versioning + object lock), focusing on how items 1 / 2 / 3 of §6 behave on real
> implementations.

---

## 9. Operations not covered (an explicit boundary)

Of the 116 operations, 4 **cannot or should not be called by a client**; everything else is reachable:

| Operation | Why it is not done |
|---|---|
| `WriteGetObjectResponse` | Can only be called from inside an S3 Object Lambda function; a desktop client physically cannot use it |
| `CreateSession` | The internal session token of S3 Express directory buckets, managed automatically by the SDK |
| `GetObjectTorrent` | BitTorrent distribution, deprecated by AWS |
| `ListDirectoryBuckets` | Exclusive to S3 Express One Zone; this feature targets general-purpose S3 and compatible implementations, and it will be added when there is demand |

`InitiateMultipartUpload` / `UploadPart` / `CompleteMultipartUpload` / `CopyPart` are not exposed directly in the UI —
the transfer engine uses them internally; **management** of multipart uploads (listing, aborting) is in the bucket
manager.

---

## 10. The costs and boundaries of the plugin approach

**Cost 1: the protocol capability is `inProcess` only.** A protocol is a high-frequency channel through which the host
**calls back into the plugin** (listing directories, streaming reads, transfer progress), and the isolated-process RPC
currently only carries requests in the plugin → host direction. A manifest that declares `contributes.protocols` but
asks for `hostMode: "isolated"` is rejected at discovery time with a reason — far better than letting it install and
behave as "the tab is there, but clicking it never connects". The cost is that an S3 plugin crash affects the host
process; given that it is a first-party plugin built from the same source as the host, that trade-off is acceptable.
Lifting it requires making the RPC carry requests in both directions, which is follow-up work under blueprint 05.

**Cost 2: rate limits have to be fed to the plugin by the host.** The global bandwidth cap is a **user preference**
and applies equally to SFTP/FTP/plugin protocols; each protocol plugin cannot be allowed its own copy (the user would
have to configure it three times, and "total bandwidth cap" would stop meaning anything). So the SDK added
`IProtocolsApi.GetTransferOptionsAsync()`, which the plugin asks afresh at the start of every transfer; and
`ThrottledStream` was copied into the plugin (a 153-line, zero-dependency token bucket).

**Cost 3: each side carries its own strings.** The 149 S3 strings moved from the host's five `.resx` files into the
plugin's own `Loc.cs`, keeping only Chinese and English — the host supports five languages, but having the host carry
a dictionary for a protocol it does not even know would be putting the cart before the horse; a missing language falls
back to English rather than showing a key name.

**Explicitly not done: migrating old data.** Value 3 of `ConnectionType` (briefly used for the built-in S3) is retired
and **not reused**; reading it always degrades to SSH. The built-in S3 never shipped in any released version, so there
is no user data to migrate; keeping a permanent migration branch for data that does not exist would only end up
hurting real plugin profiles one day.

**Boundary: a protocol id cannot change once published.** It is stored in the user's session profile
(`SessionProfile.PluginProtocolId`); renaming it would leave every old profile unable to recognise its own protocol.
The id is also forced to all lowercase — it is compared in three places (the registry, the UI and the stored profile),
and allowing uppercase would let "the same id" be judged "not the same" at different stages.

---

## 11. Verification

`dotnet build` + `dotnet test` (.NET 11 preview 7): **build 0 errors 0 warnings; 1678 tests, 1676 passed, 1 skipped,
1 failed**. The only failure is `Scan_RealXshellSessions_RecoversAtLeastOnePassword` — password derivation against real
Xshell sessions, unrelated to this feature, and a pre-existing baseline failure on this machine.

All 98 S3 tests moved into the plugin are green, including 31 integration tests that hit the loopback server and
recompute and compare SigV4 every time. This deserves its own note: the 7 contract files that were deleted by mistake
and rebuilt from their callers and tests were confirmed identical to the originals **at the behavioural level** by
this batch of integration tests, not merely "it compiles".

Before going live it is still advisable to run once against a real MinIO (self-signed certificate + path style) and
once against AWS S3 (virtual-hosted style + cross-region bucket), focusing on items 1/2/3 of §6 — the loopback server
cannot verify how real implementations react to those headers.
